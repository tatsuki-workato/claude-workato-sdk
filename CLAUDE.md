# Workato Connector SDK Development

## Project Overview
This project is for developing custom connectors using the Workato Connector SDK gem.
Connector code lives in `connector.rb`. Tests use RSpec + VCR.

## Project Structure
```
.                         # root
├── connector.rb          # Main connector code (replica of Workato cloud code)
├── fixtures/             # Input/output JSON for RSpec and CLI testing
├── spec/                 # RSpec unit tests (connector_spec.rb + spec_helper.rb)
├── tape_library/         # VCR cassettes — recorded HTTP interactions
├── .github/              # GitHub Actions workflow files (if using GitHub)
├── .gitignore            # Excludes master.key, .rspec_status, etc.
├── .rspec                # Default RSpec flags (--format documentation, --color)
├── Gemfile               # Gem dependencies for RSpec
├── Gemfile.lock          # Auto-generated; locks gem versions
├── logo.png              # Connector logo (used by workato push)
├── master.key            # Encryption key — gitignored, never commit
├── README.md             # Connector docs (used by workato push as description)
└── settings.yaml.enc     # Encrypted credentials for testing (or settings.yaml)
```

## Key Commands

### SDK CLI (workato gem)
```bash
workato new <path>          # Scaffold new connector project
workato exec <PATH>         # Execute a connector block locally
workato push                # Upload and release connector to Workato workspace
workato edit settings.yaml.enc   # Edit encrypted credentials
workato generate schema     # Generate Workato schema from JSON/CSV sample
workato oauth2              # Run OAuth2 Authorization Code flow locally
```

### Common exec patterns
```bash
# Test an action's execute lambda
workato exec actions.<action_name>.execute \
  --input='fixtures/actions/<action_name>/input.json' --verbose

# Test input_fields schema
workato exec actions.<action_name>.input_fields

# Test a method
workato exec methods.<method_name> \
  --args='fixtures/methods/<method_name>/args.json'

# Test connection
workato exec connection.authorization.acquire \
  --settings='settings.yaml.enc' --verbose
```

### RSpec
```bash
bundle exec rspec                          # Run all tests
bundle exec rspec spec/connector_spec.rb:16  # Run specific line
VCR_RECORD_MODE=once bundle exec rspec spec/actions/test_action_spec.rb  # Record HTTP
```

## Connector Code Conventions
- The connector is a single Ruby hash assigned to a top-level variable
- Main keys: `title`, `connection`, `test`, `actions`, `triggers`, `methods`, `object_definitions`, `pick_lists`
- Use `call` to invoke methods within the connector: `call('method_name', args)`
- HTTP requests use Workato's built-in `get`, `post`, `put`, `patch`, `delete` helpers (not raw HTTP libraries)
- All lambdas receive `(connection, input)` or relevant args — never use Ruby `def`; use `lambda` or `->`
- Schema fields: always define `name`, `label`, `type`, and `optional`
- Error handling: use `error("message")` to raise user-facing errors

## Testing Approach
- Unit tests live in `spec/`; fixtures (input JSON) live in `fixtures/`
- VCR records real HTTP interactions into `tape_library/`; replay them in CI
- For `secure` mode (default), set `VCR_RECORD_MODE=once` to record new cassettes
- Use `--output=fixtures/.../output.json` to capture CLI output as fixture
- Multiple credential sets in `settings.yaml.enc` use named keys (e.g., `valid_connection`, `invalid_connection`)

## Credentials & Security
- Store credentials in `settings.yaml.enc` (encrypted) or `settings.yaml` (plain, dev only)
- `master.key` is always gitignored — never commit it
- For CI, pass via `WORKATO_CONNECTOR_MASTER_KEY` environment variable

## Deployment Flow
Local dev → `workato push` to DEV workspace → UAT → PROD
- Use `--folder <folder_id>` with `workato push` to target a specific folder
- Connector versions are immutable once released; always push a new version

## Agents & Skills

This project includes specialized subagents and skills under `.claude/`. Use them for focused tasks.

| Agent | Invoke when... |
|---|---|
| `workato-sdk-pro` | Writing or reviewing `connector.rb` — actions, triggers, auth, schema |
| `workato-cli` | Running CLI commands, generating fixtures, pushing, debugging exec errors |
| `openapi-analyzer` | An OpenAPI spec is provided and needs analyzing before implementation |

| Skill | Auto-loads when... |
|---|---|
| `workato-conventions` | Writing or reviewing connector code |
| `rspec-patterns` | Writing or reviewing spec files |
| `openapi-to-workato` | Translating OpenAPI types, auth, or endpoints to Workato DSL |

`workato-sdk-pro` preloads all three skills automatically.
`openapi-analyzer` preloads `openapi-to-workato`.

---

## Development Workflow

### New connector from scratch

```
1. workato new <path>               # scaffold project (choose "secure")
2. EDITOR=nano workato edit settings.yaml.enc   # add credentials
3. workato oauth2                   # OAuth2 only — get initial tokens
   OR workato exec test --verbose   # custom_auth / api_key — verify connection
```

### Implementing from an OpenAPI spec

**Small spec (< a few hundred lines):**
```
→ workato-sdk-pro  reads spec directly + implements connector.rb
```

**Large spec (hundreds of endpoints or multiple files):**
```
→ openapi-analyzer  analyzes spec → returns implementation plan
→ workato-sdk-pro   receives plan → implements connector.rb
```

### Per action/trigger cycle

```
1. workato-sdk-pro   implement action/trigger in connector.rb
2. workato-cli       create fixtures/; run workato exec --verbose to verify
3. workato-cli       save output fixture with --output
4. workato-sdk-pro   write spec file
5. workato-cli       VCR_RECORD_MODE=once bundle exec rspec <spec> to record cassette
6. workato-cli       bundle exec rspec to confirm all tests pass
```

### Push to workspace

```
1. bundle exec rspec                             # all tests green
2. workato push --api-token=<token> \
     --environment=https://app.jp.workato.com \ # match customer's DC
     --folder=<folder_id> \
     --notes='<version notes>'
```

### Pre-push checklist

- [ ] `bundle exec rspec` passes with no failures
- [ ] `test` lambda response is under 5,000 characters
- [ ] `refresh_on: [401]` present for all token-based auth
- [ ] No hardcoded full URLs — `base_uri` defined in `connection`
- [ ] `master.key` is in `.gitignore` and not committed
- [ ] `README.md` updated with setup instructions for this connector

---

## Reference
- SDK docs: https://docs.workato.com/developing-connectors/sdk.html
- CLI reference: https://docs.workato.com/developing-connectors/sdk/cli.html
- GitHub SDK gem: https://github.com/workato/workato-connector-sdk
