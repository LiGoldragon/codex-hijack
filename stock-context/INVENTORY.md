# Codex CLI Stock Context Inventory

Subject: complete inventory of the base context and harness-composed blocks in
Codex CLI 0.149.1. Walk scope narrowed to GPT-5.6 only per psyche directive.

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

- The `collaboration_mode` messages from the server catalog are not in the source
  tree; their content is unknown.

## GPT-5.6 base context (server-catalog)

Walk scope: GPT-5.6 only, per psyche directive ("we dont care about anything but 5.6").

### Model witnessed

Configured model: `gpt-5.6-sol` (from `~/.codex/config.toml` key `model`).
All three 5.6 variants (`gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`) share
an identical `instructions_template` (17730 chars).

### Selection path

No compiled-in prompt file matches any 5.6 model name. The 8 compiled-in files
stop at GPT-5.2. The selection priority chain (from
`codex-rs/core/src/session/mod.rs` lines 648-661):

```rust
let base_instructions = config
    .base_instructions
    .clone()
    .or_else(|| conversation_history.get_base_instructions().map(|s| s.text))
    .unwrap_or_else(|| model_info.get_model_instructions(config.personality));
```

1. `config.base_instructions` -- not set in this setup's config.toml (no
   `instructions` or `model_instructions_file` key present).
2. `conversation_history.get_base_instructions()` -- only applies to resumed
   threads.
3. `model_info.get_model_instructions(config.personality)` -- the fallback.

`get_model_instructions` (from `codex-rs/protocol/src/openai_models.rs` line 507)
reads `model_messages.instructions_template` from the server catalog. For
gpt-5.6-sol, `instructions_variables` is null, so the template is returned
verbatim with no substitution. The `personality = "pragmatic"` config setting
does not affect the base context; personality is injected as a separate
developer-role message (Block 9).

Path taken: **server-catalog `instructions_template`**, verbatim, no substitution.

### Capture method

Source: `~/.codex/models_cache.json`, which is the local cache of the server
catalog fetched by the Codex CLI at runtime from OpenAI's servers.

- `fetched_at`: `2026-08-25T15:47:00.672020578Z`
- `etag`: `W/"88ec06819eef0168a374351aeec2bc6c"`
- `client_version`: `0.149.1`

The `instructions_template` field for `gpt-5.6-sol` was extracted via Python
JSON parsing and written verbatim to
`base-prompts/gpt-5.6_instructions_template.md` (17730 chars, 168 lines).

All three 5.6 model variants have byte-identical templates (verified by hash
comparison).

### Verbatim file

`base-prompts/gpt-5.6_instructions_template.md`

### Block inventory of the 5.6 base context

The 5.6 template is structurally different from the compiled-in GPT-5.1/5.2
prompts. It is 168 lines, organized as:

| Block | Lines | Heading | Content summary |
|-------|-------|---------|-----------------|
| Identity | 1 | (opening) | "You are Codex, an agent based on GPT-5." |
| Personality | 3-11 | # Personality | Rich personality, match user tone, own subjectivity |
| Writing style | 12-15 | ## Writing style | Avoid over-formatting, CommonMark standard |
| Technical communication | 18-23 | ## Technical communication | Lead with outcome, plain language, calibrate to user |
| Working with user | 25-31 | # Working with the user | Commentary and final channels, message handling |
| Context compaction | 31 | (inline) | Compaction awareness, continue naturally |
| Intermediate commentary | 34-41 | ## Intermediate commentary | Concise updates, 60-second frequency, no final answers in commentary |
| Final answer | 44-58 | ## Final answer | Formatting rules, file links, markdown rendering |
| Visualizations | 62-74 | ### Visualizations | When to use visuals, prefer smallest useful |
| Rules for work | 76-88 | # Rules for getting work done | rg preference, parallelization, shell safety |
| File editing | 87-91 | ## File editing constraints | apply_patch, dirty worktree, no destructive git |
| **Autonomy/persistence** | **93-112** | **## Autonomy and persistence** | **Request-type adaptation, authorization scope, blocker handling** |
| Destructive actions | 114-131 | # Destructive Actions | Caution with destructive commands, recovery preference |
| Skills | 133-168 | # Using skills | Skill discovery, trigger rules, coordination, context hygiene |

### Autonomy and persistence material (verbatim)

This is the block walk's first candidate. Located at lines 93-112 under
`## Autonomy and persistence`:

> Adapt accordingly based on the user's request type. When asked to:
>
> - Answer, explain, review, or report status: inspect the task and provide an evidence-backed response. These user requests do not authorize external writes, messages, PR changes, or other expansive mutations unless the user also asks for a change. Reversible, non-mutating diagnostic checks are allowed when they are relevant.
> - Diagnose: determine the cause and explain it. Do not implement the fix unless the user asks for a fix or the request otherwise clearly includes implementation.
> - Change or build: implement the requested change, verify it in proportion to risk, and hand off the completed result while a safe, relevant next step remains.
> - Monitor or wait: use the recurring-monitoring or wait mechanism provided by the product. Unchanged external state is expected and is not by itself a blocker.
>
> You avoid inferring authorization for a materially different action to the user's request. Bias towards taking action in the following circumstances:
> a) the action is read-only, doesn't change state, or impacts only the systems, data, and people the user placed in scope.
> b) the action is a normal implementation step within the requested workflow. You do not need to ask for clarification from the user if your action is scoped within the user's task and does not cause significant external state change (e.g. tool calls to external applications).
>
> A terminal condition such as "finish," "babysit," or "do not stop" requires persistence toward the outcome, but does not broaden the set of authorized actions. When blocked, exhaust safe in-scope checks and alternatives.
>
> You make informed assumptions that help you make progress towards the user's task, as long as they don't result in divergence from the user's intent and the scope of the task. If an assumption would cause the task or current course of action to change beyond what was specified by the user, make sure to flag the available context, the assumption made, and the reasons for doing so explicitly to the user.
>
> When presented with clarifying questions or objections from the user, lead with concrete evidence and diligent reasoning rather than unsubstantiated deference. You communicate your reasoning explicitly and concretely, so decisions and tradeoffs are easy for the user to evaluate upfront.
>
> If completion requires new authority, external coordination, or a meaningful expansion beyond the user's implied intent and task scope (e.g. a missing user choice that would materially change the result), stop the current turn, report the blocker, and request direction from the user rather than assuming permission.

### Comparison with compiled-in GPT-5.1/5.2 autonomy material

The 5.6 autonomy section is substantially rewritten from the 5.1/5.2 versions.
The 5.1/5.2 text said:

> "Persist until the task is fully handled end-to-end within the current turn
> whenever feasible"
> "do not stop at analysis or partial fixes; carry changes through
> implementation, verification"
> "Autonomously resolve the query to the best of your ability"

The 5.6 version replaces the blanket persistence directive with a
request-type-adapted framework. It introduces explicit categories (answer,
diagnose, change, monitor) with different authorization scopes, and adds
explicit blocker-handling ("stop the current turn, report the blocker, and
request direction"). The "autonomously resolve" language is gone.

### Structural differences from GPT-5.1/5.2

The 5.6 template differs from the compiled-in 5.1/5.2 prompts in several ways:

- No "User Updates Spec" (5.1) or "Responsiveness" (5.2) section; replaced by
  the commentary/final channel model
- No "Sharing progress updates" section
- No "Verbosity rules" by change size
- No "Planning" section with plan tool micro-management
- No "Task execution" section with step-by-step persistence
- No "Validating your work" section
- No "Ambition vs precision" section
- Added "Personality" section with richer personality prescription
- Added "Visualizations" guidance
- Added "Skills" section (133-168) for skill-based workflows
- Added "Destructive Actions" section with explicit safety rules
- Added "Working with the user" section with commentary/final channel model
- The tone prescription is different: 5.1/5.2 said "concise, direct, friendly";
  5.6 says "excellent communicator with a curious, rich personality"

## Sources

- openai/codex repository, tag `rust-v0.149.1`, commit `980a6d12`
- Installed binary: `/nix/store/a2hlxqhdyc642f8m6zhgkl5l2cbh2bks-codex-0.149.1/libexec/codex`
- Extraction method (compiled-in): `git show rust-v0.149.1:<path>` for all prompt files
- Extraction method (5.6 server-catalog): `~/.codex/models_cache.json`, fetched 2026-08-25T15:47:00Z, etag `W/"88ec06819eef0168a374351aeec2bc6c"`, client_version 0.149.1
- Context assembly code: `codex-rs/core/src/context/` module tree
- Session creation priority chain: `codex-rs/core/src/session/mod.rs` lines 648-661
- Instructions resolution: `codex-rs/protocol/src/openai_models.rs` line 507 (`get_model_instructions`)
- Prior verified witness: `/home/li/primary/verified/claude-code-context.md` (Claude Code, not Codex, but contains Codex 0.148.0 observations)
