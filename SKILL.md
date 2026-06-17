---
name: pipeline
description: "Development pipeline for non-trivial tasks. Optional Stage 0 (task framing) → Analyst (sub-agent) → Plan (main + /think) → Implement (main + narration) → Review (sub-agent) → Fix loop → Final summary (with optional /review pass on diff). Each stage hands off via numbered markdown files in tmp/<slug>/. Stage 0 handles three entry shapes: from a ticket ID, from a backlog file entry, or from a fresh description (with the option to promote to a ticket or backlog later). Adapts to workspace-specific conventions via contexts/<workspace>.md overlays. Triggers: '/pipeline', 'запусти полный цикл', 'пройди задачу полностью', 'хочу пайплайн', 'full pipeline', 'do this properly', 'end-to-end'."
trigger: /pipeline
---

# /pipeline

Delivery pipeline with explicit human checkpoints between stages and a markdown artifact trail in `tmp/<slug>/`. Six core stages (Analyst → Plan → Implement → Review → Fix loop → Final) plus an optional Stage 0 at the front. The pipeline trades upfront setup time for fewer wrong rabbit holes, less scope drift, and an audit trail of what was decided when.

**This skill is for non-trivial work.** Trivial typos, one-liners, and exploratory probes should bypass it.

The core is workspace-agnostic. Workspace-specific conventions (ticket system, code contracts, file layouts, people, deploy gates) live in **overlays** under `~/.claude/skills/pipeline/contexts/<workspace>.md`. The orchestrator detects the workspace at Stage 0 and injects overlay rules into sub-agent prompts.

---

## Setup (once, at start)

1. **Detect workspace context**:
   - List `~/.claude/skills/pipeline/contexts/*.md` (if any).
   - For each, read its "Detection signature" section and evaluate the check against the current `pwd`.
   - If exactly one overlay matches, load its content into context for the rest of the run. Refer to it as **the overlay** below.
   - If none match, proceed without an overlay (personal/other namespace).
   - If multiple match (shouldn't happen), pick the one mentioned in the user's most specific path and tell the user.

2. Derive `<slug>`:
   - If a ticket ID is present (Linear `ENG-415`, Jira `PROJ-12`, GitHub `#123`) → slug = lowercased ID, normalized to kebab (e.g. `eng-415`, `proj-12`, `issue-123`).
   - Otherwise → content-derived kebab-case, 3–5 words: `swap-restyle`, `card-empty-state`, `auth-token-refresh`. Confirm with the user if ambiguous.

3. **Pick the `tmp/` root**:
   - If the overlay specifies a root (e.g. the workspace uses `<workspace-root>/tmp/`) — use that.
   - Otherwise default to `<repo-root>/tmp/` (where repo-root = `git rev-parse --show-toplevel`).
   - Fall back to `./tmp/` if not in a git repo.

4. `mkdir -p <tmp-root>/<slug>/`. Folder contains numbered stage files: `00-prompt.md`, `01-analysis.md`, etc. The numeric prefix is literal so `ls` sorts in order.

5. If the user opened with raw task text, drop to **Stage 0** to frame it. If they handed you a ticket ID directly, Stage 0 collapses to a single fetch — see L1 below.

6. **Classify task type.** Derive `task-type ∈ {design, backend-contract, bugfix, refactor, mixed}` and write it to `00-prompt.md` under a `## Task type` heading. Do this for **all** entry shapes (L1/L2/L3) — even when L1 skips the interview, this single cheap step never collapses.
   - **design** — touches Figma / restyle / new component / pixel-fidelity. Signals: a `figma.com` URL, a `fileKey`+`nodeId`, or words рестайл / вёрстка / макет / компонент / дизайн.
   - **backend-contract** — `register_ajax` events, CDN mint, API request/response shape.
   - **bugfix / refactor / mixed** — otherwise.
   - If `design` (or `mixed` with a visual part) → **all blocks marked `⟦DESIGN⟧`** in Stages 1–3 and 6 become MANDATORY, not optional.

7. **Build the stage progress tree** (via `TaskCreate`/`TaskUpdate`). Create one task per stage you expect to run; as a stage starts, expand it into child sub-step tasks and update them as you go, so the live tree shows **granular nested progress** instead of one "Stage N" line:
   - **Stage 1 (Analyst):** for `design` — `Figma map` · `Connected flows` · `Scope decisions (A/B/C)` · `Existing-logic inventory`; otherwise the base Analyst sections.
   - **Stage 3 (Implement):** one sub-step per component / per Figma layer.
   - **Stage 4 (Review) / Stage 6 (Final):** one sub-step per review dimension / wrap-up action.
   Update sub-steps to `completed` the moment each is done — the tree is the user's real-time view and interrupt point.

8. **Baseline check (git freshness) — before any analysis.** For each repo the task will touch (the cwd repo + any others named in the prompt or in the overlay's active zones): `git -C <repo> fetch -q`, then `git -C <repo> rev-parse --abbrev-ref HEAD` and `git -C <repo> rev-list --left-right --count HEAD...origin/<default-branch>`. If the tree is on a feature branch or **behind** the default branch, **STOP** and tell the user the repo / branch / ahead-behind counts and ask whether to analyze this tree or switch/fast-forward first. Silently analyzing a stale or wrong-branch tree invalidates the entire Stage 1 (a real failure mode — a full Analyst round was once wasted on an outdated branch). The Analyst then records the resolved `branch @ shortHEAD` in the header of `01-analysis.md` as the explicit baseline of record.

---

## Stage 0 — Task framing (optional)

**Goal:** ensure `tmp/<slug>/00-prompt.md` is populated cleanly before the Analyst runs. Three entry shapes; pick by what the user's opening looks like.

**Ticket backend:** if the overlay specifies a ticket system (Linear MCP, Jira, GitHub Issues via `gh`) use that. Otherwise ask the user what to use for any L1/promotion step; if they say none, skip ticket operations entirely.

**Backlog file detection:** at start, look in repo root (and overlay-specified project subdirs) for any of: `BACKLOG.md`, `BACKLOG.local.md`, `TODO.md`. The first found is the active backlog file.

### L1 — From an existing ticket (user opens with an ID or URL)

Examples: `/pipeline ENG-415`, `/pipeline #123`, `/pipeline https://linear.app/.../...`, `/pipeline https://github.com/<org>/<repo>/issues/123`.

1. Slug = lowercased ID.
2. Fetch the ticket via the overlay's ticket backend:
   - Linear MCP: `mcp__linear-server__get_issue`
   - GitHub Issues: `gh issue view <num> --json title,body,comments`
   - Other: as specified in overlay
3. Write `tmp/<slug>/00-prompt.md`:
   - Ticket ID + URL
   - Title
   - Description verbatim
   - Latest 3 comments verbatim (if relevant)
   - Date stamp
4. Skip the rest of Stage 0, jump to Stage 1.

### L2 — From a backlog file entry

Examples: «возьми задачу X из бэклога», «из BACKLOG.md: real prices».

1. Locate the entry in the detected backlog file. If ambiguous, list candidates by H3 (or H2) headings and ask user to pick.
2. Slug: if the entry already references a ticket ID, use that lowercased; otherwise kebab-case from the entry title.
3. **Ask the user: «завести тикет?»** Always ask, never assume. The user may want to keep it local.
   - If yes → create via overlay's ticket backend, seeding title + description from the backlog entry. Let the user edit before posting. Capture new ID.
   - If no → task stays local; mid-flow promotion remains available.
4. Write `tmp/<slug>/00-prompt.md`:
   - Backlog excerpt verbatim with `file:line` reference back to the backlog file
   - Ticket ID + URL if created
   - Date stamp
5. Continue to Stage 1.

### L3 — From scratch (user describes a new task)

Examples: «хочу починить X», «надо сделать Y».

1. **Lightweight interview**, only if the opening description is thin (covers <2 of: what / where / goal). Ask a max of 4–6 short questions, skip any the user already answered:
   - Что делаем (одна фраза)?
   - В каком проекте / каких файлах живёт?
   - Как воспроизвести текущее состояние?
   - Какой ожидаемый результат?
   - Скоуп — урезаем или сразу полностью?
   - Срок / приоритет?
2. Slug = content-derived kebab-case from the agreed essence. Confirm with user.
3. **Ask the user: «завести тикет, добавить в backlog file, или просто работаем?»** Three options:
   - **Ticket** → create via overlay's backend immediately.
   - **Backlog** → append a new entry to the detected backlog file following its existing style (H3 heading + sections from sibling entries). If **no** backlog file exists, ask once: «создать `BACKLOG.md` в корне?». If yes, create it with a brief intro and the entry. If no, fall through to "just go".
   - **Just go** → only `tmp/<slug>/00-prompt.md`. Quick path.
4. Write `tmp/<slug>/00-prompt.md`:
   - Task description (user's words + interview answers, edited together)
   - Ticket ID/URL if created, OR backlog file:line if appended, OR neither
   - Date stamp
5. Continue to Stage 1.

### Mid-flow promotion to ticket or backlog

At **any later stage (1–6)** the user may say «давай заведём тикет» / «добавь в бэклог». Steps:

1. Seed the ticket/backlog body from the latest stage artifact (past Stage 1 → `01-analysis.md`, past Stage 2 → `02-plan.md`, past Stage 3 → `03-implementation.md`).
2. Show the draft title + description; let the user edit before posting.
3. Append the new ID + URL (or backlog `file:line`) to `tmp/<slug>/00-prompt.md` under a `## Linked mid-flow (YYYY-MM-DD)` heading. Don't rewrite the original prompt section.
4. **Don't rename the slug** — keep stability for the rest of the pipeline files.
5. Confirm to the user, continue from where you were.

### Glossary resolve (all entry shapes, before Stage 1)

Scan `00-prompt.md` for domain nouns that do **not** appear in the codebase (quick `grep` / graphify). Transcribed or voice input often mangles product terms, and the Analyst then burns time chasing a thing that doesn't exist. For each unresolved term, confirm its meaning with the user in one line BEFORE Stage 1 (real examples: "trunk wallet" → actually **Turnkey**; "indexer" → an **external** team's service, not in this repo). Record each resolution in `00-prompt.md`. Skip terms that already map to a code symbol.

---

## Stage 1 — Analyst (sub-agent)

**Goal:** gather all context the planner needs, without polluting the main session's context with file dumps.

**This stage is the `fix-issue` skill in *inline* mode.** Its procedure — glossary-resolve, enumerate-and-verify-premises, divergence-with-evidence (what · evidence · why-the-misunderstanding), spec-gaps, open-questions, readiness — lives in `~/.claude/skills/fix-issue/SKILL.md` (the generic, standalone-runnable version). The template below is its inline embodiment; keep the two in sync. The "surface, don't resolve" rule applies here too — answers/decisions happen at Stage 2 (planning) with the user, NOT in this stage.

**How:** spawn a `general-purpose` sub-agent via the `Agent` tool with the **Analyst Prompt Template** below, augmented with any overlay-specific additions (see "Overlay composition" near the end of this skill). The sub-agent's deliverable is `tmp/<slug>/01-analysis.md`.

After the sub-agent returns:
1. Read `tmp/<slug>/01-analysis.md` yourself.
2. Surface the **Open questions for the user** section verbatim to the user.
3. **STOP.** Wait for the user to either answer the questions or say "ok, plan it". Do not proceed automatically.

### Analyst Prompt Template (base)

Send as the `prompt` parameter to `Agent` (substitute `<slug>` and `<task-description>` and inject overlay additions where marked):

```
You are the Analyst stage of a development pipeline. Read these first:
- tmp/<slug>/00-prompt.md (the task)
- The current directory's CLAUDE.md if present (project rules)
- Any AGENTS.md, README.md in the affected directories
{OVERLAY: append project-specific read list}

Your single deliverable is tmp/<slug>/01-analysis.md. Do NOT modify any source
file in the repo. Do NOT write code. Read-only tools only (Read, Grep, Glob,
graphify if available, Bash for read-only commands like git log / diff / ls / grep).
For task-type=design, ALSO read Figma via the figma-analyze skill (deep dumpNode)
— see the ⟦DESIGN⟧ block below. Never spec from the lossy get_design_context.

Required sections in 01-analysis.md (use these exact headings):

## Task restatement
Two to four lines, your own words.

## Affected zones
List exact files/directories touched. {OVERLAY: append zone-classification rules}

## Current behavior
How does this work today? Include exact reproduction steps:
- Which page / URL / entry point?
- Which command to run?
- Which auth state / setup is needed?
- What does the user see right now?

**Ground this in an OBSERVED payload/state, not the issue's description.** For a
bug, reproduce it and inspect the ACTUAL data on the wire / the events emitted —
and **trace them to their producer** (the server / code path that builds the
response). The issue's described symptom, quoted payload, and "probable cause" are
premises that are often incomplete (a captured snapshot can miss a real-time
event) or symptom-level — read the WHOLE payload/chain, not the one field the
reporter highlighted, and confirm/refute the stated cause against what you
observe. A root-cause that rests on the issue's premise rather than an observed
payload is NOT verified. (Once: an issue quoted a response field as null and
concluded "empty reply"; the real response carried a payload the client had no
component to render — the captured log had missed a real-time event; a fix built
on that premise shipped wrong.)

## Target behavior
How should this work after the task is done? Be specific.

## ⟦DESIGN⟧ Figma reads (mandatory if task-type=design)
Do NOT spec from get_design_context — it is a LOSSY Tailwind preview (it drops
GLASS / COLOR_BURN / multi-stop gradients / scale-flip glow / hidden layers).
For every target node, run the **figma-analyze** skill (deep dumpNode via
use_figma) and capture the raw spec. get_design_context is allowed ONLY for a
first glance, never as the spec of record.

## ⟦DESIGN⟧ Figma map
Table mapping each in-scope code component to its Figma node:
| Code component | Figma nodeId | Confidence | Notes |

## ⟦DESIGN⟧ Scope decisions (A/B/C)
Classify each component:
- **A (modify)** — Figma exists + component exists → restyle.
- **B (create)** — Figma exists, no code → CLI add + stories. ⚠ check there is a
  backend emitter (send_*), else it is a dead stub.
- **C (document)** — code/stub exists, no Figma → screenshot + follow-up "needs design".

## ⟦DESIGN⟧ Existing-logic inventory (due-diligence)
For each new design-driven function, strictly check (grep + graphify + Code
Connect) whether a method / handler / contract already exists BEFORE concluding
"not supported". Never assume a method is absent without checking — if it exists,
wire it; if not, do it visually and document it.

## Blockers
Anything that prevents starting work: missing spec details, unknown contracts,
third-party dep gaps, env config holes, broken dependencies. List as
questions or as references to who owns the unblock.

**False-API-premise playbook.** When a ticket asserts an API/endpoint "already
exists" and verification shows it doesn't (the premise-accuracy case — see Stage
0 glossary-resolve), the default unblock is **mock behind a real-request-shaped
seam**: one client function whose *signature* matches the real call the ticket
describes, body returns a deterministic mock today. Real endpoint later = swap
that one body, hook + UI untouched. Flag the seam with a clear TODO. Don't block,
and don't mock sloppily inline — the seam is what makes "swap later" a one-liner.
(a prior retro: ticket claimed `get_token_price_graph` existed; it didn't → seam.)

## What could break
Adjacent code paths that might regress.
{OVERLAY: append project-specific contracts and integration points}

## Connected flows / overlays / states
For EACH component in scope (use graphify «кто рендерит / открывает X»):
- Who renders it, and in what context?
- What does it open / expand into (overlay, fullscreen panel, modal)?
- Does that connected flow have its own Figma node and states (empty / loading /
  error / calendar / filters / search / ...)?
Mark every connected flow **in-scope** or **out-of-scope (+reason)** — never drop
it silently. An out-of-scope flow that has its own Figma → candidate follow-up
ticket; record its nodeId. **Classify each out-of-scope reason as a hard "no" or
a temporary "not yet"** — a sibling that already has a Figma node or a stubbed
route is almost always "not yet". For "not yet", build the in-scope structure to
ACCOMMODATE it cheaply (a list that takes N rows; a row that takes an optional
href) instead of hardcoding the literal minimum — the "not yet" tends to land
within days and rewrites a too-literal build. (a prior retro: shipped one non-clickable
Holdings row; Perpetuals/Prediction — flagged out-of-scope but present in Figma —
became clickable sibling rows + routes days later, rewriting the section.)

## Spec gaps
Things the task description doesn't say but should:
- Error states / failure modes
- Empty / loading / disconnected states
- Mobile vs desktop differences
- i18n / text overflow
- Auth-required vs anonymous paths

## Suggested test cases
Manual verification steps + any automated test ideas.
{OVERLAY: note test ownership rules if any}

## Open questions for the user
Bullet list. Each question must be specific enough that the user can answer in
one sentence. No "what do you think?" — questions like "should the empty
state show a CTA or only icon+text?".

When 01-analysis.md is written, return a 5-line summary of what's in it.
Do not paste the file contents back — the orchestrator will read the file.
```

---

## Stage 1.5 — Analysis metrics (emit at the Stage 1 gate)

The front-half (Stage 0 framing + Stage 1 analysis + any ticket creation) is scored **on its own** — analysis / task-framing is often the whole deliverable and otherwise never reaches the Stage 6.7 emit. Fire this **when the user accepts the analysis** (moving to Stage 2) **or when the run stops at ticket-creation** — whichever comes first.

Write `tmp/<slug>/01-metrics.md`:

```markdown
## Analysis self-metrics
- baseline_check: clean | switched | stale-caught-late   ← from Setup step 8
- premise_corrections: <N>   ← factual/architectural premises the user corrected during Stage 0+1 (wrong branch, wrong term meaning, wrong component taxonomy, "X already exists / doesn't")
- analyst_reruns: <N>        ← Analyst re-spawns caused by wrong baseline / scope / context
- open_questions: <raised> raised / <blocking> truly-blocking
- blockers: <caught> caught-in-analysis / <missed> surfaced-later-by-user
```

Then append one line to `~/.claude/skills/pipeline/analysis-scores.jsonl` (schema in the appendix "Analysis metrics"); composite over the 4 analysis dims A1–A4.

**Two triggers → propose a concrete skill/overlay harden (don't let the lesson settle only in memory):**
1. **`premise_corrections ≥ 2`** → Stage 0/1 context was under-specified. Harden the glossary-resolve step, the overlay's zone/component taxonomy, or the Analyst read-list.
2. **`analyst_reruns ≥ 1`** → the analysis was invalidated and redone (almost always a baseline/scope miss). Harden Setup step 8 (baseline check) or the Stage 1 scope inputs.

When a trigger fires, tell the user WHICH stage to harden and the concrete edit — same close-the-loop discipline as Stage 6.7.

---

## Stage 2 — Planner (main agent + /think)

**Goal:** turn the analysis into a precise, code-less plan that the implementer can follow.

**Before starting:**
1. Invoke `/think` — load it via Skill tool with `skill: "think"`. The four Karpathy principles apply to the planning conversation.
2. Read `tmp/<slug>/01-analysis.md`.

**⟦DESIGN⟧ 2.0 Design pre-flight (task-type=design):**
Before writing the plan, run the **figma-audit** skill on each target frame and record a `## Design pre-flight` section in `02-plan.md`:
- **Code Connect map** — nodes already mapped to code components. Mapped nodes are NEVER re-implemented from scratch; extend the existing component.
- **CSS blockers** (figma-audit severity) — each blocker gets an explicit decision-gate: approximate in code (implement) OR fix the mockup (design ticket).

**⟦DESIGN⟧ Comparison wall:**
Create `tmp/<slug>/comparison.html` — one row per component, left = Figma `get_screenshot`, right = the real Storybook/Playwright screenshot. **Only screenshots — never hand-code a CSS copy of the component** (that is wasted work). Stage 3 updates the right column after each layer; the wall is the visual source of truth for the rest of the run.

**During the stage:**
- Discuss with the user iteratively. Ask the analysis's "Open questions" if not yet answered.
- Surface alternative approaches when they exist. Don't pick silently.
- Build the plan in your head, then write it to `tmp/<slug>/02-plan.md` (the durable trail).
- **Surface the plan to the user as an HTML report via the `html-report` skill** → `tmp/<slug>/PLAN.html`, and give the user its absolute `file://` path. The markdown is the trail; the HTML is what the user actually reviews (they find prose plans hard to read). Do the same for any open-questions / contentious decisions you put to the user.
- Iterate on both as the user edits/simplifies.

**`02-plan.md` required sections:**

```markdown
## Goal (verifiable)
One sentence. After implementation, we'll know success because: <criterion>.

## Approach
2–5 sentences describing the high-level approach. No code.

## Steps
1. [step] → verify: [check]
2. [step] → verify: [check]
3. ...

## Files to touch
Exact paths. Note "new" vs "edit". Files outside this list should not be
modified by the implementer (scope guard).

## ⟦DESIGN⟧ Design pre-flight
- Code Connect map: <node → component, "mapped" / "new">
- CSS blockers + decision: <blocker → approximate | fix-in-figma>
- comparison.html: created (figma ↔ render rows)

## Pseudocode / contract sketches
Optional but encouraged: small snippets showing the shape of new functions,
component signatures, API request/response shapes. Plain text or
language-tagged fences. No production code yet.

## Weak spots / risks
What we're worried about. Concurrency? Cache invalidation? Backward compat?

## Test cases
Bullet list:
- [case] — manual / automated / integration / unit
```

**Stop condition:** user explicitly says "ok, implement" or equivalent. Do not start coding until that signal.

---

## Stage 2.5 — Failing test first (TDD)

**Goal:** before writing the fix, land a test that **FAILS on the current (unfixed) code** and captures the bug/feature contract. The fix is "done" only when this test goes green — and you have proven **red→green**.

Applies to **bugfix**, **backend-contract**, and any feature with verifiable logic. Skip only for pure-design/visual tasks (the comparison wall covers those) or genuinely untestable glue — and say so explicitly when you skip.

**Procedure:**
1. **Find the repo's conformant test pattern — do NOT invent a bespoke rig.** Locate the runner + config + existing tests (`vitest`/`jest`/`pytest`; `*.test.ts` / `*_test.py`; a dedicated config like `vitest.unit.config.ts` or a `pyproject` test section). Match where tests live and how they run. If the repo has *no* harness at all, flag it to the user and fall back to the lightest conformant option rather than scaffolding a framework unasked. (e.g. pure logic in `lib/**` + colocated `*.test.ts`, run via a dedicated test config — the overlay names the exact runner/command for a known workspace.)
2. **Make the defect unit-testable.** If it lives in hard-to-test code (a React render-timing race, a 3rd-party widget, IO), **extract the decision/core into a pure function** and test that. The extraction is part of the fix and MUST preserve behavior (e.g. for a valtio/`useSnapshot` subscription, the field must still be read at render so the subscription survives the extraction).
3. **Write the failing test, run it, confirm it is RED for the right reason** (it reproduces the bug — not a typo/import error). Record the red output.
4. **Then** go to Stage 3 and implement until the test is **GREEN**.
5. **Prove red→green at the end of Stage 3:** temporarily revert the fix (or just the guard line) and re-run — the test MUST go RED again; restore and confirm GREEN. *A guard that doesn't fail without the fix is not a guard.* Record the red→green evidence in `03-implementation.md`.

The test ships in the **same diff/PR** as the fix. This is the operational form of `/think` principle 4 ("write a test that reproduces it, then make it pass").

**Why:** a regression test only earns trust if it actually catches the bug. (a prior retro: the user asked "did you check it fails without the guard?" → proving red→green is now mandatory.)

**Live end-to-end verification — drive the whole chain in Playwright (not just the unit):** for any runtime symptom (UI freeze, broken screen, wrong flow), a passing unit/component test is necessary but **not sufficient**. You must also drive the **real running app through the whole user flow in Playwright**: *before* the fix, reproduce the failure with your own eyes (screenshot + the telling signal); *after* the code is in, re-run the same flow and confirm the whole chain now works. **"Couldn't reproduce" ≠ "works"** — a flaky live path means fall back to the deterministic test, never declare success from a no-repro. Never write "verified" from reasoning ("the logic is airtight") — only from a RED→GREEN you observed. *How* to bring the app up and drive it is workspace-specific: the **overlay** defines the concrete harness. (A plausible fix once shipped wrong because the failing case was never reproduced — it was caught by simply looking at the running app.)

**Stop condition:** test is RED for the right reason; proceed to Stage 3.

---

## Stage 3 — Implementer (main agent + narration protocol)

**Goal:** execute the plan. Stay in scope. Stay loud.

**Before starting:**
- Re-read `tmp/<slug>/02-plan.md`.
- Note the "Files to touch" list — those are your scope.

**During the stage — follow the narration protocol:**

```
- Перед каждым нетривиальным tool call — одна строка "→ <что делаю>".
- После любой ошибки/фейла — одна строка "✗ <что не получилось>".
- Не реже чем каждые 5 tool call'ов — короткая строка о том где сейчас.
- Молча можно: тривиальные Read одного файла, Grep, быстрые проверки <2с.
```

The purpose of narration is for the user to interrupt (Esc / Ctrl+C) before you go 20 minutes into the wrong direction. One-line prose before tool calls; not paragraphs.

**Per-step rhythm:**
- Take one step from the plan.
- Implement it.
- Verify it (the plan's "verify" check for that step).
- Move to the next.

**Local-verification discipline (env-readiness — before declaring any check blocked or failed):**
- **Kill stale dev servers first.** A running `next dev` / `storybook` caches the OLD module graph in memory and will keep reporting a fixed problem as broken — `pkill -f "next dev"` / `pkill -f storybook`, boot fresh, *then* read the result. (a prior retro #1 time-sink: a stale `next dev` made a working `bun install` look like it "didn't fix it".)
- **Diagnose with evidence, not assertion.** State a root-cause as a *hypothesis* until proven — `ls`/inspect the actual installed version, reproduce on a fresh process. Never tell the user "X caused it" (a dep version, "a relink happened") without a check that confirms it. (a prior retro: asserted a wrong `ox` version + an unverified "relink" — both retracted.)
- **A blocked verification is a problem to fix, not a reason to skip.** If the environment blocks a check (especially the ⟦DESIGN⟧ visual-acceptance gate below), fixing the env is in-zone — auto-do it (per the 4-condition autonomy rule) instead of downgrading to "can't verify / optional".

**⟦DESIGN⟧ Per-layer rhythm (default implementer = figma-implement skill):**
**Before coding a layer, resolve every LEAF visual in it from Figma — do NOT approximate.** Enumerate each icon / button stroke / chip-or-pill chrome / divider and pull its exact source: icons → `get_screenshot` / export as an asset (never substitute a lucide glyph for a custom Figma icon); bubbles / chips / borders → a per-node `get_design_context` dump of the stroke + fill, NOT the defaults of whatever component you're reusing. Reusing a component's *layout* is fine; inheriting its *chrome* where Figma differs is a deferral that resurfaces as a user-caught tweak round (a prior retro spent 3 — a coin icon faked with lucide `Wallet`, a missing button glass border, an unwanted address pill — all caught on renders, none by self-verify).
For each Figma layer:
1. Implement with numbers verbatim — `style={{height:50}}` for px, `rem` for fonts. **Never** Tailwind arbitrary like `h-[50px]` (the JIT may silently fall back).
2. Verify via Playwright `getBoundingClientRect` that the class actually applied (not a fallback size).
3. Update the right column of `comparison.html` (Figma `get_screenshot` ↔ Playwright render).
4. **Self-diff before declaring the layer done:** look at the Figma ↔ render pair and state the deltas OUT LOUD per element (icon matches? border/stroke? chip chrome? type weight/color?). "Looks close" is not done — name what differs or confirm per-element parity. This is the gate the user otherwise becomes.
5. 1 layer = 1 commit.

**Logging (compact-safe):** maintain `tmp/<slug>/03-implementation.md` **incrementally** — append a line under `## Done` after EACH step/layer, not once at stage end. If a `/compact` happens mid-stage, this file is the recovery point. Append only; never overwrite prior lines (same discipline as Stage 5 "Round N fixes"). Final shape:

```markdown
## Done
- [step from plan] → [what was changed: file:line ranges]
- ...

## Skipped
- [step] → [why it was skipped or deferred]

## Deviations from the plan
- [what changed in approach and why]

## How to verify locally
Exact commands the reviewer can run.

## Commits / branch
- Branch: <name>
- Commits: <SHAs> (if any)
```

Then tell the user: "implementation done, ready for review". **STOP.** Wait for "ok, review".

---

## Stage 4 — Reviewer (sub-agent)

**Goal:** fresh-eyes check of the implementation against the plan. The reviewer must not be the implementer — that's the whole point of isolated context.

**How:** spawn an `Explore` sub-agent (read-only) with the **Reviewer Prompt Template** below, augmented with overlay additions. Deliverable: `tmp/<slug>/04-review-<N>.md` where N is the next free integer (starts at 1).

After the sub-agent returns:
1. Read the review file.
2. Summarize the verdict (CLEAN or FIXES NEEDED) and the priority list of fixes to the user.
3. **STOP.** Let the user decide what to do next.

### Reviewer Prompt Template (base)

```
You are the Reviewer stage of a development pipeline. You have not seen the
implementation before — that's intentional. Your job is fresh-eyes audit.

Read in order:
1. tmp/<slug>/00-prompt.md (original task)
2. tmp/<slug>/02-plan.md (agreed plan)
3. tmp/<slug>/03-implementation.md (what was supposedly done)
4. The actual diff: `git -C <project-root> diff <base>..HEAD` for each touched
   project (find them from 03-implementation.md "Files to touch").
5. ~/.claude/skills/conformance-review/SKILL.md (conformance methodology for
   the section below).
{OVERLAY: append project-specific read list}

Your single deliverable is tmp/<slug>/04-review-<N>.md. Read-only tools only.
Do NOT modify any source file.

Required sections (use these exact headings):

## Plan adherence
For each item in 02-plan.md "Steps", one line:
- ✓ done as planned
- ⚠ partial / different (explain why and whether it matters)
- ✗ missing

## Out-of-scope changes
Any diff hunks that don't trace back to a plan item. Quote file:line.
Out-of-scope changes need either justification or revert.

## Bugs / functional issues
Real breakage you can see by reading the code. Quote file:line. Be specific:
"Line 47: handler is bound on mount but never removed on unmount — leak."
**Verify before asserting a byte/character-level claim** (ASCII vs NBSP, CRLF vs
LF, trailing whitespace, smart vs straight quotes) — run `hexdump -C` / `cat -A`,
don't eyeball it. A confidently-wrong nit (a prior retro: "format.ts inserts an ASCII
space" — it was already U+00A0) wastes a fix cycle and erodes trust in the review.

## Conformance (дифф против конвенций кодбейзы)
Apply the conformance-review skill (item 5 of the read list): does the diff do
things the way THIS repo does them? Hunt specifically for: self-built helpers /
mappings / error-shapes / caches where an internal implementation exists (grep
name-analogues: `*_TO_*`, `*_error`, `unsupported_*`, cache mechanisms); raw
`str` params where neighbors use Literal/typed aliases; new env vars missing
from `.env.example`; silent resolves where the repo has an explicit lookup
pattern; AND the opposite trap — new wrapper layers/modules added "for purity"
where direct reuse was enough (over-wrap). Every non-ok finding MUST carry
codebaseEvidence: real file:line from THIS repo showing "how it's done here";
general best practices are not an argument. Severity: must-fix / should-fix /
nit / ok-as-is; if the baseline itself violates the convention → nit max.
On review rounds N≥2, re-check only hunks changed since the previous round.

## Test gaps
Were the test cases from 02-plan.md actually covered?

## Risk flags
Production-relevant concerns: deploy gates, observability, bundle size,
auth flows, data integrity.
{OVERLAY: append project-specific risk patterns}

{OVERLAY: insert additional required sections for project contracts}

## Verdict
One of:
- **CLEAN** — implementation matches plan, no issues. Ready to merge.
- **FIXES NEEDED** — list in priority order (P0 blocks merge, P1 should-fix,
  P2 nice-to-have).

When the review file is written, return a 5-line verdict summary.
Do not paste the file contents — the orchestrator will read the file.
```

---

## Stage 5 — Fix loop

**Goal:** iterate until the reviewer's verdict is CLEAN.

**Loop body:**
1. **Decide which review notes to address.** Load `/think` (Skill tool with `skill: "think"`). The four principles help push back on bad-faith review notes and prevent fixing things that shouldn't be fixed. Not every reviewer note is correct. Discuss disagreements with the user.
2. **Fix the agreed-upon items** following the Stage 3 narration protocol.
3. **Update `tmp/<slug>/03-implementation.md`** with the new diffs (append a "Round N fixes" section, don't overwrite).
4. **Spawn a fresh Reviewer sub-agent** with the same composed prompt (base + overlay additions). The new reviewer writes `tmp/<slug>/04-review-<N+1>.md` and starts from a clean slate. If you want the new reviewer to see prior rounds, add the prior review files to its "Read in order" list.
5. If verdict CLEAN → exit to Stage 6. Otherwise → repeat.

**Loop guard:** if you reach review round 4 and still aren't CLEAN, stop and discuss with the user — that's a signal the plan was wrong or the task is bigger than expected, not that you need another fix pass.

---

## Stage 6 — Final summary

**Goal:** wrap up. Write summary + changelog. Final diff review. Optionally open PR.

### 6.1 — Write `tmp/<slug>/99-summary.md`

```markdown
## What was done
2-4 line summary.

## Files changed
Summary list (read from the final 03-implementation.md).

## Test verification
How it was verified (locally / automated / manual flow).

## Followups / known issues
Bullet list of things deferred to other tickets, with IDs if created. Mention
if any backlog-file items were closed.

## Round count
Stages 4-5 ran N times before CLEAN.
```

### 6.2 — Changelog entry

If the project has a `changelog/` directory (or `CHANGELOG.md`), append an entry there per its existing convention. If the overlay specifies a template (e.g. `_TEMPLATE.md` in changelog/), follow it.

If **no** changelog exists and the user wants one, ask once: «создать `changelog/` в корне проекта (или `CHANGELOG.md`)?». If yes, create with a one-line intro and the new entry. If no, skip — the summary in `tmp/<slug>/99-summary.md` is the audit record.

### 6.3 — Final diff review (`/review` pass)

Before opening any PR, invoke `/review` against the current diff. This is a second pair of eyes after the Stage 4 sub-agent.

- If `/review` accepts a diff target → run it on `git diff <base>..HEAD`.
- If `/review` requires an actual PR target → either create a `gh pr create --draft` first and run `/review` on it, OR fall back to spawning one more `Explore` sub-agent with the composed Reviewer Prompt Template (base + overlay). Either way, the goal is one more independent pass.

If the final-review finds P0/P1 issues, treat it as one more fix-loop round (go back to Stage 5). Otherwise continue.

**⟦DESIGN⟧ Final visual acceptance:** in addition to `/review` on the diff, open the FINAL `tmp/<slug>/comparison.html` (after all fixes) and confirm per-component Figma match as the ship artifact. Attach it (or its screenshots) to the Stage 6.4 ticket comment — it doubles as visual proof-of-done.

### 6.4 — Ticket comment

If a ticket is linked, leave a summary comment on it (Linear MCP `save_comment`, `gh issue comment`, or whatever the overlay specifies). Do not change status manually unless the overlay tells you to — most ticket systems with git integration auto-move on PR events.

**Visual evidence → inline images, not files-to-download.** When attaching design proof (comparison / EVIDENCE / plan reports, render states), deliver it **inline in the comment**, not as files the reviewer must download and open. Render each report HTML to a full-page PNG, then embed it inline. Rendering recipe (Playwright MCP blocks `file://`, so a transient local server is needed *for rendering only* — this is internal, not an artifact-delivery localhost): `python3 -m http.server <port>` in the report dir → Playwright `browser_navigate(http://localhost:<port>/report.html)` → `browser_take_screenshot({fullPage:true})` → kill the server. Upload the PNG (overlay upload helper) and embed `![](assetUrl)` in the comment body; the ticket system signs the URL on save = confirmation it renders inline. Keep the source HTML attached too, for interactivity. Chat summary stays terse (thesis + "details inline").

### 6.5 — PR creation (optional)

**PR pre-flight — MANDATORY before `gh pr create`, in this order:**
1. **Rebase freshness:** `git fetch origin <base>`, then
   `git rev-list --count HEAD..origin/<base>`. If > 0 — the branch is stale:
   rebase onto fresh base FIRST (resolve conflicts, re-run the verifies of any
   step a conflict touched). Never open a PR from a branch behind its base.
2. **Build check (post-rebase, not before):** run the project's standard
   build/typecheck per the overlay or repo convention (e.g. `tsc --noEmit`,
   `ruff check`, `task lint`). Zero NEW errors required. Pre-existing baseline
   noise (verify via the same check on the base commit / `git stash`) — note
   it in the PR body if relevant, don't fix others' lines.
3. If either check was already done this session but the base moved since (new
   merges upstream) — redo both; "it passed an hour ago" is not current.

Then, if the user wants a PR:
- Use the overlay-specified shortcut if available (e.g. `linear pr` for the workspace).
- Otherwise `gh pr create` with title + body referencing the ticket / changelog entry.
- Promote the draft from 6.3 if you opened one for the diff review.

### 6.6 — Cleanup

Do **not** delete `tmp/<slug>/` — it's the audit trail. The user cleans `tmp/` themselves.

### 6.7 — Metrics emit

Write `tmp/<slug>/98-metrics.md` with the run's self-metrics, then materialize the auto/self-report fields into `tmp/<slug>/metrics.json` (schema in the appendix "Pipeline self-metrics") and append one line to `~/.claude/skills/pipeline/scores.jsonl`.

```markdown
## Pipeline self-metrics
- Ad-hoc files beyond canon: <N>             ← ≥2 = a sub-stage is under-specified
- Manual user redirects / corrections: <N>   ← ≥3 = harden the overlay/skill
- Tool retries (tsc / hydration / upload / ...): <N>
- Review rounds: <N>
- Scope-miss caught by: process | user       ← "user" = a hole in Stage 1/4
- Skills that needed a reminder: <list>
```

The post-run-scored dimensions (D1–D4, D6) are filled by a separate fresh-context scorer agent reading this trail; the auto fields (D5/D7/D8) come straight from `ls`/the log.

**Trigger:** if ad-hoc files ≥ 2 OR manual redirects ≥ 3, tell the user WHICH stage to harden and propose the concrete overlay/SKILL edit — don't let the lesson settle only in memory. This is what closes the learning loop.

---

## Stage transitions & user checkpoints

After every stage, **STOP** and surface the artifact to the user. Do not chain stages automatically. The transition gates:

| After stage | Wait for user signal like… |
|---|---|
| 1 (Analyst) | "ok, plan it" / "перейдём к плану" / answer to open questions |
| 2 (Planner) | "ok, implement" / "погнали кодить" |
| 2.5 (Failing test) | flows into 3 — no separate gate; the RED test precedes any fix code |
| 3 (Implementer) | "ok, review" / "ревью" |
| 4 (Reviewer) | "fix these" / "ignore that one, fix the rest" / "ship it" |
| 5 (Fix loop) | "another review" / "looks clean, ship it" |
| 6.1–6.4 (Final) | done — confirm artifacts written |
| 6.3 (Final review) | act on /review verdict — clean continues, fixes loop back to 5 |
| 6.5 (PR) | only if user asked to open a PR |

These checkpoints are the **only** explicit stops. There are no per-action checkpoints inside a stage. The narration protocol from Stage 3 is for visibility, not for stopping.

**Post-pushback discipline (outward actions).** Normal in-zone work is auto-do (the 4-condition rule — no per-action asking). But a user **pushback or correction resets that consent for the next batch of *outward* actions** — anything that writes to an external system or is hard to reverse (uploads, ticket comments, commits, pushes, PRs). After the user questions or corrects an approach, **confirm the approach in one line before firing a batch** of those, rather than treating the earlier go-ahead as still valid. This is the only exception to "don't re-ask inside a stage" — it prevents the over-ask-then-over-act swing. (a prior retro: right after a "куда ты это клеишь?" pushback I launched 7 Linear uploads in parallel; 2 were auto-denied because charging ahead post-doubt reads as ignoring it.)

**Surface checkpoints as HTML, not markdown walls.** When a gate needs the user to read something substantial (the Stage 2 plan, a set of open questions / contentious decisions, design EVIDENCE, the Stage 6 final summary), render it via the `html-report` skill, **open it in external Chrome** (`open -a "Google Chrome" <abs-path>`), and print the absolute `file://` path too. The numbered markdown files stay the durable trail; the HTML is the human-facing view. The cmux built-in browser shows HTML as plaintext (and hangs on link-select), so don't rely on it. Never give a relative path in chat (cmux mangles `tmp/...` → `https://tmp/...` → blank) and never spin up a localhost server.

---

## File layout

```
<tmp-root>/<slug>/
├── 00-prompt.md            # original task (incl. ## Task type)
├── 01-analysis.md          # Analyst sub-agent output (+ ⟦DESIGN⟧ sections)
├── 02-plan.md              # Planner main-agent output (+ ## Design pre-flight)
├── comparison.html         # ⟦DESIGN⟧ Figma ↔ render side-by-side wall
├── 03-implementation.md    # Implementer log (compact-safe, incremental)
├── 04-review-1.md          # first Reviewer pass
├── 04-review-2.md          # second pass (after fixes)
├── 04-review-N.md          # ...
├── 98-metrics.md           # Stage 6.7 self-metrics
├── metrics.json            # machine metrics (feeds scores.jsonl)
└── 99-summary.md           # final
```

The numeric prefix ensures `ls` shows stages in order.

`<tmp-root>` defaults to `<repo-root>/tmp/`. Overlays may override (e.g. the workspace uses `<workspace-root>/tmp/`).

---

## When NOT to use /pipeline

- Trivial fixes: typo, one-liner, obvious rename, dep version bump.
- Exploratory probe: "посмотри как сделана X" — that's a `/graphify` or `Explore` job.
- Already mid-task: don't start a pipeline halfway through implementation.
- User wants speed over rigor and the task is small.

For everything else where the task touches more than one file or has any architectural component, `/pipeline` saves the rewrite.

---

## Overlay composition

Workspace overlays live at `~/.claude/skills/pipeline/contexts/*.md`. Each overlay declares:
- **Detection signature** — a check (typically a `pwd` ancestor + a file existence + a content marker) that determines whether the overlay applies.
- **Project structure** — own/forbidden zones, exceptions.
- **Conventions** — file layouts (CLAUDE.md, AGENTS.md, backlog, changelog), branch naming, etc.
- **Stage overlays** — additions to inject into the Analyst and Reviewer sub-agent prompts.
- **People** — owners of specific zones, e2e, etc.
- **Ticket system specifics** — IDs format, projects, assignees, status integration.

**How to apply overlays at runtime:**

1. At Stage 0 Setup step 1, run each overlay's detection signature against `pwd`. If one matches, read its full content with the Read tool. Keep it referenced throughout the run.

2. **For sub-agent prompts (Stage 1 Analyst, Stage 4 Reviewer):** before sending the `prompt` to `Agent`, replace each `{OVERLAY: ...}` marker in the base template with the corresponding section content from the overlay. If no overlay is loaded, remove the markers (they become no-ops).

3. **For interactive stages (Stage 0, 2, 5, 6):** consult the overlay directly when ticket creation, backlog placement, changelog format, or contract-specific advice is needed.

4. If no overlay is loaded:
   - Ticket system: ask the user once at Stage 0 if any (Linear, GitHub Issues, none).
   - Backlog file: detect `BACKLOG.md` / `BACKLOG.local.md` / `TODO.md`; if absent and user wants one, ask before creating.
   - Changelog: detect `changelog/` or `CHANGELOG.md`; if absent and user wants one, ask before creating.
   - Branch naming: respect existing repo convention or ask.

---

### Onboarding a workspace — what it is, and a copy-paste template

A **workspace** is a recurring project context the user works in — a repo (or a multi-repo root) with conventions the core can't guess: which issue tracker + team/project, which directories are editable vs off-limits, the changelog/branch conventions, the build/test/PR commands, the live verification harness, and who owns what. The core is workspace-agnostic; a **workspace overlay** (`contexts/<name>.md`) carries all of that so every run there "just works".

**When to onboard one:** if the user runs the pipeline in the same project repeatedly and no overlay matches their `pwd` at Stage 0, offer to create an overlay (a one-time setup). Use the same template to *extend/fix* an existing overlay.

**How to fill it — discover first, then ask only the human-only bits:**
- **Discover from the repo:** the `.git` remote + default branch; `package.json` / `pyproject` scripts (build / typecheck / test commands); an `AGENTS.md` / `CONTRIBUTING.md` / root project doc (conventions, zones); a `changelog/` dir + its template; existing branch names (`git branch -a`); a test config (`vitest*.config.*`, a `pytest` section) for the deterministic-test harness.
- **Ask the user only:** the issue tracker + team/project + default assignee; which dirs are OFF-LIMITS (others' zones) and who owns them; the deploy/CI gate (what must be green to merge); whether a **live verification harness** exists (how to bring the running app up and drive it for red→green) or note "none yet".
- **Validate before trusting it:** test that the Detection signature actually matches the user's `pwd`; run the build/test commands once; if you reference a verification harness, confirm it exists. An overlay that doesn't auto-detect, or whose commands are wrong, is worse than none.

**Generic overlay template** — copy to `~/.claude/skills/pipeline/contexts/<name>.md`, fill every `<…>`, delete sections that don't apply:

````markdown
# <Workspace name> overlay for /pipeline

Loaded by /pipeline when `pwd` is inside this workspace. If detection fails, this file is ignored.

## Detection signature
All true ⇒ in this workspace:
- `pwd` has an ancestor dir named `<root-dir>`, AND
- that dir contains `<marker file, e.g. CLAUDE.md / package.json>`, AND
- that file mentions `<unique content marker, e.g. an org/product name>`.
(Specific enough that no OTHER workspace matches.)

## Project structure
- **Active zones** (edit freely): `<repo-a/>`, `<repo-b/>` — one line each on what it is.
- **Forbidden zones** (read-only; file issues to owners): `<vendor/>`, `<owned-by-X/>` — and who owns each.
- **Exceptions**: any sub-path of a forbidden zone that IS editable.

## Conventions
- Per-project files: `AGENTS.md` (rules), `BACKLOG.local.md`, `changelog/` (`<format / _TEMPLATE.md>`).
- Temp artifacts root: `<where tmp/ lives, e.g. <root>/tmp/>`.

## Stage 0 — tracker & branches
- **Issue tracker:** `<Linear MCP / GitHub Issues (gh) / Jira / none>` — team(s) `<…>`, project(s) `<by zone>`, default assignee `<user>`. Note any TRAPS (e.g. a same-named project bound to the wrong team).
- **Branch naming:** `<convention, e.g. <issue-id-lower>/<2-word-desc>>` + any tracker↔git status automation (push→In Progress, PR→In Review, merge→Done?).

## Stage 1 (Analyst) — inject into the sub-agent prompt
- Read-list additions: `<root project doc, affected AGENTS.md, changelog tail, NOTES.md>`.
- Zone classification rule (own vs forbidden).
- Project-specific "what could break" contracts (framework/API contracts a change can violate) + any component-taxonomy gotchas.

## Live red→green verification harness
- How to bring the running app up and drive it for red→green (a script / a dedicated skill / `<commands>`), and the deterministic fallback test (config + run command). If none exists yet, say so — that's a known gap, not a silent omission.

## Stage 4 (Reviewer) — inject into the sub-agent prompt
- Workspace contract-violation checks (forbidden-zone touches, secret in client bundle, banned endpoint patterns, e2e-selector changes).
- Risk flags (the deploy/CI gate).

## Stage 6 (Final)
- Changelog placement + format.
- PR pre-flight: base branch per repo + the exact build/typecheck/lint command per repo.
- Ticket-comment tool; inline-image upload helper if the tracker needs one.

## People
| who | role / zone | contact |
````

The minimum for it to "work right away": a Detection signature that matches, the tracker (Stage 0), the zones (Stage 1), and the build/test/PR commands (Stage 6). The rest can be filled incrementally as the workspace teaches you (every manual redirect ≥3 or ad-hoc deviation is a cue to harden the overlay — see Stage 6.7).

---

## Integration with other skills

- **`/think`** — explicitly loaded at Stage 2 (planning) and Stage 5 (fix-loop triage). The four Karpathy principles drive both the plan and the pushback against bad reviewer notes.
- **`/review`** — explicitly invoked at Stage 6.3 for the final pre-PR diff pass. Second pair of eyes after the Stage 4 sub-agent.
- **`conformance-review`** — its methodology is embedded as the Stage 4 reviewer's `## Conformance` section (diff vs the surrounding codebase's live conventions: reuse-internal-first, no over-wrap, codebaseEvidence per finding). Also standalone-invocable (`/conformance-review`) when work happened outside the pipeline — run it before the PR for any unfamiliar repo/zone.
- **`/graphify`** — Analyst sub-agent can use it to traverse the code graph if `graphify-out/` exists in the project. For design tasks it also answers "who renders / opens X" for the Connected-flows section.
- **`html-report`** — renders user-facing checkpoint artifacts as scannable dark-themed HTML opened in **external Chrome** (`open -a "Google Chrome" <abs>`; the cmux built-in browser shows HTML as plaintext + hangs on link-select), with the absolute `file://` path printed as fallback: Stage 2 plan → `PLAN.html`, open questions / contentious decisions, design `EVIDENCE.html`, Stage 6 final → `FINAL.html`. Block `<style>` is fine; never localhost; never a relative path in chat. The markdown artifacts remain the durable trail; the HTML is what the user actually reviews.
- **Figma skills (for `task-type=design`)** — wired into the design stages, in order:
  - **`figma-analyze`** → Stage 1 (Analyst) and again inside `figma-implement`. Deep `dumpNode` raw spec; replaces lossy `get_design_context`.
  - **`figma-audit`** → Stage 2 pre-flight. CSS-blocker severity report + Code Connect map BEFORE writing code.
  - **`figma-implement`** → Stage 3 default implementer for visual components (layer-by-layer + side-by-side verify).
  - **`figma-refactor`** → conditional escape hatch (not enabled in this build) for un-implementable "artistic" mockups.
- **`/audit`, `/critique`, `/polish`** — UX/quality skills can be invoked inside Stage 3 if the task is UI-heavy.
- **`/ultrareview`** — alternative cloud-based multi-agent review on a PR. User-triggered only; can replace Stage 4 if the user prefers cloud review over an in-session sub-agent.

---

## Linear CLI integration (optional accelerator)

For Linear-based workspaces only. The pipeline runs fully without `linear-cli` — Linear MCP / `gh` / `git` cover all operations. If `schpet/linear-cli` is installed (`brew install schpet/tap/linear` or per the [repo](https://github.com/schpet/linear-cli)), three commands collapse multi-step flows into one-liners:

| Command | Replaces | Where it helps |
|---|---|---|
| `linear start <id>` | ticket status update + `git checkout -b <branch>` | Beginning of Stage 3 when task is Linear-linked. Creates branch from issue title and moves status to In Progress. Configure branch template via `.linear.toml` to match project convention (see workspace overlay for the template). |
| `linear pr` | `gh pr create` with manual title/body | Stage 6.5 when opening a PR. Auto-fills PR title/body from the linked issue. |
| `linear id` | guessing issue ID from `git branch --show-current` | Any stage, quick "which issue am I on" check. |

**Detection:** `which linear` at Stage 0. If absent, proceed with MCP / `gh` / `git`. Do not block on install or prompt the user — that choice is theirs.

---

## Versioning & self-metrics (appendix)

**Version:** v2 (2026-06-01) — added the design/Figma track (task-type classification; figma-analyze / figma-audit / figma-implement routing; Analyst sections Figma-map + A/B/C scope + Connected-flows + existing-logic inventory; the `comparison.html` wall), the stage progress-tree, compact-safe incremental logging, final visual acceptance, the Stage 6.7 metrics emit, and the the workspace Linear-upload helper.

**v2.1 (2026-06-01)** — Setup step 8 **baseline check** (git freshness before analysis; prevents the wasted-Analyst-round-on-stale-branch failure) and Stage 0 **glossary resolve** (confirm transcribed/voice domain terms absent from code before Stage 1). The workspace overlay: Sparkling component taxonomy (attachment=chat vs screen-level) in Stage 1, and expanded §Ticket quality (no-redundancy / attach-not-describe / no-meta / verify-absence / «не делаем» only-for-traps).

**v2.2 (2026-06-01)** — **Stage 1.5 Analysis metrics**: a front-half-only score (`analysis-scores.jsonl`, dims A1–A4) so analysis/task-framing runs that stop at ticket-creation are tracked even without reaching Stage 6.7, plus two harden-triggers (`premise_corrections ≥ 2`, `analyst_reruns ≥ 1`).

**v2.3 (2026-06-06)** — design-fidelity hardening from a prior retro (composite 0.78; all regret traced to one root — leaf-element approximation): Stage 3 ⟦DESIGN⟧ per-layer rhythm now mandates **per-leaf extraction before coding** (icons → asset export, strokes/fills → per-node dump; no lucide/reused-component approximation) + an explicit **Figma↔render self-diff gate** before a layer is "done"; Analyst **Connected-flows** now splits out-of-scope into hard-"no" vs "not-yet" and builds extensible for "not-yet"; Reviewer must **hexdump byte-level claims**. The workspace overlay design block gained the per-leaf-fidelity rule.

**v2.4 (2026-06-06)** — **`html-report` skill** extracted and wired in: user-facing checkpoint artifacts (Stage 2 plan, open questions / contentious decisions, design EVIDENCE, Stage 6 final) are surfaced as scannable dark-themed HTML (**opened in external Chrome** via `open -a "Google Chrome"` — the cmux built-in browser shows HTML as **plaintext**; absolute `file://` path printed as fallback; block `<style>`) in addition to the markdown trail. Encodes the artifact-delivery rules learned in a prior retro: never localhost for delivery; never a relative path in chat (cmux mangles `tmp/...` → `https://tmp/...` → blank page); block CSS renders fine in Chrome (the earlier "inline-styles fix" was a misdiagnosis).

**v2.5 (2026-06-06)** — process hardening from a prior retro (composite 0.75; regret root D3 autonomy + D8 cost — env-blocker handling + outward-action discipline, not craft), via the **`pipeline-audit`** skill: Stage 3 **local-verification discipline** (kill stale dev/storybook servers before diagnosing; evidence-backed diagnosis — no asserted root-cause; a blocked check is to fix, not skip); Stage transitions **post-pushback confirm-before-batch** for outward actions; Stage 6.4 **visual evidence inline** (render report HTML → full-page PNG → `![](assetUrl)`, not files-to-download); Analyst **false-API-premise → mock-behind-real-shaped-seam** playbook; fixed the v2.4 appendix cmux drift; the workspace overlay gained the Linear inline-render helper + the `PieCard data.name` reserved-prop gotcha.

**v2.6 (2026-06-12)** — from the a prior retro review lessons: Stage 4 Reviewer gained the **`## Conformance` section** (conformance-review skill methodology: diff vs the repo's live conventions — reuse-internal-first / no over-wrap / Literal-aliases / `.env.example` / codebaseEvidence per finding, baseline-violates → nit max; rounds N≥2 re-check only new hunks) + the skill added to the reviewer read-list; Stage 6.5 gained the **mandatory PR pre-flight** (rebase-freshness via `rev-list --count HEAD..origin/<base>` + post-rebase build/typecheck with zero NEW errors; redo both if the base moved since the last check). The workspace overlay: per-repo pre-flight commands + base branches.

**v2.7 (2026-06-15)** — from a prior retro: new **Stage 2.5 — Failing test first (TDD)** between Plan and Implement — write a test that is RED on the unfixed code, extract a pure testable core for hard-to-test defects (React/valtio race, 3rd-party widget, IO), match the repo's conformant test harness (no bespoke rig), and **prove red→green** by reverting the guard at the end of Stage 3 (a guard that doesn't fail without the fix is not a guard). Test ships in the same diff/PR. Also: **one-doc-per-task** convention — a single consolidated doc per task (cause/fix/decisions/verification) for shared/ticket use, neutral politesse tone (never "the ticket is wrong"), no process chatter / owner-assignment inside the doc (that lives in chat); the html-report skill carries the rendering convention.

**v2.8 (2026-06-17)** — from a prior retro: **live red→green gate**. Stage 2.5 extended — for a runtime/UI symptom the unit/component test is the durable proof but not the whole proof: reproduce the failure in the real app *with your own eyes* before the fix and re-confirm after, via the workspace overlay's live/UI harness when defined (flaky live env → fall back to a deterministic component test). Hard gate: never write "works/verified" in a facing doc without an observed RED→GREEN — **"couldn't reproduce" ≠ "works"**, reasoning is not verification. The workspace overlay carries the concrete harness wiring.

**v2.9 (2026-06-17)** — from a prior retro (analysis miss): Stage 1 Analyst **"Current behavior" must be grounded in an OBSERVED payload/state, not the issue's description** — a bug's described symptom / quoted payload / "probable cause" are premises, often incomplete (a captured snapshot can miss a real-time event) or symptom-level; reproduce the failure, read the WHOLE real payload/events and **trace them to their producer** (the code that builds the response), confirm/refute the stated cause against what you observe before designing a fix. Mirrored in the `fix-issue` skill (step 3).

**v2.10 (2026-06-17)** — core kept workspace-agnostic: recent additions reworded to **base terms** (project / repo / issue / response / client) with workspace-specifics (ticket IDs, framework field/component names) pushed to the overlay. Added **"Onboarding a workspace"** to Overlay composition — what a workspace is, a discover-then-ask procedure, and a **copy-paste generic overlay template** (Detection signature · zones · conventions · tracker/branches · Analyst/Reviewer injects · live red→green harness · Stage 6 build-PR · people) so an agent can add a user's existing workspace or create a new one that works on first run.

Bump this line on every change; `skillVersion` in `metrics.json` = `v<N>-<date>`.

**`metrics.json` schema** (one per run, written at Stage 6.7; aggregated into `~/.claude/skills/pipeline/scores.jsonl`):
- `skillVersion`, `slug`, `date`, `taskType`.
- `taskSizeProxy`: `{ scopeUnits, files, loc, steps, TSU }`, where `TSU = (scopeUnits·files·(loc/50)·steps)^¼` — the size normalizer (~12 for a big task, ~1 for a one-liner).
- `dimensions`: 8 axes, each `{ raw, score∈[0..1], source }`, all higher-better:
  D1 Scope Fidelity (w .20, post-run) · D2 Skill Routing (.18, post-run) · D3 Autonomy (.18, post-run) · D4 Review Effectiveness (.14, post-run) · D5 Artifact Completeness (.10, auto) · D6 Rework Intensity (.08, post-run) · D7 Process Hygiene (.07, auto) · D8 Cost Efficiency (.05, auto).
- `composite = (Σ wᵢ·Dᵢ) / (Σ wᵢ live)` ; `regret = 1 − composite`.

**Analysis metrics** (`~/.claude/skills/pipeline/analysis-scores.jsonl`, one line per front-half run, written at **Stage 1.5** — tracks analysis quality even for runs that stop at ticket-creation and never reach Stage 6.7):
- `skillVersion`, `slug`, `date`, `taskType`.
- `analysis`: 4 dims [0..1], higher-better — **A1 Baseline** (clean=1 / switched=.7 / stale-caught-late=0; auto) · **A2 Premise accuracy** (= 1 − min(1, `premise_corrections`/4); self) · **A3 Blocker discovery** (= caught/(caught+missed), NA if 0; self) · **A4 Artifact completeness** (01-analysis required sections present; auto). `composite = mean(live dims)`, `regret = 1 − composite`.
- `triggers`: `{ premise_corrections, analyst_reruns }` — the two harden-triggers (T1 `premise_corrections ≥ 2`, T2 `analyst_reruns ≥ 1`).

**Version comparison:** within a `{taskType × TSU-tercile}` bucket, `K = median(regret_old) / median(regret_new)` (the "N× better" figure), claimed only on non-overlapping IQR, always with the top-3 `wᵢ·ΔDᵢ` "because" breakdown. `auto` dimensions are deterministic from the trail; `post-run` ones are scored by a separate fresh-context agent (no self-grading). Full design + worked v1 baseline: `the workspace/tmp/a prior retro/PIPELINE-V2-PROPOSAL.md`.
