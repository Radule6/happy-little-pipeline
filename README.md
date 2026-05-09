# Happy Little Pipeline

<p align="center">
  <img src="docs/bob-ross.webp" alt="Bob Ross painting a happy little mountain" width="640">
</p>

> *"We don't make mistakes, just happy little accidents."*
> — Bob Ross

> A disciplined, repeatable workflow for shipping non-trivial features with [Claude Code](https://docs.claude.com/en/docs/claude-code/overview). Named after Bob Ross — trust the process, happy little accidents are part of the work, and you end up with a finished painting instead of a panicked deliverable.
>
> **When NOT to use:** trivial fixes (typo, single-line config, adding one log line). For those, just edit and commit. Anything else, follow the chain.

> **Note:** italicized names like *feedback_simplify_pass* or *reference_specs_gitignored* in this doc point to my private knowledge-vault notes. They're left as plain references so the structure of the pipeline is preserved; you don't need access to them to follow the workflow.

---

## The pipeline

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   1. brainstorming  ──▶  2. happy little spec  ──▶  3. user reviews     │
│                            (write spec + VC          spec + VC in        │
│                             to vault, NOT repo)      Obsidian            │
│                                                                          │
│                          │                                              │
│                          ▼                                              │
│                                                                          │
│   4. happy little plan ──▶ 5. write plan to   ──▶  6. user reviews     │
│      (TDD-shaped              vault                  plan                │
│       checklist)                                                         │
│                                                                          │
│                          │                                              │
│                          ▼                                              │
│                                                                          │
│   7. /compact checkpoint                                                 │
│      - wipe brainstorm Q&A history before the long execution phase      │
│      - parent carries only: spec path, plan path, TodoWrite state       │
│      - the spec/plan files on disk are the durable record               │
│                                                                          │
│                          │                                              │
│                          ▼                                              │
│                                                                          │
│   8. subagent-driven-development                                         │
│      (one fresh subagent per task; default = background)                │
│                                                                          │
│      Per task (parent-driven sequence):                                  │
│        a. happy little implementer (Sonnet/Opus subagent, background)   │
│           - writes failing test, implements, test green, commits        │
│           - returns structured handoff                                   │
│        b. happy little spec compliance (fresh Sonnet subagent)          │
│           - verifies code matches spec + validation contract            │
│           - returns ✅ or fix list with file:line                       │
│        c. fix blocking issues, re-review; mark task complete            │
│                                                                          │
│                          │                                              │
│                          ▼                                              │
│                                                                          │
│   9. happy little simplify (parent-invoked, ONCE, end-of-branch)        │
│      - run /simplify on the full multi-task diff                        │
│      - bundled skill: 3-agent fanout (reuse/quality/efficiency)         │
│      - reuse pass needs full diff to catch cross-task duplication       │
│      - Phase 3 auto-applies by default; stop at Phase 2 only on very    │
│        large diffs where you want findings inspection first             │
│      - lands as one "simplify sweep" commit before adversarial review   │
│                                                                          │
│                          │                                              │
│                          ▼                                              │
│                                                                          │
│  10. happy little adversarial review (Opus + ultrathink, end-of-branch) │
│      - constrained context: spec + VC + git diff + tests only           │
│      - no implementation chat history, no design rationale              │
│      - judges code against the validation contract                      │
│                                                                          │
│  11. apply blocking fixes                                                │
│                                                                          │
│  12. operational doc set                                                 │
│      - runbook in vault                                                  │
│      - unified system reference in vault (one document per system that   │
│        ties everything together)                                         │
│                                                                          │
│  13. happy little memory update                                          │
│      - project memory pointing to spec/plan/runbook/system-doc           │
│      - feedback memories for any new discipline learned                  │
│      - reference memories for any new infra fact                         │
│                                                                          │
│  14. happy little doc refresh                                            │
│      - flip status fields (current → delivered/deferred)                 │
│      - bump lastVerified: dates                                          │
│      - replace "in-progress / awaiting" prose with delivered language    │
│      - add "Status: Delivered (commit X)" banners on now-historical      │
│        design + build docs                                               │
│                                                                          │
│  15. arm Stop hook before final message                                  │
│       (touch ~/.claude/state/notify-on-stop)                             │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Validation contracts (required spec section for medium+ work)

Every spec for medium-or-larger work must include a `## Validation Contract` section listing observable, assertable success criteria. Each item is testable yes/no — not "the narration sounds natural" but "narration field is present, non-empty, ≤500 chars."

Format:

```markdown
## Validation Contract

### Positive assertions
- [ ] VC-1: <what the system MUST do>
      Verify: <command / query / test that produces yes-or-no>
- [ ] VC-2: ...

### Negative assertions
- [ ] VC-N: <what the system must NOT do>
      Verify: ...
```

Right-sizing:
- **Trivial:** no contract.
- **Small:** 3–5 lightweight assertions inline in the spec, no VC-N IDs.
- **Medium+:** formal section with VC-N IDs and verification commands.

The contract is **the** success criterion for the adversarial final review. The reviewer's checklist is the contract. This is the cleanest loop — contract written before code, judged independently after code.

**Spec review checkpoint (step 3 of the diagram):** during user spec review, specifically poke at VC observability. Flag any item that's subjective ("sounds natural"), requires interpretation, or doesn't have a clear yes/no verification command. Weak VCs are how you end up with a self-satisfying adversarial review — the implementer wrote the spec and the contract; the reviewer judges code against a contract that was tuned to be satisfiable. Spec review is the safety valve.

See *feedback_validation_contracts*.

---

## Where each artifact lives

| Artifact | Location | Why |
|---|---|---|
| Spec (design doc) | `vault/Projects/<repo>/docs/<topic>-design.md` | Repo `specs/` is gitignored locally; vault is the durable cross-machine canonical home. |
| Validation contract | `## Validation Contract` section *inside the spec* | Inline so it can't drift from the design. |
| Implementation plan | `vault/Projects/<repo>/docs/<topic>-build.md` | Same reason. Plans accompany specs. |
| Operational runbook | `vault/Projects/<repo>/docs/<topic>-runbook.md` | Triage by symptom, manual replay. |
| Unified system reference | `vault/Projects/<repo>/docs/<topic>-system.md` | One document per system; first thing to read at the start of any future session that touches this system. |
| Project memory entry | `vault/Projects/<repo>/memory/project_<topic>.md` (auto-memory) | Surfaces the system on next session start. |
| Feedback memories | `vault/Projects/<repo>/memory/feedback_<rule>.md` or `_meta/global-memory/feedback_<rule>.md` | Discipline that should persist. Cross-project rules live in global. |
| Reference memories | `vault/Projects/<repo>/memory/reference_<thing>.md` | Infra facts. |
| Code | The actual repo (`~/Desktop/<repo>`) | Obviously. |

**Don't write specs/plans/runbooks into the repo's `specs/` folder** — that folder is gitignored locally. Vault is the source of truth.

---

## Per-task discipline (the inner loop)

Each task in the implementation plan goes through this sequence. The **parent** (Claude in the main session) drives the sequence. Subagents do not invoke other subagents — that's fragile.

### Step 1 — Happy little implementer (fresh subagent, background)

- Subagent gets: full task text from the plan, scene-setting context, repo state, repo conventions, the relevant slice of the validation contract.
- Model: **Sonnet or Opus, never Haiku** (*feedback_subagent_models*)
- Default execution: **background** (*feedback_subagent_dispatch_defaults*) — you stay free to chat with the parent.
- Subagent writes the failing test first, runs it, watches it fail.
- Implements minimal code to pass.
- Re-runs tests, watches them pass.
- Commits.
- Returns a **structured handoff** (*feedback_structured_subagent_handoffs*).

### Step 2 — Happy little spec compliance (fresh reviewer subagent)

- Verifies: did the implementer build what was asked? Including the validation contract slice for this task.
- Reads the actual code (does NOT trust the implementer's report).
- Returns: ✅ Spec compliant OR ❌ list of missing/extra/misinterpreted items with file:line refs.
- If ❌: implementer fixes, then re-review.

### Step 3 — Mark task complete; move to next

Don't pause to check in with the user between tasks ("Should I continue?" wastes their time). Execute the whole plan. Stop only on real blockers.

### Why fresh subagents

Each subagent has a clean context. They don't inherit your session's history — you give them exactly what they need. This is what keeps quality up across a 10-task plan: no context pollution between tasks, no "ah I remember from earlier" assumptions.

### Note: no per-task `/simplify` and no per-task code quality reviewer

`/simplify` runs **once at end-of-branch** on the full multi-task diff, not per-task — see the dedicated section below. Standalone "code quality reviewer" subagent is subsumed by `/simplify`'s 3-agent fanout. The per-task review is **spec compliance only**.

---

## Structured subagent handoffs

Implementer + reviewer subagents must end their final message with this exact structure:

```markdown
## Status
DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT

## Files touched
- path:line — what changed

## Decisions made
- decision: <what>
  rationale: <why>

## Risks introduced
- risk: <what>
  mitigation: <how / "unmitigated">

## Validation contract status
- VC-N: pass | fail | skipped (reason)

## Blockers (only if status is BLOCKED)
- blocker: <what>
  impact: <what's blocked>
```

Read-only research subagents (Explore, ad-hoc lookup) are exempt — they naturally return findings.

The format makes composition cleaner. When dispatching three subagents and reading back three reports, the consequential bits live in known places, not buried in prose.

See *feedback_structured_subagent_handoffs*.

---

## Happy little simplify (end-of-branch, parent-invoked, ONCE)

After all per-task spec compliance is green, the parent invokes `/simplify` once on the full branch diff. This is the ONLY simplify pass — there is no per-task simplify.

**Why end-of-branch, not per-task:**
- The reuse pass needs the full feature diff to catch cross-task duplication. Task-by-task it can't see that tasks 2 / 5 / 7 each rewrote the same helper.
- Per-task adds 3 agents × N tasks of overhead with mostly premature findings — later tasks may legitimately reuse the "duplicated" pattern.
- Trade-off accepted: one fat "simplify sweep" commit instead of clean per-task commits. Reads well in git history as a deliberate cleanup phase before adversarial review.

**How:**
- Parent runs `/simplify` (bundled Claude Code skill, not a subagent, not a custom skill on disk).
- Skill spawns 3 review agents in parallel: code reuse, code quality, efficiency.
- **Phase 3 runs by default** — the skill auto-applies the fixes the 3 agents recommend.
- Stop at Phase 2 only on very large diffs where you want findings inspection before a sprawling auto-apply.
- Run **after** all per-task tests are green, **before** the adversarial reviewer.

**Why parent-invoked:** subagents cannot reliably nested-dispatch `/simplify` (subagent fans out 3 more agents — fragile, may not be supported). Implementer narrating "I'll run simplify now" is theater.

See *feedback_simplify_pass*.

---

## Happy little adversarial review (after all tasks done)

Dispatch ONE Opus subagent at the end of the branch to judge the diff against the validation contract.

**Constrained context — what the reviewer gets:**
- The spec (especially `## Validation Contract` section)
- `git diff <base>...<head>`
- The test files
- Repo conventions (CLAUDE.md, key reference memories)

**What the reviewer does NOT get:**
- This conversation's history
- The build plan rationale
- Design notes or decision logs
- Any "here's what we decided and why" framing

The reviewer is adversarial: it judges code against the contract, not against our reasoning. If we under-specified the contract, the reviewer surfaces that. If the diff diverges from the spec, the reviewer flags it.

**Use `ultrathink`** — this is one of two places `ultrathink` is reserved for (*feedback_subagent_dispatch_defaults*).

**Dispatch prompt template (use this exactly, fill the bracketed parts):**

```
Adversarial final review of branch <branch-name>.

Read these files (and ONLY these files):
1. The spec at <vault path to design doc> — pay close attention to the `## Validation Contract` section.
2. `git diff <base-sha>...<head-sha>` (run this; the diff is what you're judging).
3. The test files for this work: <list of test paths>.
4. Repo conventions: ~/.claude/CLAUDE.md and <project>/memory/MEMORY.md for any rules that apply.

Do NOT consult: the build conversation history, the implementation plan rationale, decision logs, or any "here's what we decided and why" framing. Your job is to judge code against the contract, not to validate our reasoning.

Use ultrathink for this review.

For each VC-N item in the contract, return:
- VC-N: pass | fail | ambiguous (explain) — file:line refs

Plus structured findings:
- Critical: things that must be fixed before merge
- Important: should be fixed
- Minor: nits

End with the structured handoff format.
```

After review: apply blocking fixes only, defer the rest as follow-up tickets.

See *feedback_adversarial_final_review*.

---

## After implementation: operational + reference docs

This is the step most likely to be skipped. **Don't skip it.**

### Runbook (vault)

Operator-facing. "It's 7am, the morning run errored — what do I do?" Triage by symptom. List of every command needed (bootstrap, dry-run, real run, replay, health probe). Cost / volume baselines. Permission gaps documented.

### Unified system reference (vault)

The single most valuable doc per system. It's what you re-read at the start of any future session. Contains:
- TL;DR (one paragraph)
- Architecture diagram (ASCII)
- Step-by-step pipeline timing
- Storage schema with examples
- Trigger surfaces
- Failure handling matrix
- Configuration / secrets list
- Key file map
- Consumers (current + future)
- Deferred work + data gaps

### Memory entries

Auto-memory surfaces these on next session start, so the next time you open the project Claude already knows the locked decisions:

- **`project_<topic>.md`** — locked decisions + pointer to the system doc + spec + plan + runbook
- **`feedback_<rule>.md`** — any new discipline learned
- **`reference_<thing>.md`** — any new infra fact

---

## Happy little doc refresh — living-doc freshness at end of work

Step 14 in the diagram. Catches doc drift at the moment work ends, when context is fresh — so you don't need to hand-hold a future session through a staleness sweep two weeks later.

**What to refresh:**

- **`project_*.md` memories:** flip `status:` field (`current` → `delivered` / `deferred` / `superseded`); update `lastVerified:` to today; replace any "current / awaiting / in-progress" prose with delivered/superseded language.
- **`docs/*-system.md`:** update if the architecture changed — new components, removed layers, new failure modes.
- **Project `README.md`** "Active work" / "Specs" sections: flip current → delivered.
- **Now-historical `docs/*-design.md` and `*-build.md`:** add a one-line banner at the top: `> **Status:** Delivered (commit <sha>).` Don't rewrite the body — the design doc is a frozen record.

**What NOT to do:**

- Do NOT bake snapshot lists into living docs ("Uncommitted UI polish (working tree): [9 items]"). That's git's job — the doc rots within hours.
- Do NOT update `docs/*-design.md` or `*-build.md` content after delivery. They're frozen records of what was specified, not living state.

**Living vs. frozen** — every doc declares its decay class via frontmatter `status:`:

| Status | Meaning | Update frequency |
|---|---|---|
| `living` | Currently in motion | Re-verify weekly; refresh at end of any work touching it |
| `delivered` | Work completed and shipped | Update only if architecture changes after delivery |
| `frozen` | Historical record (design, build) | Never updated — append a "Status: Delivered" banner once and leave |
| `superseded` | Replaced by a later doc | Add pointer to successor; otherwise leave |
| `deferred` | Work paused / parked | Update only when picked back up |

See *feedback_living_doc_freshness*.

---

## Discipline rules

These came out of building real systems. Follow them.

### Subagent dispatch is the default
- During pipeline execution, **dispatch is automatic**. Don't ask "should I dispatch a subagent here?" — just do it.
- Default execution mode: **background**. You stay free to chat with the parent while subagents grind.
- Foreground only when the next step blocks on the result (e.g., the adversarial final reviewer at end-of-branch).

### `ultrathink` is reserved
- Use only in: (a) brainstorm/spec phase when design is genuinely ambiguous, and (b) the adversarial final reviewer.
- Do NOT use for: implementer subagents, per-task spec compliance reviewers, simplify-internal agents.

### Subagent model floor
- **Quality-sensitive subagents:** Sonnet or Opus. Never Haiku.
- **Adversarial final review:** Opus.
- **Mechanical implementers (clear spec, isolated change):** Sonnet is enough.

### `/simplify` is parent-invoked, end-of-branch only
- `/simplify` is a bundled Claude Code skill (not a subagent, not a custom skill on disk).
- The parent (main session) invokes it; subagents cannot reliably nested-dispatch it.
- Run **once at end-of-branch** on the full branch diff, **never per-task**. The reuse pass needs full diff to catch cross-task duplication.
- **Phase 3 runs by default** (auto-apply). Stop at Phase 2 only on very large diffs where you want findings inspection first.

### Validation contracts before code
- Every medium+ spec includes a formal `## Validation Contract` section.
- Small specs include 3–5 lightweight assertions inline.
- The contract is the adversarial reviewer's checklist.

### Adversarial final reviewer with constrained context
- One Opus + ultrathink subagent at end-of-branch.
- Gets: spec + VC + diff + tests. NOT: chat history, design rationale.
- Judges code against the contract, not our reasoning.

### Structured subagent handoffs
- Implementer + reviewer subagents end with: Status / Files touched / Decisions / Risks / VC status / Blockers.
- Read-only research is exempt.

### Living-doc freshness
- `project_*.md` and `*-system.md` describe stable architecture; for state, run `git log <branch>`.
- Every living doc has frontmatter `status:` + `lastVerified:`.
- Step 14 of the pipeline refreshes living docs at end of work — flip status, bump dates, banner historical docs.

### Don't read `.env`
- Never `Read`, `Edit`, or `Bash` the user's `.env` without explicit scoped permission.

### Don't push without review
- Never `git push` to a remote without explicit user confirmation per push.

### Stop hook arming
- Stop hook is sentinel-gated.
- Arm only before the **final** message of an autonomous pipeline (not between tasks).
- `touch ~/.claude/state/notify-on-stop`

### Spec/plan placement
- Vault for canonical. Repo `specs/` is gitignored locally and shouldn't be the source of truth.

### TDD red→green→refactor
- Failing test FIRST. Run it, watch it fail. Then implement.
- This catches "the test was wrong" bugs that pass-first writes silently miss.

### Continuous execution
- Don't pause to check in between tasks during subagent-driven execution.
- Stop only on: BLOCKED, ambiguity that genuinely prevents progress, all tasks done.

### Tier the deferred work explicitly
- Use Tier 1 / 1.5 / 2 / 3 framing.
- Tier 1 = must ship before any prod use.
- Tier 1.5 = blocking gaps the final review surfaces.
- Tier 2 = important for sustained operation.
- Tier 3 = polish, defer.

---

## Anti-patterns (what NOT to do)

These came from real mistakes on a working project.

| Anti-pattern | What we learned |
|---|---|
| Picking a worse abstraction "just because" | When implementing detectors, the first pass overrode `model_copy` to make tests pass. Better fix: change the tests, not the public API surface. **Don't change semantics of public APIs to paper over test design.** |
| Optimizing for the wrong scale | Almost suggested provider Batch API (24h SLA) for a 10/day volume. **Match tooling to actual scale.** Concurrent in-process parallelism is right for small-to-medium; provider Batch API is right for >10K/day overnight. |
| Using "1-24 hour" ranges in answers | Imprecise commitments. Always cite the actual SLA + what's typical. **Concrete numbers, with caveats.** |
| Pre-computing per-user data without thinking | First instinct for watchlist insights was to store one row per user×basket×day. Wrong. **For per-user derived data, compute on-demand and cache by composite key (user_id, input_hash, date_iso).** |
| Storing user-mutable data in ES | ES is bad for transactional writes. User data (baskets, watchlists, configs) belongs in Postgres. ES is for derived/append-only/searchable. |
| Building the wrong shape | One-time, detectors emitted 25 per-event alerts/day. The product spec wanted 3 polished daily reports. **Read the product spec carefully before locking the data model.** |
| Hand-waving failure handling | First-pass `bulk_upsert` raised on any error. Final review caught this — partial success is the realistic case. **Always think about per-item failure isolation when batching.** |
| Hardcoded retry / batch values | The "30s timeout, 3-attempt retry, 1024 max tokens" numbers should be config-able. Document this as a known gap. |
| Implementer subagents narrating "I'll run simplify" | They can't actually invoke `/simplify` — it's a bundled skill that fans out 3 agents itself. **Parent runs `/simplify` once at end-of-branch, not per-task, and not from a subagent.** |
| Per-task `/simplify` | Adds 3 agents × N tasks of overhead with mostly premature findings; reuse pass can't see cross-task duplication task-by-task. **Run `/simplify` once at end-of-branch on the full diff.** |
| Final reviewer with full context | Reviewer that sees our design rationale rubber-stamps it. **Constrain context: spec + VC + diff + tests only.** |
| Snapshot lists in living docs | "Uncommitted UI polish (working tree): [9 items]" — git is the source of truth for state, not memory. **Living docs describe architecture; for state, run `git log <branch>` and read commits.** |
| Status prose without `lastVerified` | "Status: current / awaiting / shipped" without a frontmatter date rots silently. **Always pair status framing with a `lastVerified:` field so future sessions know when to distrust it.** |

---

## When to deviate

The full pipeline is overkill for:

- Single-line bug fixes
- Renaming a variable
- Adding one log line
- Trivial config changes
- Doc-only updates

For those: edit, test, commit, push. No spec, no plan, no subagent.

The pipeline kicks in when:
- Multiple files
- New abstractions
- Schema changes
- Cross-cutting impact (auth, logging, error handling)
- Anything that needs an FE handoff
- Anything operator-facing

When in doubt: write the brainstorming spec. If the spec ends up being three sentences, you saved nothing — but you also lost nothing.

---

## Skills referenced (in invocation order)

1. `superpowers:brainstorming` — design phase
2. `obsidian-vault` — write spec/plan/runbook to vault, set up project folders
3. `superpowers:writing-plans` — TDD-shaped implementation plan
4. `superpowers:subagent-driven-development` — execute the plan
5. `superpowers:test-driven-development` — used inside subagent
6. **`/simplify`** — bundled Claude Code skill, parent-invoked **once at end-of-branch** on the full diff (3-agent fanout: reuse / quality / efficiency)
7. `superpowers:requesting-code-review` — adversarial final cross-cutting review
8. `superpowers:finishing-a-development-branch` — wrap, push, MR

Each skill has its own canonical instructions; this doc tells you the **order and discipline** that ties them together.

---

## About

This is a personal feature-build pipeline I follow when working with [Claude Code](https://docs.claude.com/en/docs/claude-code/overview). The italicized references throughout (e.g. *feedback_simplify_pass*) point to private notes in my knowledge vault — they're left as readable names so the workflow's structure is visible, but you don't need access to them to follow the chain. The full pipeline is opinionated and prescriptive on purpose: the point is to stop re-inventing the workflow at the start of every project.

Feedback and forks welcome.
