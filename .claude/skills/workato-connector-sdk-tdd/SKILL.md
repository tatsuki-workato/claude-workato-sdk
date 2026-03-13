---
name: workato-connector-sdk-tdd
description: TDD workflow methodology for Workato Connector SDK. Applies Kent Beck's Red-Green-Refactor cycle to connector development — when to write specs, when to touch connector.rb, and how to sequence the build loop.
---

# Workato Connector SDK — TDD Workflow

Apply Kent Beck's Red-Green-Refactor cycle to Workato Connector SDK development. Code patterns live in `rspec-patterns`; this skill governs *when* and *in what order* to act.

---

## The Cycle

### 🔴 Red — spec first, before touching connector.rb

1. Create `spec/actions/<name>_spec.rb` (or triggers/, pick_lists/, etc.)
2. Write the minimum assertions that describe the *behavior* you want
3. Run it — **it must fail**. A passing spec at this stage means the behavior already exists or the test is wrong.

```bash
bundle exec rspec spec/actions/<name>_spec.rb
# Expected: failure (NameError, NoMethodError, or assertion failure)
```

### 🟢 Green — minimum connector.rb to pass

Write only enough `connector.rb` code to turn the spec green. Hardcoding return values is acceptable here — correctness matters, not elegance.

```bash
bundle exec rspec spec/actions/<name>_spec.rb
# Expected: green bar
```

If the spec needs real HTTP, record the cassette first:

```bash
VCR_RECORD_MODE=once bundle exec rspec spec/actions/<name>_spec.rb
```

### 🔵 Refactor — clean up under green

With the green bar as your safety net: extract shared lambdas, DRY up schemas, improve naming. Re-run the full suite after each change.

```bash
bundle exec rspec
# Must stay green throughout
```

---

## Sequencing rules

| Rule | Why |
|---|---|
| One failing spec at a time | Isolates cause; faster feedback loop |
| Never write connector.rb code without a failing spec | Untested code has no safety net |
| Never refactor on a red bar | You can't tell if refactoring broke something |
| Triangulate before generalizing | Add a second test with different input to force the real implementation — don't generalize from one example |
| Spec describes what, connector.rb describes how | Keep assertions in terms of outputs, not implementation details |

---

## Per-lambda workflow

### New action

```
1. spec/actions/<name>_spec.rb  — write failing execute spec
2. connector.rb                 — add action stub (hardcoded response)
3. VCR_RECORD_MODE=once rspec   — record cassette with real HTTP
4. connector.rb                 — replace stub with real HTTP call
5. rspec                        — green
6. rspec (full suite)           — stay green
7. refactor if needed           — re-run after each change
```

### New trigger

Same as action; use `poll` instead of `execute`. Test the closure (second poll) as a separate spec after the first poll passes.

### New pick_list or method

Specs for these rarely need VCR (pure logic). Write the spec, make it green with inline logic, then extract to connector.rb.

---

## When to break the cycle

- **Spiking**: exploring an unknown API — use `workato exec --verbose` without writing specs first, then delete the spike code and restart with a spec.
- **Fixtures**: `input.json` / `output.json` are not part of TDD; generate them via `workato exec --output` after going green, then commit as regression anchors.
