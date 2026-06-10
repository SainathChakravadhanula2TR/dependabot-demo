# Dependabot Best Practices Guide

This document explains the configuration choices made in `.github/dependabot.yml` and why each one exists. The goal is to make automated dependency updates useful and low-noise, not a burden on the team.

---

## The Core Problems This Config Solves

Without any configuration, Dependabot behaves like this by default:
- Checks for updates **every day**
- Opens **one PR per outdated package** — a repo with 20 dependencies can wake up to 20 PRs overnight
- Proposes **major version upgrades** alongside minor ones — these often contain breaking changes that can silently break your app if merged without careful review

This config addresses all three.

---

## What We Changed and Why

### 1. Weekly Schedule Instead of Daily

```yaml
schedule:
  interval: "weekly"
  day: "monday"
  time: "09:00"
  timezone: "America/Chicago"
```

**Default behavior:** Dependabot checks daily and opens PRs immediately when it finds anything.

**What this does:** Checks run once a week, on Monday mornings. All updates found that week are batched together rather than trickling in as separate PRs throughout the week.

**Why it matters:** Daily checks create constant background noise. A weekly cadence means the team deals with dependency updates once a week, at a predictable time, rather than randomly throughout the day.

---

### 2. Blocking Major Version Updates

```yaml
allow:
  - dependency-name: "*"
    update-types:
      - "version-update:semver-minor"
      - "version-update:semver-patch"
```

**Default behavior:** Dependabot proposes all updates — patch, minor, and major — with no distinction.

**What this does:** Only minor and patch updates are allowed through. Major version bumps are never proposed.

**Why it matters:** Major versions (e.g. Flask 2 → 3, React 17 → 18) almost always contain breaking changes — removed APIs, changed behavior, new config formats. Merging them automatically is risky. They need a developer to read the migration guide, test carefully, and plan the upgrade. This config keeps those out of the automated flow entirely.

**Why `allow` and not `ignore`:** We use an `allow` whitelist rather than an `ignore` blacklist because Python packages (pip) do not always follow strict semantic versioning. The `ignore` approach with `update-types` can misclassify major bumps and let them through. An `allow` whitelist is more reliable — if the update type is not explicitly permitted, it does not get through, regardless of how pip classifies the version jump.

---

### 3. Grouping All Updates Into One PR

```yaml
groups:
  all-dependencies:
    patterns:
      - "*"
```

**Default behavior:** One PR per package. Ten outdated packages = ten PRs.

**What this does:** All eligible updates (after the major version filter above) are bundled into a single PR titled something like "Bump the all-dependencies group with 9 updates." That PR contains a table showing every package, its old version, and its new version.

**Why it matters:** Reviewing one grouped PR takes the same effort as reviewing one individual PR, but clears the entire backlog at once. It also reduces notification noise significantly — one PR notification instead of ten.

---

### 4. PR Cap

```yaml
open-pull-requests-limit: 10
```

**Default behavior:** No cap — Dependabot opens as many PRs as it finds updates for.

**What this does:** Limits concurrent open Dependabot PRs to 10. Once that limit is hit, no new PRs are opened until existing ones are merged or closed.

**Why it matters:** Prevents the PR queue from becoming overwhelming if the team falls behind on reviews. Acts as a natural backpressure mechanism.

---

### 5. Labels for Filtering

```yaml
labels:
  - "dependencies"
  - "python"
  - "automated"
```

**Default behavior:** Dependabot applies its own default label (`dependencies`) only.

**What this does:** All Dependabot PRs get tagged with `dependencies`, `python`, and `automated`. These labels must be created manually in the repo first (Issues → Labels → New label).

**Why it matters:** Teams can filter the PR list by label to quickly find or hide all automated dependency PRs. CI pipelines and auto-merge rules can also be scoped to only apply to PRs with these labels.

---

### 6. Commit Message Prefix

```yaml
commit-message:
  prefix: "chore"
  include: "scope"
```

**Default behavior:** Dependabot uses its own commit message format with no conventional-commit prefix.

**What this does:** All Dependabot commits are prefixed as `chore(deps):` — for example, `chore(deps): Bump flask from 2.2.0 to 2.3.3`.

**Why it matters:** Keeps the git log consistent with the team's conventional commit standard. Makes it easy to filter dependency updates out of changelogs or release notes if needed.

---

## Before vs. After

| Behavior | Without this config | With this config |
|---|---|---|
| How often Dependabot runs | Daily | Weekly (Mondays) |
| PRs opened per run | One per package | One grouped PR for all packages |
| Major version upgrades | Proposed automatically | Blocked — never proposed |
| PR queue risk | Can flood with 20+ PRs overnight | Capped at 10 |
| PR discoverability | Hard to filter | Labelled for easy filtering |
| Git log consistency | Dependabot's own format | Follows team's `chore(deps):` convention |

---

## Extending This Config

**To add a second language** (e.g. npm for a frontend project), add a second block under `updates:`:

```yaml
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
    allow:
      - dependency-name: "*"
        update-types:
          - "version-update:semver-minor"
          - "version-update:semver-patch"
    groups:
      all-dependencies:
        patterns:
          - "*"
```

**To assign a reviewer** to all Dependabot PRs, uncomment and fill in:

```yaml
    reviewers:
      - "your-github-username"
```

**To temporarily pause updates**, set:

```yaml
    open-pull-requests-limit: 0
```
