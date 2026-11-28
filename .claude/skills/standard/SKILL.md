---
name: standard
description: Turn a correction into a turple-dog convention. Use when the user corrects how code, UI, tests, packaging or deployment should be done, or says "always" or "never" do something that applies beyond the current change (rule E1), or runs /standard <rule>. Proposes the rule as a PR to turple-dog/engineering-standards so every repo gets it on the next sync.
---

# Propose a standard

The conventions live in `CONVENTIONS.md` in `turple-dog/engineering-standards`.
Every org repo gets them through a synced block in its `CLAUDE.md` and
`AGENTS.md`, so a rule merged there reaches every repo and every agent.

## 1. Apply it now

Fix the current change to follow the correction first. The rule PR is in
addition to that, not a substitute for it.

## 2. Decide where it belongs

- **Applies to any repo, or to a kind of project** (all UI, all cloud apps):
  it goes in `CONVENTIONS.md`. Continue below.
- **Only fits this repo:** add it to this repo's own `CLAUDE.md` and
  `AGENTS.md`, outside the `BEGIN`/`END turple-dog/engineering-standards`
  markers, and stop here.
- **Unclear:** ask the user one short question: "every repo, or just this one?"

## 3. Write the rule

Get a fresh copy of the standards on a new branch:

- On the homelab: `~/repos/engineering-standards`. Run `git pull` on `main`
  first; if the working tree isn't clean, stop and tell the user.
- Anywhere else: `gh repo clone turple-dog/engineering-standards` into a
  temp directory.

Then edit `CONVENTIONS.md`:

- Read the whole file. If an existing rule already covers this, tighten that
  rule instead of adding a new one. If the correction contradicts a rule,
  change that rule and say so in the PR.
- Put the rule in the section it fits, with the next free ID for that
  section's letter (`U8` after `U7`). Never renumber or reuse an ID: reviews
  and PRs cite them. A new section gets an unused letter and an *Applies to*
  line if it doesn't cover every repo.
- Write it the way the file is written: one bold ID, one to three short
  sentences, a concrete example in backticks where it helps. Say what to do,
  not why it matters in general.
- If it can be enforced by a lint rule, test or hook (E2), add a sentence
  naming the check.

## 4. Open the PR

Commit on a branch named `standard/<short-slug>` and open a PR against
`main`. In the description, quote the user's correction, give the rule ID,
and name any rule that changed. Don't merge it: the user reviews it, and
merging opens the sync PRs in every repo.

Give the user the PR link in one line.
