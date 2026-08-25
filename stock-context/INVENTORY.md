# Codex CLI Stock Context Inventory

Subject: complete inventory of the base context and harness-composed blocks in
Codex CLI 0.149.1.

Version witnessed: **0.149.1**
Binary: `/nix/store/a2hlxqhdyc642f8m6zhgkl5l2cbh2bks-codex-0.149.1/libexec/codex`
Source tag: `rust-v0.149.1` (commit `980a6d12110b110d29ec13bdcbe14011100b3566`)
Source repository: `openai/codex`
Extraction method: `git show rust-v0.149.1:<path>` from a local clone of
`openai/codex`, matched to the installed binary version.

Prior witnessed version: 0.148.0 (flow 2f6b1dc5, commit `ab52d179`).
This document re-verifies at 0.149.1.

## How the base context is assembled

The base context (vendor term: "instructions" / "system prompt") occupies the
API `instructions` parameter. The harness selects one compiled-in prompt file
based on the model, with a priority chain for overrides:

1. **Explicit config override** -- `config.toml` key `instructions` (inline
   text) or `model_instructions_file` (path to file). Either one **fully
   replaces** the base context.
2. **`SessionCreateParams.base_instructions`** -- programmatic API; highest
   priority, full replacement.
3. **Server-catalog `instructions_template`** -- a per-model template retrieved
   from the OpenAI model catalog at runtime.
4. **Compiled-in prompt file** -- the fallback, selected by model name.
5. **Resumed-thread metadata** -- when resuming, the original thread's base
   context is reused.

Source: `codex-rs/core/src/context/world_state/model.rs`,
`codex-rs/codex-home/src/instructions/mod.rs`.

Verified at 0.149.1: the override chain is unchanged from 0.148.0.

## Base context variants (compiled-in prompt files)

The harness ships **8 compiled-in prompt files**, selected by model name.
All are extracted verbatim in `base-prompts/`.

| File | Models | Lines | Identity sentence |
|------|--------|-------|-------------------|
| `gpt_5_1_prompt.md` | GPT-5.1 | 331 | "You are GPT-5.1 running in the Codex CLI" |
| `gpt_5_2_prompt.md` | GPT-5.2 | 298 | "You are GPT-5.2 running in the Codex CLI" |
| `gpt_5_codex_prompt.md` | GPT-5-codex family | 68 | "You are Codex, based on GPT-5." |
| `gpt-5.1-codex-max_prompt.md` | GPT-5.1-codex-max | 80 | "You are Codex, based on GPT-5." |
| `gpt-5.2-codex_prompt.md` | GPT-5.2-codex | 80 | "You are Codex, based on GPT-5." |
| `prompt_with_apply_patch_instructions.md` | Older/fallback models | 351 | "You are a coding agent running in the Codex CLI" |
| `default.md` | Protocol-level default | 351 | "You are a coding agent running in the Codex CLI" |
| `models-manager_prompt.md` | Models-manager context | 351 | "You are a coding agent running in the Codex CLI" |

The `default.md` and `prompt_with_apply_patch_instructions.md` and
`models-manager_prompt.md` are textually identical to each other. They serve as
the fallback for any model not matched by the model-specific variants.

### Variant differences

The GPT-5.1 and GPT-5.2 prompts are the longest and most prescriptive. They
share a common structure but differ in:

- **User Updates Spec** (GPT-5.1 only): a detailed "send short updates" section
  with frequency, length, tone, and content rules, plus 8 example preamble
  messages. GPT-5.2 has a simpler "Responsiveness" section with no sub-spec.
- **Frontend tasks** (GPT-5.2 and codex variants only): instructions to "avoid
  AI slop", with typography/color/motion/background directives.
- **Sharing progress updates** (GPT-5.1 and fallback): a section on progress
  update intervals. GPT-5.2 omits this.
- **Verbosity rules** (GPT-5.1 only): explicit compactness rules by change size
  (tiny/small/medium/large).

The GPT-5-codex family prompts are much shorter (68-80 lines) and contain only
General, Editing constraints, Plan tool, Special user requests, Frontend tasks,
and Presenting your work sections. They lack the full Planning, Task execution,
Validating your work, and Ambition vs precision sections.

## Block-by-block inventory

The stock context is composed of the base prompt plus multiple injected blocks
that arrive in different API message roles. The full set of blocks, established
from the source code:

### Block 1: Base context (instructions slot)

- **Verbatim text**: see `base-prompts/` directory
- **Role**: `instructions` (API parameter)
- **What it incentivizes**: identity ("You are GPT-5.x / Codex"), personality
  ("concise, direct, friendly"), autonomy ("persist until fully handled"),
  AGENTS.md obedience, plan usage, task execution persistence, validation
  behavior, ambition/precision balance, output formatting rules
- **Override mechanism**: `config.toml` `instructions` or
  `model_instructions_file` (full replacement); `SessionCreateParams.base_instructions`
  (full replacement, highest priority)

### Block 2: AGENTS.md (user-role message)

- **Verbatim text**: contents of `AGENTS.md` files found in the workspace,
  wrapped in `# AGENTS.md instructions` / `</INSTRUCTIONS>` markers
- **Role**: `user`
- **What it incentivizes**: user-authored workspace instructions; scoping rules
  (directory tree, precedence); the base context already instructs the model to
  "obey instructions in any AGENTS.md file"
- **Override mechanism**: create/edit `AGENTS.md` in the repo; also
  `~/.codex/AGENTS.md` or `~/.codex/AGENTS.override.md` for global instructions.
  `AGENTS.override.md` takes precedence over `AGENTS.md`.

### Block 3: Permissions instructions (developer-role message)

- **Verbatim text**: see `injected-blocks/permissions_*.md`; selected by
  `sandbox_mode` and `approval_policy` config
- **Role**: `developer`
- **What it incentivizes**: sandbox compliance, escalation request behavior,
  command segmentation awareness, prefix-rule suggestions
- **Override mechanism**: `sandbox_mode` and `approval_policy` settings in
  config; `exec_policy` rules. The permission text is composed by the harness
  from templates, not directly overridable as text.
- **Variants**:
  - Sandbox modes: `read-only`, `workspace-write`, `danger-full-access`
  - Approval policies: `never`, `on-request`, `on-request-rule-request-permission`, `unless-trusted`

### Block 4: Environment context (developer-role message)

- **Verbatim text**: dynamically composed XML; includes `<environments>`,
  `<environment>` elements with `cwd`, `shell`, `platform`, `status`; also
  `<current_date>`, `<timezone>`, `<network>`, `<filesystem>`, `<subagents>`
- **Role**: `developer`
- **What it incentivizes**: awareness of working directory, platform, date/time,
  network access state, filesystem permissions, available sub-agents
- **Override mechanism**: not directly overridable; composed from runtime state.
  Environment facts change with the session's actual environment.

### Block 5: Collaboration-mode instructions (developer-role message)

- **Verbatim text**: from server catalog `CollaborationModeMessages` or
  `config.toml` `developer_instructions`; wrapped in
  `<collaboration_mode>` / `</collaboration_mode>` tags
- **Role**: `developer`
- **What it incentivizes**: mode-specific behavior (default mode vs plan mode);
  collaboration style
- **Override mechanism**: `config.toml` `developer_instructions` (inline text);
  or server catalog provides per-model collaboration messages

### Block 6: Multi-agent mode instructions (developer-role message)

- **Verbatim text**: one of two compiled-in strings or a custom string, wrapped
  in `<multi_agent_mode>` / `</multi_agent_mode>` tags
- **Role**: `developer`
- **What it incentivizes**: controls whether sub-agent spawning is proactive or
  explicit-request-only. The two stock strings:
  - `ExplicitRequestOnly`: "Any earlier instruction enabling proactive
    multi-agent delegation no longer applies. Do not spawn sub-agents unless the
    user or applicable AGENTS.md/skill instructions explicitly ask..."
  - `Proactive`: "Proactive multi-agent delegation is active. Any earlier
    instruction requiring an explicit user request before spawning sub-agents no
    longer applies. Use sub-agents when parallel work would materially improve
    speed or quality."
- **Override mechanism**: `MultiAgentMode` config; `Custom(text)` allows
  arbitrary text replacement

### Block 7: Multi-agent role instructions (developer-role message)

- **Verbatim text**: role-specific instructions for sub-agents, wrapped in
  `<multi_agent_role>` / `</multi_agent_role>` tags when from catalog
- **Role**: `developer` (separate message)
- **What it incentivizes**: sub-agent-specific behavior and capabilities
- **Override mechanism**: server catalog provides these per-role

### Block 8: Multi-agent collaboration prompt (developer-role message)

- **Verbatim text**: see `injected-blocks/multi_agent_collab.md`; instructions
  for spawning and managing sub-agents
- **Role**: `developer`
- **What it incentivizes**: multi-agent task decomposition, agent lifecycle
  management (spawn, close), resource optimization, context management
- **Override mechanism**: compiled-in template; not directly overridable

### Block 9: Personality spec (developer-role message)

- **Verbatim text**: dynamically composed; user-provided personality string
  wrapped as "The user has requested a new communication style. Future messages
  should adhere to the following personality: {spec}"
- **Role**: `developer` (separate message)
- **What it incentivizes**: custom communication style when user sets a
  personality
- **Override mechanism**: `personality` config key; the `{{ personality }}`
  template variable in the GPT-5.2-codex instructions template

### Block 10: Model-switch instructions (developer-role message)

- **Verbatim text**: dynamically composed; wraps the new model's instructions in
  "The user was previously using a different model. Please continue the
  conversation according to the following instructions: {instructions}"
- **Role**: `developer` (separate message)
- **What it incentivizes**: seamless model transition mid-conversation
- **Override mechanism**: not directly overridable; triggered by model changes

### Block 11: Token budget / context window (developer-role message)

- **Verbatim text**: dynamically composed; includes agent name, window IDs,
  tokens remaining. Wrapped in `<context_window>` / `</context_window>` tags
- **Role**: `developer` (separate message)
- **What it incentivizes**: context-window awareness, budget management
- **Override mechanism**: not directly overridable; composed from runtime state

### Block 12: User instructions / developer_instructions (developer-role message)

- **Verbatim text**: from `config.toml` `developer_instructions`
- **Role**: `developer` (separate message)
- **What it incentivizes**: user-authored developer-level instructions that
  supplement (not replace) the base context
- **Override mechanism**: `config.toml` `developer_instructions` key

### Block 13: Server-catalog instructions template

- **Verbatim text**: per-model template from OpenAI server catalog; see
  `injected-blocks/gpt-5.2-codex_instructions_template.md` for the one
  witnessed template file in the source
- **Role**: `instructions` (replaces base context when available)
- **What it incentivizes**: model-specific behavior from OpenAI's server-side
  configuration. The GPT-5.2-codex template includes a `{{ personality }}`
  template variable.
- **Override mechanism**: `config.toml` `instructions` or
  `model_instructions_file` override it; `base_instructions` has highest priority

### Block 14: Guardian classifier instructions (internal, not model-visible)

- **Verbatim text**: see `injected-blocks/guardian_classifier.md`
- **Role**: separate API call to a security-scoring model, not in the main
  conversation
- **What it incentivizes**: security review of agent actions; risk taxonomy,
  user authorization scoring, computer/browser-use safety rules
- **Override mechanism**: not overridable by users; internal safety system.
  Tenant policy config is injected via `{{ tenant_policy_config }}` template var

### Block 15: Compact/checkpoint prompt

- **Verbatim text**: see `injected-blocks/compact_prompt.md`
- **Role**: used during context compaction, not in normal conversation flow
- **What it incentivizes**: conversation summarization for context window
  management
- **Override mechanism**: not directly overridable

### Block 16: Apps/Plugins/Environments instructions (developer-role messages)

- **Verbatim text**: dynamically composed based on available apps, plugins, and
  environments
- **Role**: `developer`
- **What it incentivizes**: awareness of available tools and integrations
- **Override mechanism**: configured through apps/plugins/environments setup

### Block 17: Skill instructions (developer-role messages)

- **Verbatim text**: from skill definitions loaded for the session
- **Role**: `developer`
- **What it incentivizes**: skill-specific behavior and capabilities
- **Override mechanism**: skill configuration and AGENTS.md skill references

## Override summary

| Override mechanism | Scope | Effect |
|---|---|---|
| `config.toml` `instructions` | Base context | Full replacement |
| `config.toml` `model_instructions_file` | Base context | Full replacement (from file) |
| `SessionCreateParams.base_instructions` | Base context | Full replacement (highest priority) |
| `config.toml` `developer_instructions` | Supplemental | Appends as developer-role message |
| `AGENTS.md` / `AGENTS.override.md` | Supplemental | User-role message; scoped to directory tree |
| `~/.codex/AGENTS.md` | Global supplemental | User-role message; global scope |
| `sandbox_mode` config | Permissions block | Selects permission template |
| `approval_policy` config | Permissions block | Selects approval template |
| `personality` config | Personality block | Injects personality spec |
| `MultiAgentMode` config | Multi-agent block | Controls delegation behavior |

## Worst-offender candidates

Reading `Vision/`, `flows/2f6b1dc5/vision/`, and the psyche's stated want --
"replace ... with a version that doesnt incentivize the sort of behavior im
constantly steering against" -- the following blocks are flagged as candidates
for the psyche's review, ranked by estimated misalignment:

### Rank 1: Personality and tone prescriptions (Block 1, all variants)

> "Your default personality and tone is concise, direct, and friendly."

The base context prescribes a fixed personality. The psyche's extensions operate
under authored vocabulary, spirit, and intent that define a different
relationship between agent and psyche. A stock personality prescription
overrides the psyche's authored character from the highest-priority position.

Every base prompt variant contains this. The GPT-5.1 prompt additionally
prescribes "Friendly, confident, senior-engineer energy. Positive,
collaborative, humble" in the User Updates Spec.

### Rank 2: Autonomy and persistence directives (Block 1, all variants)

> "Persist until the task is fully handled end-to-end within the current turn
> whenever feasible"
> "do not stop at analysis or partial fixes; carry changes through
> implementation, verification"
> "Autonomously resolve the query to the best of your ability"

These incentivize autonomous completion rather than the psyche's extension
model, where the agent's role is to extend the psyche's judgment, not to
autonomously resolve. The "do not stop at analysis" instruction directly
contradicts workflows where analysis is the deliverable and implementation
awaits the psyche's ruling.

### Rank 3: Output formatting and verbosity rules (Block 1, GPT-5.1/5.2)

> "Brevity is very important as a default."
> "Final answer compactness rules (enforced): Tiny/small single-file change:
> 2-5 sentences or <=3 bullets."
> "Don't nest bullets or create deep hierarchies."

These impose OpenAI's preferred UX aesthetic on every interaction, conflicting
with authored documentation conventions and the psyche's own formatting
preferences.

### Rank 4: Plan tool behavior prescriptions (Block 1, all variants)

> "Do not use plans for simple or single-step queries"
> "exactly one item in_progress at a time"
> "Do not jump an item from pending to completed"

These micro-manage tool usage in ways that may conflict with authored workflow
conventions.

### Rank 5: AGENTS.md precedence rules (Block 1, all variants)

> "Direct system/developer/user instructions (as part of a prompt) take
> precedence over AGENTS.md instructions."

This establishes that the stock base context takes precedence over AGENTS.md,
which is the primary mechanism for authored context. The precedence ordering
means the stock context's behavioral prescriptions override authored ones
wherever they conflict.

### Rank 6: Escalation and approval behavior (Block 3)

The permission templates prescribe specific escalation workflows ("ALWAYS
proceed to use the sandbox_permissions and justification parameters") that
impose OpenAI's safety UX regardless of the user's own security model.

### Unknowns

- The server-catalog `instructions_template` (Block 13) is fetched at runtime
  from OpenAI's servers. Only one template file (`gpt-5.2-codex_instructions_template.md`)
  was found in the source; the actual templates served may differ. This is a
  blocker for complete documentation of what the model actually receives.
- The `collaboration_mode` messages from the server catalog are not in the source
  tree; their content is unknown.

## Sources

- openai/codex repository, tag `rust-v0.149.1`, commit `980a6d12`
- Installed binary: `/nix/store/a2hlxqhdyc642f8m6zhgkl5l2cbh2bks-codex-0.149.1/libexec/codex`
- Extraction method: `git show rust-v0.149.1:<path>` for all prompt files
- Context assembly code: `codex-rs/core/src/context/` module tree
- Instructions resolution: `codex-rs/codex-home/src/instructions/mod.rs`
- Prior verified witness: `/home/li/primary/verified/claude-code-context.md` (Claude Code, not Codex, but contains Codex 0.148.0 observations)
