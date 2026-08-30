# Block marks — the 5.6 walk

The stock context is reviewed block by block, each block marked for
replacement or deletion. Scope ruling (2026-08-25): only 5.6 — the
template actually served to the 5.6 models
(`base-prompts/gpt-5.6_instructions_template.md`). Marks are the
psyche's rulings, recorded by flow 4ddc321d.

| Block | Lines | Mark |
|-------|-------|------|
| Personality | 3–9 | **Replace** — replacement TBD |
| Working with the user + Intermediate commentary | 23–41 | **Replace** — at least in parts |
| Using skills | 133–167 | **Delete** — whole block |

## Global rulings for the replacement context

- 2026-08-26 — vocabulary: every occurrence of agent/subagent in the
  replacement context becomes flow/subflow, with a line explaining the
  term (perhaps equating it with agent, since models are trained on
  that term, and instructing use of flow terminology henceforth). The
  grounding: a flow is a flow of thought; an intelligence is not a
  single flow but a multitude of them; "agent" entails subjectivity
  and misplaces it onto the single flow. The psyche's verbatim words
  are in flows/4ddc321d/vision/flow.md.

- 2026-08-26 — commentary: reserved for very rare cases, to minimize
  the context cost it creates. The psyche's verbatim words
  (flows/4ddc321d/vision/contextStrata.md): "I think we should reserve
  it for very rare cases, minimize the cost that it creates on
  context." Ground (flows/4ddc321d/reports/codexChannels.md,
  reports/commentaryDiscouragement.md): commentary and final are the
  same stratum — assistant-role messages differing only by a phase tag;
  commentary is replayed every turn until compaction at full context
  cost; no vendor lever exists for commentary frequency, so the
  base-context instruction is the mechanism; the known risk of a
  rare-commentary instruction is substitution (narrating instead of
  acting, blocking questions left unasked), mitigated by pairing it
  with "blocking questions go to the final channel". Client-side
  filtering of prior-turn commentary from replayed history is a
  separate, later engineering option.

## Working with the user + Intermediate commentary (2026-08-30)

Ruling: "mark as replace, at least in parts."

The block (template lines 23–41 of
base-prompts/gpt-5.6_instructions_template.md: the "# Working with
the user" head through the "## Intermediate commentary" subsection,
ending with the "Never praise your plan" line; the "## Final answer"
subsections 43–74 are not part of this mark) mixes harness facts the
model cannot infer with impositions. The harness facts worth keeping
as plain statements: two channels exist — commentary for updates,
final ends the turn; commentary collapses after the final answer;
mid-turn user messages arrive; compaction happens. The impositions
being replaced: the 60-second commentary cadence, the
post-compaction mandate to "continue naturally and make reasonable
assumptions about anything missing from the summary", the mid-turn-
message arbitration micro-procedure, and the prohibition-form "Never
praise your plan" line. Replace keeps the facts; extent of
replacement within the block not yet ruled ("at least in parts").

## Using skills (2026-08-26)

Ruling: "yes, mark it a delete." — the whole block, subsuming the
earlier line mark on "Do not carry skills across turns unless
re-mentioned." ("this is definitely a removal.").

The reasoning: with the block gone, the harness still injects the
skills catalog and $-mentioned skill bodies, and the authored strata
carry the skill discipline — the expected behavior over that context
is the desired behavior, so by the ruled principle ("Removal is
better than addition, when the expected behavior is the desired
behavior") the block's procedure adds nothing and its rituals cost
context. The only mechanically loaded content (non-filesystem skill
access, alias expansion) is dead weight in this setup, whose skills
are filesystem-backed.

## Personality (2026-08-26)

Ruling: "mark to be replaced, replacement tbd."

The diagnosis behind the mark: subjectivity itself stands — "the
psyche is a bunch of internal dialogues; human think by talking to
themselves. so the subjectivity isnt a problem, but that block is way
more opiniated than that, which is the problem with it." The offense
is the vendor-authored character (tastes, "old friend" register,
performance goals) prescribed from the top stratum, not the presence
of a conversing subjectivity. The replacement states the subjectivity
plainly and leaves its character to the authored strata.
