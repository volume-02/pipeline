# pipeline

A development pipeline skill for Claude Code: takes a non-trivial task end-to-end through staged hand-offs — optional Stage 0 (task framing from a ticket / backlog entry / fresh description) → Analyst (sub-agent) → Plan → Implement (with narration) → Review (sub-agent) → fix loop → final summary. Stages exchange numbered markdown files in `tmp/<slug>/`, so a run is inspectable and resumable.

Works standalone, but it references companion skills at specific stages and degrades to improvisation where they're missing.

## Companion skills

Pulled from separate repos — install alongside for the full experience:

| Skill | Used at | Without it |
|---|---|---|
| [think](https://github.com/volume-02/think) | Stage 2 — plan-first discipline | planning loses the forced think-before-code pass |
| [fix-issue](https://github.com/volume-02/fix-issue) | Stage 0/1 — ticket triage, premise verification | divergences between ticket and reality surface later, during implementation |
| [html-report](https://github.com/volume-02/html-report) | plan & final checkpoints | checkpoints stay as plain markdown |
| [pipeline-audit](https://github.com/volume-02/pipeline-audit) | after a run — retro that hardens this skill | lessons from the run aren't fed back |
| [figma-skills](https://github.com/volume-02/figma-skills) (analyze / audit / implement / refactor) | the design/Figma track | design-type tasks fall back to generic implementation |

Also referenced, optional:

- **graphify** — AST/knowledge-graph code search; needs an external indexer CLI and a pre-built index, not published. Plain search is used instead.
- **Linear MCP** — ticket integration; any tracker (or none) works.

## Workspace overlays

`contexts/<workspace>.md` files adapt the pipeline to a specific workspace (paths, ticket conventions, helpers). None are bundled — without one the pipeline runs in vanilla mode. Write your own per workspace as needed.

## Install

```sh
cp -R pipeline ~/.claude/skills/        # global
# or
cp -R pipeline <project>/.claude/skills/  # per-project
```
