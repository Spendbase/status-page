---
name: commit
description: Create a git commit whose subject follows Conventional Commits (fix:, feat:, ci:, docs:, chore:, refactor:, test:, perf:, style:, build:). Use whenever the user asks to commit, make a commit, or stage-and-commit changes in this repo. Mandatory for all commits in Spendbase/status-page. Do NOT add any Co-Authored-By trailer.
---

# `/commit` — Conventional Commits for Spendbase/status-page

Draft a commit message that conforms to the repo's commit convention, then make the commit.

## Required format

```
<type>[(<scope>)][!]: <imperative subject under 72 chars>

[optional body — wrapped at ~72 chars, explains the WHY]

[optional footer(s) — e.g. BREAKING CHANGE: ...]
```

- `<type>` **must** be one of exactly: `fix`, `feat`, `ci`, `docs`, `chore`, `refactor`, `test`, `perf`, `style`, `build`.
- `<scope>` is optional and lowercase (e.g. `fix(health-check): …`).
- Trailing `!` marks a breaking change (also add a `BREAKING CHANGE:` footer).
- Subject is lowercase, imperative, no trailing period.
- **Never** add `Co-Authored-By: …`, `Signed-off-by:`, or any Claude/AI attribution trailer. The user wants commits to look fully human-authored.

## Picking the type

| Change | Type |
|---|---|
| Anything under `.github/workflows/` or `scripts/health-check.sh` | `ci` |
| `README.md`, `CLAUDE.md`, inline doc comments only | `docs` |
| User-visible new functionality (new service, new UI section, new config knob) | `feat` |
| Fixing a bug in existing behavior | `fix` |
| Rename/move/restructure, no behavior change | `refactor` |
| Add/update tests only | `test` |
| Measurable speed/memory improvement | `perf` |
| Formatting, whitespace, Tailwind-class reordering, no code change | `style` |
| `package.json` deps, `next.config.js`, `tsconfig.json`, `tailwind.config.js`, build tooling | `build` |
| Repo housekeeping that doesn't fit elsewhere (gitignore, LICENSE, meta files) | `chore` |

When a change spans types, pick the one that best describes the **primary user-facing effect** of the change, not the one with the most lines.

## What the skill does when invoked

Run these steps in order. Keep tool calls batched where independent.

1. **Gather context** (parallel):
   - `git status`
   - `git diff --staged` and `git diff` (unstaged)
   - `git log --oneline -10` to match house tone
   - `git branch --show-current`
2. **Decide what to stage.**
   - If the user named specific files, stage only those with `git add -- <paths>`.
   - If anything is already staged, respect that selection and don't widen it unless the user asked for "stage everything".
   - **Never** use `git add -A` or `git add .` blindly — they can grab `.env`, local scratch files, or editor artifacts. If the user says "commit everything", enumerate the untracked files first and confirm before adding.
3. **Draft the message.**
   - Pick the type from the table above based on the staged diff.
   - Write the subject in imperative mood ("remove X", not "removed X" / "removes X").
   - Add a body only when the WHY isn't obvious from the diff. Skip it for one-line, self-evident changes.
   - Do **not** reference tickets, PR numbers, or Claude unless the user explicitly asks.
4. **Commit** with a HEREDOC so the body formats correctly:
   ```bash
   git commit -m "$(cat <<'EOF'
   <type>: <subject>

   <body if any>
   EOF
   )"
   ```
5. **Verify** with `git status` after and report the resulting commit hash + subject.

## Hard rules

- **Never** `--amend` an existing commit unless the user explicitly asked. Create a NEW commit on top instead.
- **Never** `--no-verify` or skip hooks.
- **Never** `git push` from this skill. Committing is in scope; pushing is a separate step the user initiates.
- If a pre-commit hook fails: fix the underlying issue, re-stage, make a NEW commit. Do not amend.
- If the staged diff is empty, stop and tell the user — do not create an empty commit.
- Refuse to stage files that look like secrets (`.env*`, `*.pem`, `credentials*.json`, `id_rsa*`). Warn the user if they explicitly request such a file.

## Examples

Diff: removed `.github/workflows/nextjs.yml` (stub file).
→ `ci: remove stub nextjs.yml workflow`

Diff: added `CLAUDE.md` for repo onboarding.
→ `docs: add CLAUDE.md with architecture and workflow notes`

Diff: new entry in `public/urls.cfg` for a new service.
→ `feat: monitor billing.spendbase.co`

Diff: bump `next` from 15.5.26 to 15.6.0 in `package.json` + `yarn.lock`.
→ `build: bump next to 15.6.0`

Diff: fix off-by-one in `useServices.tsx` day grouping.
→
```
fix: include today in 90-day summary window

The for-loop ran `-1..89`, pushing the current day twice when the
first log of the day arrived just before midnight UTC. Start from 0.
```
