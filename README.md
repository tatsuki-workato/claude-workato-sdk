# Workato Connector SDK — Claude Code Template

A Claude Code project template for developing Workato custom connectors with AI assistance.  
Includes a `CLAUDE.md`, subagents, and skills tuned for the Workato SDK DSL, CLI, and OpenAPI-based workflows.

---

## Prerequisites

- Ruby (recommend via `rbenv` or `rvm`)
- Workato SDK gem: `gem install workato-connector-sdk`
- Claude Code CLI

---

## Quick start

### 1. Scaffold a new connector project

```bash
workato new ./my-connector
cd my-connector
# When prompted, select: 1 - secure (recommended)
```

### 2. Copy this template's Claude config into the project

```bash
cp /path/to/this-template/CLAUDE.md .
cp -r /path/to/this-template/.claude .
```

### 3. Add credentials

```bash
# Encrypted (recommended)
EDITOR=nano workato edit settings.yaml.enc

# Plain text (dev only — do not commit)
# Edit settings.yaml directly
```

### 4. Establish connection

```bash
# OAuth2 Authorization Code connectors
workato oauth2

# All other auth types (custom_auth, api_key, basic_auth)
workato exec test --verbose
```

### 5. Start Claude Code

```bash
claude
```

---

## Project structure

```
.
├── connector.rb              # Main connector code
├── fixtures/                 # Input/output JSON for CLI and RSpec
├── spec/                     # RSpec unit tests + spec_helper.rb
├── tape_library/             # VCR cassettes (encrypted HTTP recordings)
├── .github/                  # GitHub Actions workflows (optional)
├── .gitignore                # Excludes master.key, .rspec_status, etc.
├── .rspec                    # RSpec default flags
├── Gemfile / Gemfile.lock    # Gem dependencies
├── logo.png                  # Connector logo (used by workato push)
├── master.key                # Encryption key — NEVER commit
├── README.md                 # This file (also used by workato push as description)
├── settings.yaml.enc         # Encrypted credentials (or settings.yaml for dev)
├── CLAUDE.md                 # Claude Code project memory
└── .claude/
    ├── agents/
    │   ├── workato-sdk-pro.md    # SDK implementation specialist
    │   ├── workato-cli.md        # CLI execution specialist
    │   └── openapi-analyzer.md   # OpenAPI spec analyzer
    └── skills/
        ├── workato-conventions/  # SDK DSL rules and patterns
        ├── rspec-patterns/       # RSpec + VCR test patterns
        └── openapi-to-workato/   # OpenAPI → Workato translation rules
```

---

## Agents & Skills

### Agents

| Agent | Role | Invoke when... |
|---|---|---|
| `workato-sdk-pro` | Implements and reviews `connector.rb` | Writing actions, triggers, auth, schema |
| `workato-cli` | Runs CLI commands and interprets output | Generating fixtures, exec, push, debugging |
| `openapi-analyzer` | Analyzes OpenAPI specs | A spec file is provided before implementation |

### Skills

| Skill | Content | Auto-loads when... |
|---|---|---|
| `workato-conventions` | DSL rules, naming, HTTP helpers, schema patterns | Writing or reviewing connector code |
| `rspec-patterns` | RSpec boilerplate, VCR cassette management | Writing or reviewing spec files |
| `openapi-to-workato` | Auth/type/endpoint mapping from OpenAPI to Workato | Translating an OpenAPI spec |

`workato-sdk-pro` preloads all three skills.  
`openapi-analyzer` preloads `openapi-to-workato`.

---

## Development workflow

### Implementing from an OpenAPI spec

```
# Small spec (< a few hundred lines)
→ Pass spec to workato-sdk-pro directly

# Large spec (hundreds of endpoints or multiple files)  
→ openapi-analyzer analyzes spec and produces an implementation plan
→ workato-sdk-pro implements connector.rb from the plan
```

### Per action/trigger cycle

```
1. workato-sdk-pro   implement in connector.rb
2. workato-cli       create fixtures/, run workato exec --verbose
3. workato-cli       save output with --output flag
4. workato-sdk-pro   write spec file
5. workato-cli       VCR_RECORD_MODE=once bundle exec rspec <spec>
6. workato-cli       bundle exec rspec — confirm all green
```

### Push to Workato workspace

```bash
workato push \
  --api-token=<token> \
  --environment=https://app.jp.workato.com \
  --folder=<folder_id> \
  --notes='<version notes>'
```

Data center URLs: `app.workato.com` / `app.eu.workato.com` / `app.jp.workato.com` / `app.sg.workato.com` / `app.au.workato.com`

---

## Pre-push checklist

- [ ] `bundle exec rspec` passes with no failures
- [ ] `test` lambda response is under 5,000 characters
- [ ] `refresh_on: [401]` present for all token-based auth
- [ ] No hardcoded full URLs — `base_uri` defined in `connection`
- [ ] `master.key` is in `.gitignore` and not committed
- [ ] `README.md` updated with setup instructions for this connector

---

## Security

- `master.key` — never commit. Share with teammates via a secure channel (1Password, etc.)
- In CI, set `WORKATO_CONNECTOR_MASTER_KEY` as an environment secret instead of committing the key
- `settings.yaml.enc` is safe to commit — it's encrypted with `master.key`
- `settings.yaml` (plain) is for local dev only — add to `.gitignore`

---

## Reference

- [Connector SDK docs](https://docs.workato.com/developing-connectors/sdk.html)
- [SDK CLI reference](https://docs.workato.com/developing-connectors/sdk/cli.html)
- [SDK gem (GitHub)](https://github.com/workato/workato-connector-sdk)
- [Claude Code docs](https://code.claude.com/docs)
