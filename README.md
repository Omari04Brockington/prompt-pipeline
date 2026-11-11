# prompt-pipeline

A [Claude Code](https://claude.com/claude-code) plugin with two skills:

- **`prompt-pipeline`** turns a rough request into approved, verified work.
- **`deep-review`** reviews the result in a fresh session, checking code quality,
  codebase fit, spec conformance, and impact on the product's users.

## The pipeline

| Stage | What happens |
|-------|--------------|
| **0 - Gate** | Trivial asks are answered normally; substantial ones run the pipeline. |
| **1 - Scout: Repo** | Find what already exists: rules, related code, conventions, product context, CI checks. |
| **2 - Scout: Practices** | Research current best practices from dated primary sources. |
| **3 - Grill** | Decide only what changes the outcome, with evidence-backed multiple-choice questions. |
| **4 - pipeline.md** | Write a transient plan (goal, criteria, decisions, steps, verification, risks) outside the repo. |
| **5 - Plan check** | An independent agent audits `pipeline.md` before you see it. |
| **6 - Approval** | You approve, revise, or cancel. Nothing changes before approval. |
| **7 - Execute and verify** | Agents do the work and run every check; material divergence returns for approval. |
| **8 - Handoff** | The pull request description becomes the review brief; `pipeline.md` is deleted. |

Every stage is skippable on request ("skip scouting", "no grilling", "just do it").

## Deep review

Four independent finders (Quality, Implementation fit, Spec conformance, User and
product impact) and one adjudicator that re-reads the code, drops anything below 80%
confidence, and ranks the rest by user impact. Every finding says **what happens if
it is left unresolved**, and the review ends with a verdict.

## Install

```
/plugin install prompt-pipeline --marketplace Omari04Brockington/prompt-pipeline
```

Or in two steps:

```
/plugin marketplace add Omari04Brockington/prompt-pipeline
/plugin install prompt-pipeline@omari-skills
```

If you previously cloned this repo into `~/.claude/skills/prompt-pipeline`, remove that
folder (and any separate `~/.claude/skills/deep-review`) after installing, so each
skill loads only once.

## Use

```
/prompt-pipeline:prompt-pipeline <your rough request>
```

Then, in a new session after the handoff:

```
/prompt-pipeline:deep-review pr=<number>
```

Both skills can also trigger automatically from their descriptions.

## License

MIT - see [`LICENSE`](./LICENSE).
