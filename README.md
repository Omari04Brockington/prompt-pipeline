# prompt-pipeline

A [Claude Code](https://claude.com/claude-code) **skill** that turns a rough request
into refined, reviewed work through a six-stage workflow — with independent review
gates before and after execution. Every stage is skippable on request.

## The six stages

| Stage | What happens |
|-------|--------------|
| **0 — Gate** | Judge complexity. Trivial asks are answered normally; substantial ones run the pipeline. |
| **1 — Prompt engineering** | Interactively refine the request into the best possible prompt (revised prompt + suggestions + option-picker questions). |
| **2 — Prompt review** | An independent agent audits the finalized prompt for ambiguity, missing constraints, and scope. |
| **3 — Plan** | Enter plan mode; produce a concrete implementation plan and get approval. |
| **4 — Plan review** | An independent agent reviews the plan for gaps, wrong assumptions, and ordering hazards. |
| **5 — Execution** | Do the work per the approved plan, keeping the build green. |
| **6 — Code review** | Three parallel finders (correctness · security/edge-cases · simplicity) + one adjudicator produce a single ranked list; confirmed issues are fixed. |

See [`SKILL.md`](./SKILL.md) for the full instructions.

## Install

Skills are global and personal, living under `~/.claude/skills/`. Clone this repo into
that location so the folder name matches the skill name:

```bash
git clone https://github.com/<owner>/prompt-pipeline.git ~/.claude/skills/prompt-pipeline
```

To edit and version your own copy, just edit `SKILL.md` in place and commit — the
running Claude Code picks up changes on the next session.

## Use

In Claude Code, invoke it explicitly:

```
/prompt-pipeline <your rough request>
```

…or let it trigger automatically on the first substantial task of a session. Skip any
stage by saying "skip planning", "just do it", "no review", etc.

## License

MIT — see [`LICENSE`](./LICENSE).
