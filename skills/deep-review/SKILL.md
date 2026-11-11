---
name: deep-review
description: A four-finder-plus-adjudicator review that checks a diff for code quality, fit with the existing codebase, conformance to what was asked for, and impact on the product's users. Every surviving finding says what happens to users or the platform if it is left unresolved, and findings are ranked by that impact. Runs standalone ("review my changes") or on a pull request handed off by the sibling prompt-pipeline skill (pr=<number>).
when_to_use: Invoke when the user asks to review a diff, branch, pull request, or working-tree changes, or after prompt-pipeline's Handoff stage in a fresh session with pr=<number> or branch=<name>.
---

# Deep Review

Reviews a set of changes against a brief (what was asked for), a codebase map (how
this codebase already does things), and a product map (who the users are and what
the product promises). Four independent finders report; one adjudicator re-verifies
every finding, scores confidence, assigns user impact, and states what happens if the
finding is left unresolved.

The goal is work that is both technically sound and good for the people who use the
product. A correct change that confuses or misleads users is not done.

## 0 - Mode detection

Read the `args` string you were invoked with:

- `pr=<number>`: **handoff mode** on a pull request.
- `branch=<name>`: **handoff mode** on a branch compared with the default branch.
- `range=<a>..<b>` or a named ref ("since main"): standalone mode on that range.
- Anything else, or no args: **standalone mode** on the working tree.

Each value runs up to the next space or the end of the string.

## 1 - Diff scope

The target is the repository being reviewed, normally the current working directory.
Check it is a git repo (`git -C <target> rev-parse --is-inside-work-tree`).

- **`pr=<n>`**: `gh pr diff <n>`. If `gh` is unavailable, fetch the PR's head and base
  and use `git diff <base>...<head>`.
- **`branch=<name>`**: find the default branch (`git symbolic-ref
  refs/remotes/origin/HEAD --short`, else try `main`, then `master`) and use
  `git diff <default>...<name>`.
- **Named range**: `git diff <ref>...HEAD`.
- **Standalone, no args**: `git diff HEAD` (working tree plus staged) plus untracked
  files.
- **Not a git repo**: ask which files or folders to review. Never assume a scope.

If a diff is empty, say so and stop rather than reviewing nothing. Keep the diff text;
every finder and the adjudicator receive it.

## 2 - Review brief

Materialize the brief as a file in the session's scratchpad or temporary directory.
Never write it into the target repository and never commit it.

**Handoff mode (`pr=`):** read the pull request title and description
(`gh pr view <n> --json title,body`). A prompt-pipeline handoff contains Goal,
Acceptance criteria, Scope, Decisions, Verification, and Risks. Build the brief from
those sections without asking anything. If the description lacks a goal or acceptance
criteria, say so, then fall back to the interactive brief below.

**Standalone, or handoff without a usable description:** build it interactively,
capped at 3 rounds:
- Tell the user up front: "I'll ask a couple of quick questions to build a review
  brief - up to 3 rounds, then I'll proceed either way."
- Round 1: use `AskUserQuestion` (up to 4 questions) covering the goal of the change,
  whether there is a spec or ticket to check against and where, who the change is for,
  and areas to scrutinize. Always include an option to skip straight to "draft from the
  diff alone".
- Draft the brief from the diff plus the answers. Show it with an option literally
  "Approve this brief - proceed to review."
- If not approved: round 2 with narrower questions, redraft, show again. Same for
  round 3.
- If not approved after round 3, proceed with the latest draft but prepend: "**Note:
  this brief was not explicitly approved - proceeding with the latest draft.**" This
  note travels into the finder prompts and into what the user sees.
- "Finished", "approve", or "go" at any point counts as approval.

Show the brief (or a short summary if long) before moving on.

## 3 - Codebase and product map

**Cache location.** Compute a slug from the target's absolute path: strip the leading
`/` and replace every remaining `/` with `-` (for example `/home/dev/my-app` becomes
`home-dev-my-app`). The cache file is `~/.claude/cache/codebase-maps/<slug>.md`.
Cloud sessions do not keep this cache between sessions, so expect regeneration there.

**Reuse or regenerate.** The map's first lines record the commit it was built from and
the date. Reuse it unless one of these is true, in which case regenerate:
- The user asks to refresh it ("refresh the map", "the map is stale").
- The file is missing.
- The recorded commit is not an ancestor of `HEAD`, or more than 50 commits or 30 days
  separate it from `HEAD`.

When reusing, mention the map's date and that "refresh the map" forces regeneration.

**Generate.** Create the cache directory if needed (`mkdir -p`), then launch two agents
in parallel:

1. **Structure map** - pinned to a cheap, fast model (for example Haiku), since this is
   mechanical description. Brief, verbatim (fill in `<target>`):

   > Explore the directory `<target>`. Produce a markdown document describing its
   > structure and purpose. If this is a software repository, cover only the segments
   > that actually apply - e.g. backend, frontend, CI/CD, infra, data, docs, tests -
   > and omit ones that don't exist; don't invent categories to fill a template. If
   > this is not a software repository (e.g. a notes vault, knowledge base, or
   > docs-only collection), instead describe whatever organizational structure
   > actually exists - folder taxonomy, note types, templates, indices, tagging or
   > linking conventions - using whatever sections make sense; do not force software
   > categories onto it. Be concrete: reference real paths and file names. Keep
   > exploration proportional to size: skim structure and a representative sample of
   > files, don't read everything. If the target has roughly more than 2,000 files,
   > explicitly sample - top 2-3 directory levels plus a handful of representative
   > files per top-level folder - and say in the map that it's a sampled overview, not
   > exhaustive.

2. **Product and users** - a capable model (for example Sonnet), since this needs
   judgment. Brief, verbatim:

   > Read the product sources in `<target>`: README, CLAUDE.md or AGENTS.md, and any
   > vision, specification, roadmap, or user-facing documentation. Write a markdown
   > section titled "Product and users" covering: who the users are (including distinct
   > groups, such as beginners versus experts); what the product promises them; the
   > core user flows; things the product must never do (safety, trust, legal, money,
   > or data rules); and quality bars stated in the documents. Quote or cite the source
   > file for each point. Distinguish what is implemented from what is documented as a
   > target. If the repository has no product documentation, say so in one line
   > rather than guessing.

Write the combined map **yourself** to the cache path, starting with a header line
recording the commit (`git rev-parse HEAD`) and the date. Subagents do not write to
this path. Show the user a short summary before launching the finders.

## 4 - Four parallel finders

Launch all four in a single message (four concurrent Agent calls) so they run in
parallel.

**Choose the agent type once and reuse it for all finders and the adjudicator:** check
the subagent types available in this session right now; do not assume a name exists.
If an available type's name contains `code-review` or `code-reviewer`
(case-insensitive), use it. Otherwise use `general-purpose` with an explicit Sonnet
model override.

Give each finder **only the context it uses**, pasted inline as text (not paths):

| Finder | Receives |
|---|---|
| Quality | diff |
| Implementation fit | diff + structure map |
| Spec conformance | diff + review brief |
| User and product impact | diff + review brief + "Product and users" section |

Tell every finder explicitly: **work only from the text given in this prompt - do not
explore the target repository, run Bash or Grep, or read other files.** If something
is unclear from what is given, note the uncertainty in the finding rather than
investigating.

Every finder returns each finding as: a one-line title, a `file:line` citation (or
`file:section` for non-code content), a one or two sentence description, and a
concrete failure scenario. Each also lists anything well done, tagged clearly as a
compliment, not a finding.

**Quality finder:**
> Review the diff above for code quality: best practices, correctness, readability,
> error handling, security, edge cases, duplication, and naming.

**Implementation-fit finder:**
> Using the structure map above, review the diff for whether it was built the right way
> given how the rest of this codebase already does things - not "is this good code in
> isolation" but "does it fit this codebase's existing patterns, structure, and
> conventions." Flag places where the diff introduces a different pattern than the map
> shows is established, without a stated reason.

**Spec-conformance finder:**
> Using the review brief above, check whether the diff matches what was asked for.
> Report: (a) requirements that are missing or partial; (b) behavior that wasn't
> requested (scope creep); (c) requirements that look implemented but are actually
> wrong or incomplete. Cite the specific part of the brief for each finding, plus the
> file:line it corresponds to. If the brief is empty, missing, or contains no actual
> requirements, say so explicitly and return no findings for this axis rather than
> inventing a spec from the diff alone.

**User and product impact finder:**
> Using the review brief and the "Product and users" section above, review the diff
> from the point of view of the people who use this product. Look for: behavior a user
> would find confusing, misleading, or surprising; outputs that could be mistaken for
> something stronger than they are (for example an estimate shown as a fact or a
> recommendation); broken or degraded core flows; error states a user cannot
> understand or recover from; changes that serve one user group but hurt another; and
> anything that violates a stated "must never" rule or quality bar. For each finding,
> name the affected user group and the flow. If the product section says there is no
> product documentation, review only against the brief and say so.

## 5 - Adjudicator

After all four finders return, spawn one more agent (same type and model as step 4).
Give it: all four finders' raw output verbatim, the diff, the brief, and the full map.
Instruct it, verbatim:

> Re-verify every finding above against the actual current file content and the
> line(s) cited - do not trust a finder's description at face value. Read the file
> yourself. If a finding cites the wrong line, misdescribes what the code does, or the
> described problem doesn't actually exist in the current content, drop it. If
> multiple findings describe the same underlying issue, merge them into one and list
> every finder that raised it.
>
> For every finding that survives verification, score your confidence that it is real
> on a continuous 0-100 scale, using this rubric as calibration anchors (not discrete
> buckets - you may land anywhere on the scale, e.g. 62 or 88):
> - 0: Not confident at all. This is a false positive that doesn't stand up to light
>   scrutiny, or is a pre-existing issue.
> - 25: Somewhat confident. This might be a real issue, but may also be a false
>   positive. You weren't able to verify that it's a real issue.
> - 50: Moderately confident. You were able to verify this is a real issue, but it
>   might be a nitpick or not happen very often in practice.
> - 75: Highly confident. You double-checked the issue and verified it is very likely
>   a real issue that will be hit in practice.
> - 100: Absolutely certain. You double-checked the issue and confirmed it is
>   definitely real. The evidence directly confirms this.
>
> Drop any finding scoring below 80. Confidence decides whether a finding is real; it
> does not decide how much it matters.
>
> For every surviving finding, assign an impact level using the "Product and users"
> section and the brief:
> - Critical: users lose money or data, see wrong or misleading information presented
>   as reliable, a "must never" rule is broken, or security is compromised.
> - High: a core user flow breaks, misleads, or blocks users; or a stated
>   requirement is not met.
> - Medium: a degraded or confusing experience with a workaround, or a maintenance cost
>   that will slow future work.
> - Low: polish, naming, or minor inconsistency with no user-visible effect.
>
> Then write "If left unresolved" for each finding in plain language: who is affected
> (which users, or the platform and its maintainers), when it happens (what triggers
> it), and what they experience or what it costs. Ground it in the product section,
> not in generic statements.

Rank survivors by impact (Critical, High, Medium, Low), then by confidence (highest
first), then by number of finders that raised it, then alphabetically by file path.

**Verdict:**
- **Fix before merging** if any Critical or High finding survives.
- **Ship, with follow-ups** if only Medium or Low findings survive.
- **Ship** if none survive.
- **Needs discussion** instead, when a finding depends on a product decision the brief
  does not settle. Name the decision.

Produce ONE finalized review using exactly this template:

```
# Deep Review

**Verdict:** <Fix before merging | Ship, with follow-ups | Ship | Needs discussion: <decision>>
**Scope:** <one line: what was diffed, e.g. "PR #12, 6 files">
**Brief:** <one line summary, or "none provided - spec conformance skipped">
**Maps:** <cache path and date, or "generated fresh this run">

## Findings

### 1. <short title> - Impact: Critical (confidence <score>/100)
- **Raised by:** <Quality | Implementation fit | Spec conformance | User and product impact>
- **Location:** `<file:line>` (or `<file:section>`)
- **Issue:** <1-2 sentence description>
- **Failure scenario:** <concrete inputs or state -> wrong output, crash, or consequence>
- **If left unresolved:** <who is affected, when, and what they experience or what it costs>
- **Recommendation:** <what to change>

### 2. <short title> - Impact: High (confidence <score>/100)
...

## Looks good
- <thing a finder explicitly praised> (<finder>, `<file:section>`)

## Summary
<N> finding(s) surfaced (<c> critical, <h> high, <m> medium, <l> low). <M> raw
finding(s) from the four finders were checked and dropped (false positive,
unverifiable, or below the confidence threshold).
```

If no findings survive, replace `## Findings` with the single line
`No findings above the confidence threshold.` and adjust the summary line.

## 6 - Consumption

Present the finalized review to the user. Do not fix anything, and do not post
comments to the pull request, unless the user asks. When asked to fix, start with
Critical and High findings, and confirm anything ambiguous or high-cost first.

## Notes

- `prompt-pipeline` ships beside this skill in the same plugin. Its Handoff stage
  writes the pull request description that handoff mode reads.
- Never write the review brief into the target repository or commit it.
- The only durable artifact this skill writes is the map cache file, and only under
  `~/.claude/cache/codebase-maps/`.
- If scope is ambiguous, ask rather than guess. Silently picking "everything" or
  "nothing" is wrong in both directions.
