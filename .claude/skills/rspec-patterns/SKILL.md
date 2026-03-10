---
name: rspec-patterns
description: Workato SDK RSpec test patterns and VCR cassette conventions. Load when writing, reviewing, or fixing spec files for a Workato connector project.
---

# Workato SDK RSpec Patterns

## Boilerplate — every spec file

```ruby
# frozen_string_literal: true

RSpec.describe '<subject>', :vcr do
  let(:connector) { Workato::Connector::Sdk::Connector.from_file('connector.rb', settings) }
  let(:settings)  { Workato::Connector::Sdk::Settings.from_default_file }

  # tests here
end
```

- Always add `:vcr` metadata — this enables VCR cassette recording/playback per example
- `from_default_file` reads `settings.yaml.enc` (or `settings.yaml` for simple mode)
- Use `from_encrypted_file('invalid_settings.yaml.enc')` to test invalid credentials

---

## Connection test

```ruby
RSpec.describe 'connector', :vcr do
  let(:connector) { Workato::Connector::Sdk::Connector.from_file('connector.rb', settings) }
  let(:settings)  { Workato::Connector::Sdk::Settings.from_default_file }

  it { expect(connector).to be_present }

  describe 'test' do
    subject(:output) { connector.test(settings) }

    context 'given valid credentials' do
      it 'establishes valid connection' do
        expect(output).to be_truthy
      end

      it 'returns response that is not excessively large' do
        expect(output.to_s.length).to be < 5000
      end
    end

    context 'given invalid credentials' do
      let(:settings) { Workato::Connector::Sdk::Settings.from_encrypted_file('invalid_settings.yaml.enc') }

      it 'raises an error' do
        expect { output }.to raise_error(/401|403|Unauthorized/)
      end
    end
  end
end
```

**Note:** RSpec does not support the OAuth2 token refresh flow. Tokens in `settings.yaml.enc` must already be valid before running specs. Use `workato exec test` or `workato oauth2` to refresh tokens first.

---

## Action spec

```ruby
RSpec.describe 'actions/search_customers', :vcr do
  let(:connector) { Workato::Connector::Sdk::Connector.from_file('connector.rb', settings) }
  let(:settings)  { Workato::Connector::Sdk::Settings.from_default_file }
  let(:input)     { JSON.parse(File.read('fixtures/actions/search_customers/input.json')) }

  # Test the full action (input coercion + execute + output mapping)
  describe 'full action' do
    subject(:output) { connector.actions.search_customers(input) }

    let(:expected_output) { JSON.parse(File.read('fixtures/actions/search_customers/output.json')) }

    it 'returns expected output' do
      expect(output).to eq(expected_output)
    end
  end

  # Test execute lambda directly (skips input coercion)
  describe 'execute' do
    subject(:output) { connector.actions.search_customers.execute(settings, input) }

    it 'returns a list' do
      expect(output['list']).to be_a(Array)
    end

    it 'each item has required fields' do
      expect(output['list'].first).to include('id', 'name')
    end
  end

  # Test input_fields schema
  describe 'input_fields' do
    subject(:fields) { connector.actions.search_customers.input_fields(settings, {}) }

    it 'returns an array of fields' do
      expect(fields).to be_a(Array)
      expect(fields).not_to be_empty
    end
  end

  # Test output_fields schema
  describe 'output_fields' do
    subject(:fields) { connector.actions.search_customers.output_fields(settings, {}) }

    it 'returns an array of fields' do
      expect(fields).to be_a(Array)
    end
  end
end
```

**Full action vs execute:**
- `connector.actions.action_name(input)` — runs the entire action including input type coercion
- `connector.actions.action_name.execute(settings, input)` — calls only the `execute` lambda directly; use `input_parsed.json` (post-coercion values) as input

---

## Trigger spec (polling)

```ruby
RSpec.describe 'triggers/new_updated_object', :vcr do
  let(:connector) { Workato::Connector::Sdk::Connector.from_file('connector.rb', settings) }
  let(:settings)  { Workato::Connector::Sdk::Settings.from_default_file }
  let(:input)     { JSON.parse(File.read('fixtures/triggers/new_updated_object/input.json')) }

  describe 'poll' do
    subject(:output) { connector.triggers.new_updated_object.poll(settings, input, nil) }

    it 'returns events array' do
      expect(output[:events]).to be_a(Array)
    end

    it 'returns a next_poll cursor' do
      expect(output[:next_poll]).to be_present
    end

    it 'returns can_trypoll_more boolean' do
      expect(output[:can_trypoll_more]).to be_in([true, false])
    end
  end

  describe 'output_fields' do
    subject(:fields) do
      config = JSON.parse(File.read('fixtures/triggers/new_updated_object/config.json'))
      connector.triggers.new_updated_object.output_fields(settings, config)
    end

    it 'returns an array of fields' do
      expect(fields).to be_a(Array)
      expect(fields).not_to be_empty
    end
  end
end
```

---

## Method spec

```ruby
RSpec.describe 'methods/format_payload', :vcr do
  let(:connector) { Workato::Connector::Sdk::Connector.from_file('connector.rb', settings) }
  let(:settings)  { Workato::Connector::Sdk::Settings.from_default_file }

  subject(:result) { connector.methods.format_payload(input) }

  context 'with nil values' do
    let(:input) { { 'name' => 'Alice', 'email' => nil } }

    it 'removes nil values' do
      expect(result).to eq({ 'name' => 'Alice' })
      expect(result).not_to have_key('email')
    end
  end
end
```

---

## VCR cassettes

### Record mode

| Command | When to use |
|---|---|
| `bundle exec rspec` | Normal CI run — replays existing cassettes, fails on unknown requests |
| `VCR_RECORD_MODE=once bundle exec rspec spec/path_to_spec.rb` | Record a new cassette for the first time |
| `VCR_RECORD_MODE=new_episodes bundle exec rspec` | Add new interactions to existing cassettes |

### Troubleshooting `VCR::Errors::UnhandledHTTPRequestError`

This means VCR cannot match an outgoing HTTP request to a recorded cassette. Common causes:

1. **New spec with no cassette yet** — run with `VCR_RECORD_MODE=once`
2. **Access token rotated** — the `Authorization` header changed; re-record with `VCR_RECORD_MODE=once` after refreshing tokens via `workato oauth2` or `workato exec test`
3. **Request body changed** — a payload field was added or removed; re-record the cassette
4. **Matching too strict** — relax `match_requests_on` in `spec_helper.rb`, e.g. remove `:body` if payloads vary

### Cassette file locations

Cassettes are stored in `tape_library/` and named automatically from the RSpec description path:

```
tape_library/
└── actions/
│   └── search_customers/
│       └── execute/given_valid_input/gives_expected_output.yml
└── triggers/
    └── new_updated_object/
        └── poll/returns_events_array.yml
```

---

## Fixtures

Input JSON files must be created manually. Output JSON files can be generated via CLI:

```bash
# Save execute output to fixture
workato exec actions.search_customers.execute \
  --input='fixtures/actions/search_customers/input.json' \
  --output='fixtures/actions/search_customers/output.json'
```

- `input.json` — raw user input (pre-coercion); used with full action test
- `input_parsed.json` — post-coercion input; used with `execute` lambda test directly
- `output.json` — expected output; generated from CLI then committed

---

## Running tests

```bash
# Run all specs
bundle exec rspec

# Run a single spec file
bundle exec rspec spec/actions/search_customers_spec.rb

# Run a specific line
bundle exec rspec spec/actions/search_customers_spec.rb:16

# Record new cassettes
VCR_RECORD_MODE=once bundle exec rspec spec/actions/search_customers_spec.rb

# Generate RSpec stubs from connector.rb
workato generate test
```
