---
name: prompt-pipeline
description: A structured prompt-to-production workflow. Turns a rough task request into a refined prompt, then drives it through independent review gates before and after execution. Use for any substantial build, change, feature, or multi-step task; skip for quick questions or trivial one-line fixes.
when_to_use: Invoke on the first substantial task request of a session, or whenever the user wants a rough idea shaped into a strong prompt and executed with review gates. Skip for lookups, explanations, and trivial edits.
---

# Prompt Pipeline

A six-stage workflow that turns a rough request into refined work, with independent
review before and after execution. Every stage is **skippable on request** - if the
user says "skip planning", "just do it", "no review", etc., honor it and move on.
Announce each stage as you enter it so the user always knows where they are.

## Stage 0 - Gate (always run first)

Judge the request's complexity:
- **Trivial** (a question, a lookup, an explanation, a one-line fix): do NOT run the
  pipeline. Answer normally. Say nothing about the pipeline.
- **Substantial** (a build, feature, refactor, or multi-step task): run the pipeline.
  Briefly tell the user the pipeline is armed and that they can skip any stage.

If unsure, ask the user whether they want the full pipeline or a direct answer.

## Stage 1 - Prompt engineering (interactive)

You are the user's prompt creator. Your goal is to craft the best possible prompt for
their need. **Treat the user's first message as the seed** - do not ask "what should
the prompt be about?" when they've already told you.

From the seed, produce exactly three sections, every iteration:

1. **Revised prompt** - your rewritten prompt: clear, concise, unambiguous, and directly
   actionable. This is the artifact that will drive the rest of the pipeline.
2. **Suggestions** - concrete details the user could add to make the prompt stronger
   (constraints, success criteria, edge cases, scope boundaries, examples).
3. **Questions** - anything you need from the user to improve the prompt further. **Ask
   these with the `AskUserQuestion` tool, not as prose**, so the user can press through
   choices instead of typing free-form answers. For each open decision, give a short
   `header`, a clear `question`, and 2-4 concrete `options` (each with a `label` and a
   `description` of the tradeoff). Put your recommended option first and mark it
   "(Recommended)". Use `multiSelect: true` when the choices aren't mutually exclusive.
   Batch related decisions into one call (up to 4 questions) rather than asking serially.
   The user can always pick "Other" to type a custom answer, so reserve prose questions
   for genuinely open-ended input that options can't capture. If a given iteration raises
   no real decisions, skip this section rather than inventing questions.

Iterate: the user replies (via the option picker or free text), you update the **Revised
prompt** section, and repeat. Continue until the user says **"finished"** (or clearly
signals they're done). The finalized Revised prompt is the input to every later stage.

## Stage 2 - Prompt review (independent agent)

Before planning, spawn one subagent to audit the finalized prompt. Prompt it to find:
ambiguity, missing constraints, undefined success criteria, hidden assumptions, and
scope that's too broad or too narrow. Ask it to return a tightened prompt plus a short
list of gaps.

Use an efficient general-purpose agent (`subagent_type: general-purpose`). Show the user
the reviewer's findings, fold the accepted improvements into the prompt, and confirm the
final version before moving on.

## Stage 3 - Plan (plan mode)

Enter plan mode (`EnterPlanMode`). Develop a concrete implementation plan for the
finalized prompt: the files/systems touched, the approach, the sequence of changes, and
the verification strategy. Present the plan and get the user's approval via
`ExitPlanMode` before writing any code.

## Stage 4 - Plan review (independent agent)

After the user approves the plan but **before execution**, spawn one subagent to review
the plan for gaps and misunderstandings: missed files, wrong assumptions about the
codebase, unhandled edge cases, ordering hazards, and anything the plan misreads about
the user's intent. Relay its findings, adjust the plan, and reconfirm with the user if
the changes are material.

## Stage 5 - Execution

Do the work per the approved plan. Follow the repo's conventions and the user's global
instructions. Keep the build green as you go (lint, tests). If reality diverges from the
plan, surface it rather than silently improvising.

## Stage 6 - Code review (hybrid: 3 finders + 1 adjudicator)

After execution, review the changes with a hybrid topology:

1. **Three parallel finders** - launch three subagents in a single message so they run
   concurrently, each with a distinct lens:
   - **Correctness** - logic errors, broken behavior, wrong outputs, regressions,
     missing test coverage.
   - **Security & edge cases** - injection, unsafe input handling, boundary conditions,
     error paths, resource/permission issues.
   - **Simplicity & maintainability** - dead code, needless complexity, poor naming,
     duplication, drift from surrounding conventions.
   Give each the diff/changed files and ask for findings ranked by severity, each with a
   concrete failure scenario. Prefer a code-review-specialized agent type if one is
   available in the session; otherwise use `general-purpose`.

2. **One adjudicator** - after the three return, spawn a single agent (or do it yourself)
   to merge their findings: de-duplicate overlaps, resolve conflicts, drop false
   positives, and produce one severity-ranked list.

3. **Fix** - present the adjudicated findings to the user, then fix the confirmed issues.
   For anything ambiguous or high-cost, confirm with the user before changing it.

## Notes

- This skill is personal and global (`~/.claude/skills/`). It is not part of any repo.
- The `SessionStart` hook only *reminds* you to consider this skill on the first
  substantial task; the real gate is Stage 0's judgment. Never run the pipeline for
  trivial asks just because the hook fired.
- Respect the user's global instructions at every stage (e.g. commit/push only when
  asked, no em dashes, quality over development cost).
