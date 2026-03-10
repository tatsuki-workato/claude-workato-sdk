---
name: openapi-to-workato
description: OpenAPI spec to Workato connector translation rules. Load when reading an OpenAPI/Swagger spec file, converting API documentation to connector.rb, or mapping OpenAPI types/auth/endpoints to Workato DSL patterns.
---

# OpenAPI → Workato Connector Translation

## Reading an OpenAPI spec

Before implementing, always extract these from the spec:

1. **Base URL** — `servers[].url` → `base_uri` in `connection`
2. **Auth scheme** — `securitySchemes` → determines `authorization.type`
3. **Endpoints to implement** — `paths` → actions / triggers
4. **Reusable schemas** — `components/schemas` → `object_definitions`
5. **Enums** → `pick_lists`

---

## Auth mapping

| OpenAPI `securitySchemes.type` | Workato `authorization.type` |
|---|---|
| `oauth2` / `authorizationCode` | `oauth2` |
| `oauth2` / `clientCredentials` | `custom_auth` |
| `http` / `bearer` | `custom_auth` (acquire + apply Bearer header) |
| `http` / `basic` | `basic_auth` |
| `apiKey` in header | `api_key` or `custom_auth` with `apply` |
| `apiKey` in query | `custom_auth` with `apply` using `.params()` |

### OAuth2 Authorization Code → `type: 'oauth2'`

```yaml
# OpenAPI spec
securitySchemes:
  oauth2:
    type: oauth2
    flows:
      authorizationCode:
        authorizationUrl: https://example.com/oauth/authorize
        tokenUrl: https://example.com/oauth/token
        scopes:
          read: Read access
          write: Write access
```

```ruby
# connector.rb
authorization: {
  type: 'oauth2',
  authorization_url: lambda do |connection|
    "https://example.com/oauth/authorize?scope=read+write"
  end,
  token_url: lambda do |connection|
    "https://example.com/oauth/token"
  end,
  apply: lambda do |connection, access_token|
    headers('Authorization': "Bearer #{access_token}")
  end,
  refresh_on: [401],
  refresh: lambda do |connection, refresh_token|
    response = post("https://example.com/oauth/token")
      .payload(grant_type: 'refresh_token', refresh_token: refresh_token,
               client_id: connection['client_id'], client_secret: connection['client_secret'])
      .request_format_www_form_urlencoded
    [{ access_token: response['access_token'], refresh_token: response['refresh_token'] }]
  end,
}
```

### Client Credentials → `type: 'custom_auth'`

```ruby
authorization: {
  type: 'custom_auth',
  acquire: lambda do |connection|
    response = post("https://example.com/oauth/token")
      .payload(grant_type: 'client_credentials',
               client_id: connection['client_id'],
               client_secret: connection['client_secret'])
      .request_format_www_form_urlencoded
    { access_token: response['access_token'] }
  end,
  apply: lambda do |connection|
    headers('Authorization': "Bearer #{connection['access_token']}")
  end,
  refresh_on: [401],
  detect_on: [/invalid_token/, /token_expired/],
}
```

---

## HTTP method mapping

| OpenAPI | Workato helper |
|---|---|
| `GET` | `get(path)` |
| `POST` | `post(path, payload)` |
| `PUT` | `put(path, payload)` |
| `PATCH` | `patch(path, payload)` |
| `DELETE` | `delete(path)` |

Path parameters (`/customers/{id}`) → Ruby string interpolation:

```ruby
# OpenAPI: GET /customers/{id}
get("/customers/#{input['id']}")

# OpenAPI: GET /customers?page={page}&limit={limit}
get("/customers").params(page: input['page'], limit: input['limit'])
```

---

## Schema type mapping

| OpenAPI type | Workato `type` | Notes |
|---|---|---|
| `string` | `string` | default |
| `string` + `format: date` | `date` | |
| `string` + `format: date-time` | `date_time` | |
| `string` + `format: password` | `string` + `control_type: 'password'` | |
| `string` + `enum: [...]` | `string` + `control_type: 'select'` + pick_list | |
| `integer` / `number` | `integer` / `number` | |
| `boolean` | `boolean` | |
| `object` | `object` + `properties: [...]` | |
| `array` of objects | `array` + `of: 'object'` + `properties: [...]` | |
| `array` of strings | `array` + `of: 'string'` | |
| `$ref: '#/components/schemas/Foo'` | → `object_definitions['foo']` | extract to object_definitions |

### Example translation

```yaml
# OpenAPI components/schemas
Customer:
  type: object
  required: [id, email]
  properties:
    id:
      type: integer
    email:
      type: string
      format: email
    status:
      type: string
      enum: [active, inactive, pending]
    created_at:
      type: string
      format: date-time
```

```ruby
# connector.rb — object_definitions
object_definitions: {
  customer: {
    fields: lambda do |connection, config_fields, object_definitions|
      [
        { name: 'id',         type: 'integer',  label: 'ID',           optional: true  },
        { name: 'email',      type: 'string',   label: 'Email',        optional: false,
          control_type: 'email' },
        { name: 'status',     type: 'string',   label: 'Status',       optional: true,
          control_type: 'select', pick_list: 'customer_statuses' },
        { name: 'created_at', type: 'date_time', label: 'Created at',  optional: true  },
      ]
    end
  }
},

pick_lists: {
  customer_statuses: lambda do
    [['Active', 'active'], ['Inactive', 'inactive'], ['Pending', 'pending']]
  end
}
```

---

## Endpoint → Action mapping

```yaml
# OpenAPI
paths:
  /customers:
    get:
      operationId: searchCustomers
      summary: Search customers
      parameters:
        - name: name
          in: query
          schema:
            type: string
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
      responses:
        '200':
          content:
            application/json:
              schema:
                type: object
                properties:
                  items:
                    type: array
                    items:
                      $ref: '#/components/schemas/Customer'
                  total:
                    type: integer
```

```ruby
# connector.rb — action
actions: {
  search_customers: {
    title: 'Search customers',
    subtitle: 'Search customers by name',
    description: lambda { "Search <span class='provider'>customers</span>" },

    input_fields: lambda do |object_definitions|
      [
        { name: 'name',  type: 'string',  label: 'Name',  optional: true  },
        { name: 'limit', type: 'integer', label: 'Limit', optional: true,
          hint: 'Max records to return. Default: 20' },
      ]
    end,

    execute: lambda do |connection, input|
      get('/customers')
        .params(input.reject { |_, v| v.blank? })
        .after_error_response(/.*/) { |_c, body, _h, msg| error("#{msg}: #{body}") }
    end,

    output_fields: lambda do |object_definitions|
      [
        { name: 'items', type: 'array', of: 'object', label: 'Customers',
          properties: object_definitions['customer'] },
        { name: 'total', type: 'integer', label: 'Total count' },
      ]
    end,
  }
}
```

---

## Pagination patterns

### Offset/limit (most common)

```yaml
# OpenAPI parameters
- name: offset
  in: query
- name: limit
  in: query
```

```ruby
# Polling trigger with offset cursor
poll: lambda do |connection, input, closure|
  offset = closure || 0
  response = get('/events')
    .params(offset: offset, limit: 100, since: input['since'])
    .after_error_response(/.*/) { |_c, body, _h, msg| error("#{msg}: #{body}") }
  events = response['items'] || []
  {
    events: events,
    next_poll: offset + events.size,
    can_trypoll_more: events.size >= 100
  }
end
```

### Cursor-based

```yaml
# OpenAPI response
next_cursor:
  type: string
  nullable: true
```

```ruby
poll: lambda do |connection, input, closure|
  params = { limit: 100 }
  params[:cursor] = closure if closure.present?
  response = get('/events').params(params)
  {
    events:           response['items'] || [],
    next_poll:        response['next_cursor'],
    can_trypoll_more: response['next_cursor'].present?
  }
end
```

---

## What to ignore from OpenAPI

- `deprecated: true` endpoints — skip unless explicitly requested
- `readOnly` fields in request schemas — omit from `input_fields`
- `writeOnly` fields in response schemas — omit from `output_fields`
- Internal/admin-only endpoints (paths with `/admin/`, `/internal/`) — skip unless confirmed
- Endpoints with no meaningful Workato use case (e.g., health checks, metadata)
