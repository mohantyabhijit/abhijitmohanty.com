+++
title = "Git Workflows That Actually Scale"
date = 2025-09-22T09:00:00+05:30
description = "A deep dive into trunk-based development, feature flags, and why long-lived branches cause more problems than they solve."
tags = ["git", "workflow", "engineering"]
slug = "git-workflows-that-scale"
draft = false
+++

Most git workflow debates are about the wrong thing. People argue about Git Flow vs. GitHub Flow vs. trunk-based development as if the branching model is the hard part. It is not. The hard part is keeping the feedback loop short.

A workflow that lets you merge code multiple times a day is fundamentally different from one where branches live for weeks. The first one forces you to write small, safe changes. The second one lets you accumulate risk until merge day becomes a disaster.

## The Problem with Long-Lived Branches

I have worked on teams that used Git Flow religiously. Feature branches that lived for two weeks. Release branches that accumulated cherry-picks. Hotfix branches that diverged from develop. The merge conflicts alone consumed hours every sprint.

Long-lived branches cause three specific problems:

### 1. Merge Conflicts Scale Exponentially

If two developers each change 10 files over two weeks, the probability of a conflict is high. If they each merge daily changes of 1-2 files, conflicts are rare and trivial.

The math is simple: conflict probability increases with the number of changed lines and the time those changes remain unintegrated.

### 2. Integration Bugs Hide Until the Worst Moment

When you merge a two-week branch, you are integrating two weeks of assumptions about the rest of the codebase. Some of those assumptions are wrong. You just do not know which ones until the merge.

I have seen a "simple" branch merge break authentication because someone renamed a shared utility function on the main branch three days after the feature branch was created. The feature branch kept using the old name. Tests passed on both branches independently. The merge compiled fine. The bug only showed up in production.

### 3. Code Review Quality Degrades

A pull request with 2000 lines changed across 40 files does not get a real review. Reviewers skim it, approve it, and pray. A pull request with 80 lines across 3 files gets genuine scrutiny.

```
PR size vs. review quality:

  50 lines  ████████████████████  thorough review
 200 lines  ████████████          decent review
 500 lines  ████████              surface scan
1000 lines  ████                  "looks good to me"
2000 lines  ██                    rubber stamp
```

## Trunk-Based Development

The alternative is trunk-based development: everyone commits to `main` (or a very short-lived branch that merges within a day). This sounds scary until you add the safety nets.

### The Rules

1. **Branches live for at most one day.** If it cannot be merged today, break it into smaller pieces
2. **All commits must pass CI.** No exceptions. If the build is red, fixing it is the top priority
3. **Use feature flags for incomplete work.** The code is in production, but the feature is not visible to users until the flag is enabled
4. **Backward-compatible changes only.** Database migrations, API changes, and schema updates must work with both the old and new code simultaneously

### A Typical Day

```bash
# Morning: pull latest, create a short branch
git checkout main && git pull
git checkout -b add-rate-limiting

# Work for a few hours, commit frequently
git add -A && git commit -m "add rate limiter middleware"
git add -A && git commit -m "wire rate limiter into API routes"
git add -A && git commit -m "add rate limit exceeded response"

# Afternoon: push, get a quick review, merge
git push -u origin add-rate-limiting
# Open PR, reviewer approves within an hour
# Squash merge into main
# Delete branch

# If not done by end of day, still merge what's safe
# Hide incomplete parts behind a feature flag
```

## Feature Flags Make This Possible

The biggest objection to trunk-based development is: "What if the feature takes a week? I cannot ship half-built features."

Feature flags solve this completely. You merge code to main behind a flag. The code is deployed but inactive:

```go
func handleDashboard(w http.ResponseWriter, r *http.Request) {
    user := getUser(r)
    
    if featureflags.IsEnabled("new-dashboard", user) {
        renderNewDashboard(w, r)
    } else {
        renderOldDashboard(w, r)
    }
}
```

Benefits of this approach:

- **Code is integrated immediately.** No merge conflicts later
- **You can test in production** by enabling the flag for internal users
- **Rollback is instant.** Flip the flag, no deploy needed
- **Gradual rollout is built in.** Enable for 1% of users, then 10%, then 50%, then 100%

### Feature Flag Hygiene

Feature flags are technical debt if you do not clean them up. After a feature is fully rolled out:

1. Remove the flag check from code
2. Remove the flag from your flag service
3. Delete the old code path

I set a calendar reminder for 2 weeks after full rollout. If the feature is stable, clean up. If not, you have a fast rollback mechanism.

## Database Migrations in Trunk-Based Development

Database changes are the trickiest part. You cannot just merge a migration that renames a column if the old code is still running.

The pattern is **expand and contract**:

### Step 1: Expand (Add the New Thing)

```sql
-- Migration: add new column, keep old one
ALTER TABLE users ADD COLUMN full_name VARCHAR(255);

-- Backfill new column from old columns
UPDATE users SET full_name = CONCAT(first_name, ' ', last_name);
```

Deploy code that writes to **both** columns but reads from the old one.

### Step 2: Migrate (Switch Reads)

Deploy code that reads from the new column. Writes still go to both.

### Step 3: Contract (Remove the Old Thing)

```sql
-- Migration: drop old columns (only after all code uses new column)
ALTER TABLE users DROP COLUMN first_name;
ALTER TABLE users DROP COLUMN last_name;
```

This is more work than a single migration. But it means zero downtime and every intermediate state is safe to roll back.

## When Trunk-Based Development Does Not Work

I will be honest about the prerequisites. Trunk-based development requires:

- **Fast CI** (under 10 minutes). If your build takes 45 minutes, developers will batch changes to avoid waiting
- **Good test coverage.** You need confidence that green CI means safe to deploy
- **Feature flag infrastructure.** Even a simple config file works to start
- **Team discipline.** Everyone must commit to small changes and fast reviews
- **Automated deployment.** Manual deploy processes create bottlenecks

If your team does not have these yet, start by fixing CI speed and test coverage. The branching model is downstream of your engineering maturity.

## The Branching Model I Actually Use

In practice, I use a slight variation:

```
main (always deployable)
  │
  ├── feature/add-rate-limiting     (lives < 1 day)
  ├── feature/update-user-api       (lives < 1 day)  
  └── fix/timeout-handling          (lives < 1 day)
```

- `main` is always deployable
- Feature branches are short-lived (hours, not days)
- No `develop` branch. No `release` branch. No `hotfix` branch
- Tags for releases if needed: `git tag v1.2.3`
- Staging branch for preview deployments (like this blog uses)

## Summary

The best git workflow is the one that keeps changes small and integration frequent:

1. Merge to main at least once a day
2. Use feature flags for incomplete work
3. Use expand-and-contract for database changes
4. Keep CI fast so developers are not afraid to push
5. Review small PRs thoroughly instead of rubber-stamping large ones

The branching model is a tool. The goal is short feedback loops and low integration risk. Pick whatever gets you there.
