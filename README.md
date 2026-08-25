# codex-hijack

Documentation and replacement of the Codex CLI harness's stock context.

The stock context (the base context the Codex CLI harness composes into the top
stratum ahead of everything authored) incentivizes behaviors that conflict with
the psyche's philosophy and approach. This repository documents the stock
context block by block, identifies the worst offenders, and will house
replacement base contexts.

## Structure

- `stock-context/INVENTORY.md` -- complete block-by-block inventory of the
  stock context at the witnessed version, with override mechanisms and
  worst-offender candidates flagged for review
- `stock-context/base-prompts/` -- verbatim extracted base prompt files for
  every model variant
- `stock-context/injected-blocks/` -- verbatim extracted template files for
  permission, collaboration, guardian, and other harness-composed blocks

## Witnessed version

Codex CLI 0.149.1, from openai/codex tag `rust-v0.149.1` (commit `980a6d12`).

## Override mechanisms

The base context can be fully replaced via:
- `config.toml` `instructions` (inline text)
- `config.toml` `model_instructions_file` (path to replacement file)
- `SessionCreateParams.base_instructions` (programmatic, highest priority)

`developer_instructions` appends as a separate developer-role message without
replacing the base context.

See `stock-context/INVENTORY.md` for the complete override table.
