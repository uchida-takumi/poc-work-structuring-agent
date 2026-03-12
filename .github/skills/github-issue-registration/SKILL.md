---
name: github-issue-registration
description: 'Create high-quality GitHub issues with consistent structure. Use for bug reports, feature requests, and project tasks. Includes triage checks, duplicate search, labels/milestones assignment, and completion validation before submit.'
argument-hint: 'Paste context (problem, repo, logs, desired outcome) and this skill will structure and submit an issue workflow.'
user-invocable: true
disable-model-invocation: false
---

# GitHub Issue Registration

Create clear, actionable GitHub issues that reduce back-and-forth and speed up triage.

## When to Use
- You need to register a new issue in GitHub.
- You want consistent quality for bug reports and feature requests.
- You want to avoid duplicate issues and missing acceptance criteria.

## Required Inputs
- Repository owner/name
- Issue type: `bug`, `feature`, or `task`
- Short title
- Problem statement or desired outcome

If key inputs are missing, ask focused follow-up questions before drafting.

## Workflow
1. Clarify the issue goal.
Determine issue type (`bug`, `feature`, `task`) and target repository.

2. Preflight checks.
Search for duplicates using keyword combinations from title, error messages, and main nouns. If likely duplicates exist, present them and ask whether to proceed.

3. Gather structured details.
For `bug` issues, collect reproducible steps, expected result, actual result, impact, and environment.
For `feature` issues, collect user problem, proposed solution, alternatives considered, and acceptance criteria.
For `task` issues, collect objective, scope boundaries, deliverables, and definition of done.

4. Draft the issue body.
Use the appropriate template from [Issue Templates](./references/issue-templates.md). Keep sections concise and evidence-based.
Default output language is Japanese unless the user requests otherwise.

5. Apply triage metadata.
Suggest labels, milestone, and priority based on content. If labels exist in the repository, align naming exactly.

6. Validate quality gates.
Confirm before submit:
- Title is specific and searchable.
- Scope is clear and non-overlapping.
- Reproduction or acceptance criteria are testable.
- Risks, blockers, and dependencies are listed when relevant.
- Sensitive data is removed.

7. Submit issue.
Prefer `gh issue create` when CLI auth is available; otherwise provide web-ready final title/body for manual submission.

8. Post-submit summary.
Return issue URL/number, applied metadata, and next recommended action.

## Decision Logic
- If issue type is unclear, ask one question to classify (`bug` vs `feature` vs `task`) before drafting.
- If duplicate confidence is high, recommend linking to existing issue instead of creating a new one.
- If details are incomplete, produce a draft marked `Needs Info` and list missing fields.
- If user asks for minimal mode, generate only `title + body` without metadata suggestions.

## Completion Checklist
- Duplicate search completed or explicitly skipped.
- Template sections filled for chosen issue type.
- Metadata proposal included (or intentionally omitted in minimal mode).
- Submission path completed (`gh` or manual).
- Final output includes issue link or copy-ready payload.
