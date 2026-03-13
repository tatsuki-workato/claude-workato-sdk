---
name: workato-conventions
description: Workato Connector SDK coding conventions and patterns. Auto-load when writing or reviewing connector.rb, defining actions, triggers, methods, object_definitions, or pick_lists.
user-invocable: false
---

# Workato Connector SDK Conventions

## Connector top-level structure

A connector is a single Ruby hash. All keys are optional except `title` and `test`.

```ruby
{
  title: 'My Connector',
  custom_action: Boolean,         # optional — enable ad-hoc HTTP actions from UI
  custom_action_help: Hash,       # optional — help panel config for custom actions
  connection: Hash,
  test: lambda do |connection| ... end,
  actions: Hash,
  triggers: Hash,
  methods: Hash,
  object_definitions: Hash,
  pick_lists: Hash,
}
```

## General rules

- **Never use Ruby `def`** — all code must be lambdas (`lambda do ... end`)
- **Naming**: use `snake_case` for all action/trigger/method/pick_list keys
- **HTTP**: use Workato's built-in helpers only — `get`, `post`, `put`, `patch`, `delete` — never `Net::HTTP`, `RestClient`, etc.
- **Error handling**: always add `.after_error_response(/.*/)` on HTTP calls; raise with `error("message")`
- **Reuse**: extract repeated logic into `methods`; call with `call('method_name', args)`
- **base_uri**: define in `connection` to avoid repeating full URLs in actions/triggers

## Schema fields

Every field needs at minimum: `name`, `label`, `type`, `optional`.

```ruby
{ name: 'email', label: 'Email address', type: 'string', optional: false }
```

Common `type` values: `string`, `integer`, `number`, `boolean`, `date`, `date_time`, `object`, `array`

Use `control_type` to improve UX: `text`, `password`, `select`, `multiselect`, `checkbox`, `url`, `email`, `subdomain`, `integer`

Use `hint` to guide users. Use `sticky: true` for commonly-used optional fields.

For nested objects: set `type: 'object'` and `properties: [...]`
For arrays: set `type: 'array'`, `of: 'object'`, and `properties: [...]`

## Actions

Title pattern: `"[Verb] [object]"` — e.g. `"Create lead"`, `"Search customers"` (not `"Lead created"`)

```ruby
actions: {
  search_customers: {
    title: 'Search customers',
    subtitle: 'Search by name or email',
    description: lambda do |input, picklist_label|
      "Search <span class='provider'>customers</span>"
    end,
    help: lambda do |input, picklist_label|
      { body: '...', learn_more_url: '...', learn_more_text: 'Learn more' }
    end,
    config_fields: [...],
    input_fields: lambda do |object_definitions, connection, config_fields|
      object_definitions['customer']
    end,
    execute: lambda do |connection, input|
      get('/api/v2/customers', name: input['name'])
        .after_error_response(/.*/) do |_code, body, _header, message|
          error("#{message}: #{body}")
        end
    end,
    output_fields: lambda do |object_definitions, connection, config_fields|
      object_definitions['customer']
    end,
    sample_output: lambda do |connection, input|
      call('get_sample_customer', {})
    end,
  }
}
```

## Triggers

- **Polling triggers**: use `poll` lambda; track cursor with `closure`; return `{ events: [...], next_poll: ..., can_trypoll_more: bool }`
- **Webhook triggers**: implement `webhook_subscribe`, `webhook_unsubscribe`, `webhook_notification`
- Add an optional `since` input field so users can backfill on first run
- `webhook_notification` lambda **cannot** make HTTP calls or invoke `call()`

```ruby
# Polling cursor pattern
poll: lambda do |connection, input, closure|
  since = (closure || input['since']).to_time.utc.iso8601
  response = get('/records', updated_after: since)
  {
    events: response['items'],
    next_poll: response['next_cursor'],
    can_trypoll_more: response['has_more']
  }
end
```

## object_definitions

Define reusable schemas here; reference them in actions/triggers via the `object_definitions` argument.

```ruby
object_definitions: {
  customer: {
    fields: lambda do |connection, config_fields|
      [
        { name: 'id',    label: 'ID',    type: 'string', optional: true  },
        { name: 'email', label: 'Email', type: 'string', optional: false },
      ]
    end
  }
}
```

## pick_lists

Use for static or dynamic dropdowns. Always return `[['Display label', 'value'], ...]`.

```ruby
pick_lists: {
  # Static
  statuses: lambda do
    [['Active', 'active'], ['Inactive', 'inactive']]
  end,

  # Dynamic (API call)
  customers: lambda do |connection|
    get('/api/v2/customers')['items'].map { |c| [c['name'], c['id']] }
  end,
}
```

Wire up with `pick_list: 'list_name'` + `control_type: 'select'` in schema fields.

## methods

Extract all reusable logic here. Call with `call('method_name', args)` or `call(:method_name, args)`.

```ruby
methods: {
  format_payload: lambda do |input|
    input.reject { |_k, v| v.blank? }
  end,
}
```

## custom_action

Set `custom_action: true` to let users build ad-hoc HTTP actions from the Workato UI (useful when no built-in action covers a specific API endpoint).

Pair with `custom_action_help` to guide users toward the right documentation:

```ruby
custom_action: true,

custom_action_help: {
  learn_more_url:  'https://developer.example.com/api',
  learn_more_text: 'API documentation',
  body: "<p>Build your own action with an HTTP request. <b>Authorization is included automatically.</b></p>"
}
```

- `body`: HTML string shown in the help panel
- `learn_more_url`: link target for the "learn more" button
- `learn_more_text`: label for that button
- Both keys are optional; omit `custom_action_help` entirely to show no help text

## connection

`connection` has three keys: `fields`, `authorization`, `base_uri`.

### fields

Input fields users fill in to connect. Always use `control_type: 'password'` for secrets.

```ruby
fields: [
  { name: 'client_id',     label: 'Client ID',     optional: false },
  { name: 'client_secret', label: 'Client secret', optional: false, control_type: 'password' },
  { name: 'subdomain',     label: 'Subdomain',     optional: false, control_type: 'subdomain',
    hint: 'your-company from https://your-company.example.com' },
]
```

### authorization types

| type | When to use |
|---|---|
| `oauth2` | Authorization Code Grant — user logs in via browser redirect |
| `custom_auth` | Client Credentials, ROPC, or any token-based flow without redirect |
| `basic_auth` | Username + password sent as HTTP Basic |
| `api_key` | Static API key in header or query param |

---

### OAuth2 — Authorization Code Grant (`type: 'oauth2'`)

Use when a user must log in via browser. Workato handles the redirect and callback automatically.

```ruby
authorization: {
  type: 'oauth2',

  authorization_url: lambda do |connection|
    params = {
      response_type: 'code',
      client_id: connection['client_id'],
      scope: 'read write'
    }.to_param
    "https://example.com/oauth/authorize?" + params
  end,

  # Option A — let Workato call the token endpoint automatically
  token_url: lambda do |connection|
    "https://example.com/oauth/token"
  end,

  # Option B — call the token endpoint manually (needed for non-standard flows)
  acquire: lambda do |connection, auth_code, redirect_uri|
    response = post("https://example.com/oauth/token")
      .payload(
        grant_type:    'authorization_code',
        code:          auth_code,
        redirect_uri:  redirect_uri,
        client_id:     connection['client_id'],
        client_secret: connection['client_secret']
      )
      .request_format_www_form_urlencoded
    [
      {
        access_token:             response['access_token'],
        refresh_token:            response['refresh_token'],
        refresh_token_expires_in: response['refresh_token_expires_in']
      },
      nil,  # owner ID — substitute nil if not needed
      {}    # optional extra fields to merge into connection hash
    ]
  end,

  # Apply token to every request
  apply: lambda do |connection, access_token|
    headers('Authorization': "Bearer #{access_token}")
  end,

  # Trigger refresh when API returns these status codes or body patterns
  refresh_on: [401, 403],
  detect_on:  [/Unauthorized/, /Token expired/],

  # How to get a new access_token using the refresh_token
  refresh: lambda do |connection, refresh_token|
    response = post("https://example.com/oauth/token")
      .payload(
        grant_type:    'refresh_token',
        refresh_token: refresh_token,
        client_id:     connection['client_id'],
        client_secret: connection['client_secret']
      )
      .request_format_www_form_urlencoded
    [
      {
        access_token:  response['access_token'],
        refresh_token: response['refresh_token']  # omit if API does not rotate refresh tokens
      }
    ]
  end,
}
```

**Key rules for `oauth2`:**
- `acquire` output must be an array: `[token_hash, owner_id_or_nil, extra_connection_hash_or_nil]`
- Token hash keys must be exactly `access_token`, `refresh_token`, `refresh_token_expires_in`
- `refresh` output must be an array: `[token_hash]`
- `refresh_on` triggers `refresh` lambda; `detect_on` catches errors hidden inside 200 responses
- Always include `refresh_on: [401]` — connections will break silently without it
- Use `token_url` for standard flows; use `acquire` when the token endpoint requires non-standard auth (e.g., HTTP Basic for client credentials)
- Test locally with `workato oauth2` CLI command

---

### OAuth2 — Client Credentials Grant (`type: 'custom_auth'`)

Use for server-to-server flows with no user redirect. `acquire` fetches a token on demand; Workato re-runs it when `refresh_on` or `detect_on` triggers.

```ruby
authorization: {
  type: 'custom_auth',

  acquire: lambda do |connection|
    response = post("https://example.com/oauth/token")
      .payload(
        grant_type:    'client_credentials',
        client_id:     connection['client_id'],
        client_secret: connection['client_secret']
      )
      .request_format_www_form_urlencoded
    {
      access_token: response['access_token']
      # store additional values here if needed; they merge into the connection hash
    }
  end,

  apply: lambda do |connection|
    headers('Authorization': "Bearer #{connection['access_token']}")
  end,

  refresh_on: [401, 403],
  detect_on:  [/invalid_token/, /token_expired/],
}
```

**Key rules for `custom_auth`:**
- `acquire` returns a **hash** (not an array), which is merged into the `connection` hash
- `acquire` only runs when `refresh_on` / `detect_on` signals fire — not on every request
- Always include the error codes that your `test` lambda returns when credentials are invalid inside `detect_on`, otherwise the initial connection may appear valid but fail later
- Do not put expiry timers in `acquire` — rely on `refresh_on` / `detect_on`

---

### base_uri

Always define. Avoids hardcoding full URLs in actions/triggers.

```ruby
base_uri: lambda do |connection|
  "https://#{connection['subdomain']}.example.com/api/v2"
end
```

## Best practices

- Use `object_definitions` + `config_fields` for multi-object actions/triggers (avoids duplicated schemas)
- Use `base_uri` in `connection` — never hardcode full URLs in actions/triggers
- Use `pick_list` for environment selection (production vs. sandbox)
- Always include `sample_output` to give users datapill previews
- Use `summarize_input` / `summarize_output` for large array payloads
- Response size from `test` lambda should be under 5,000 characters
