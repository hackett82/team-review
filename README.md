# Team — Review Scripts

This directory contains the Team Lead or Scrum Master tooling used to monitor daily development activity across the Team project. The scripts aggregate data from Git repositories and Azure DevOps, run automated code quality checks, and post a structured daily review to Microsoft Teams.

The primary entry point is `run-daily-review.js`, which orchestrates everything into a single consolidated report. The supporting scripts can also be run independently when you need a focused view of a specific area.

---

## Quick Start

```cmd
node reviews\scripts\run-daily-review.js --save
```

This runs the full daily review, saves the output to `reviews/daily-review-YYYY-MM-DD.txt`, and posts a two-part message to the configured Teams channel.

> **Prerequisite:** A `.env` file must exist at `reviews/scripts/.env` before any script will work. See [Prerequisites & Setup](#prerequisites--setup) below.

---

## Prerequisites & Setup

### Node.js
Node.js 18 or later is required. No `npm install` is needed — all scripts use only Node.js built-in modules (`fs`, `path`, `https`, `child_process`).

### `.env` File
Create `reviews/scripts/.env` by copying the structure below. This file is excluded from version control and should never be committed.

```env
# ─── Azure DevOps — Main (Work Items, Repos) ──────────────────────────────────
AZURE_DEVOPS_PAT=your_pat_here
AZURE_DEVOPS_ORG=https://dev.azure.com/PFCB-IT
AZURE_DEVOPS_PROJECT=PFC Systems (2026)
AZURE_DEVOPS_PROJECT_ID=your_project_guid_here
AZURE_DEVOPS_TEAM=Enterprise_WebApps_Team
AZURE_DEVOPS_ITERATION_ID=your_iteration_guid_here

# ─── Git Repositories (supports multiple repos) ───────────────────────────────
GIT_REPO_1_NAME=Website 3.0
GIT_REPO_1_PATH=C:/Repos/Website 3.0
GIT_REPO_1_BRANCHES=develop,master

GIT_REPO_2_NAME=pfcbconductor
GIT_REPO_2_PATH=C:/Repos/pfcbconductor
GIT_REPO_2_BRANCHES=develop,master

# ─── Output ───────────────────────────────────────────────────────────────────
REVIEWS_DIR=C:/Repos/Website 3.0/reviews

# ─── Analysis Thresholds ──────────────────────────────────────────────────────
LOOKBACK_HOURS=24
STALE_BRANCH_WARN_DAYS=14
STALE_BRANCH_CRIT_DAYS=30
STUCK_STORY_NOTICE_DAYS=3
STUCK_STORY_WARN_DAYS=5
STUCK_STORY_CRIT_DAYS=10
STUCK_STORY_AREA_PATH=PFC Systems (2026)\Enterprise_WebApps_Team

# ─── Large Commit & File Hotspot Thresholds ───────────────────────────────────
LARGE_COMMIT_FILES=10
LARGE_COMMIT_FILES_CRIT=20
LARGE_COMMIT_LINES=200
LARGE_COMMIT_LINES_CRIT=250
FILE_HOTSPOT_THRESHOLD=3

# ─── ADO Story Hygiene ────────────────────────────────────────────────────────
STORY_HYGIENE_AC_MIN_LENGTH=50
STORY_HYGIENE_REOPENED_LOOKBACK_DAYS=14

# ─── Pipeline Monitoring (separate ADO org) ───────────────────────────────────
PIPELINE_ADO_ORG=https://dev.azure.com/pfcbweb
PIPELINE_ADO_PROJECT=Automation_Website
PIPELINE_ADO_PAT=your_pipeline_pat_here
PIPELINE_WATCH_IDS=

# ─── Story Deep Dive ──────────────────────────────────────────────────────────
STORY_DEEP_DIVE_LOOKBACK_FILES=5

# ─── Microsoft Teams ──────────────────────────────────────────────────────────
TEAMS_WEBHOOK_URL=https://your-org.webhook.office.com/...
```

### Azure DevOps PAT Permissions
The PAT configured in `AZURE_DEVOPS_PAT` requires the following scopes:

| Scope | Access | Used For |
|---|---|---|
| Work Items | Read & Write | Querying stories, posting comments |
| Build | Read | Pipeline status |
| Code | Read | Cross-referencing commits to stories |

If `PIPELINE_ADO_ORG` is a different organization than `AZURE_DEVOPS_ORG`, create a separate PAT for that org and set it in `PIPELINE_ADO_PAT`. If omitted, the main PAT is used as a fallback.

#### How to Create a PAT in Azure DevOps

1. Sign in to `https://dev.azure.com/PFCB-IT`
2. Click your profile avatar in the top-right corner and select **Personal access tokens**
3. Click **+ New Token**
4. Fill in the fields:
   - **Name** — something descriptive, e.g. `Team Review Scripts`
   - **Organization** — select `PFCB-IT` (or the org matching `AZURE_DEVOPS_ORG`)
   - **Expiration** — set to the maximum allowed (typically 1 year); add a calendar reminder to rotate it before it expires
   - **Scopes** — select **Custom defined**, then check the following:

   | Scope | Permission |
   |---|---|
   | Work Items | Read & Write |
   | Build | Read |
   | Code | Read |

5. Click **Create**
6. **Copy the token immediately** — Azure DevOps will not show it again after you leave the page
7. Paste the token into `reviews\scripts\.env` as the value for `AZURE_DEVOPS_PAT`

> If you need a second PAT for a separate pipeline org (see `PIPELINE_ADO_PAT`), repeat these steps while signed in to that org and paste the result into `PIPELINE_ADO_PAT`.

Note: you will still need to potentially code or re-use an environment veriable for a PAT and so while adding more is potentially possible, make sure you are also include their use in the other javascript files.

### Adding a Third Git Repository
Add a new numbered block to `.env`:
```env
GIT_REPO_3_NAME=MyOtherRepo
GIT_REPO_3_PATH=C:/Repos/MyOtherRepo
GIT_REPO_3_BRANCHES=develop,main
```
The scripts will automatically include it in all commit queries, branch activity, and cross-reference checks.

---

## `run-daily-review.js` — Master Orchestrator

This is the primary script. It runs all data gathering and report sections in sequence, assembles them into a single output, and optionally saves and posts to Teams.

### CLI Flags

#### Section Flags — Run a single section instead of the full report

| Flag          | Runs           | Output File (with `--save`) |
|---            |---             |---                          |
| `--activity`  | Section 6 only | *(console only)* |
| `--codereview` | Section 5 only | `codereview-YYYY-MM-DD.txt` |
| `--comments`  | Section 2 only | *(console only)* |
| `--crossref`  | Section 3 only | `crossref-YYYY-MM-DD.txt` |
| `--pipeline`  | Section 7 only | *(console only)* |
| `--standup`   | Section 1 only | `standup-YYYY-MM-DD.txt` |
| `--synopsis`  | Section 4 only | `git-synopsis-YYYY-MM-DD.txt` |

#### Action Flags

| Flag              | What It Does |
|---                |---|
| `--dry-run`       | Use with `--post-comments` to preview the comment text that would be posted without actually posting to ADO |
| `--notify`        | Posts output to Teams without saving (use with a section flag or `--story`) |
| `--post-comments` | Posts three types of automated comments to ADO stories: (1) a code review summary for each active branch containing a 4 or 5 digit story ID, (2) a merge summary when a PR linked to a story is merged, and (3) a new-branch notification when a branch is created. Combine with `--story <ID>` to post to a single story only. |
| `--save`          | Saves the full report to `reviews/daily-review-YYYY-MM-DD.txt` and posts to Teams |
| `--since <YYYY-MM-DD>` | Override the default 24-hour lookback window with a specific start date (e.g. `--since 2026-04-01`) |
| `--story <ID>`    | Deep dive on a single ADO story — shows full description, all comments, linked commits found in recent review files, and current state. When combined with `--post-comments`, restricts all comment posting to that one story only. |
| `--trend`         | Analyzes the last 30 saved daily review files and produces a trend report: BLOCKER/WARNING/INFO counts over time, most frequently flagged files, and pattern frequency rankings |
| `--verbose`       | Include the matched code snippet alongside each code review finding |

#### Example Commands

```cmd
REM Full daily review — save and post to Teams
node reviews\scripts\run-daily-review.js --save

REM Just the standup, print to console only
node reviews\scripts\run-daily-review.js --standup

REM Deep dive on story 9015 and post to Teams
node reviews\scripts\run-daily-review.js --story 9015 --notify

REM Post automated code review findings to ADO stories (dry run first)
node reviews\scripts\run-daily-review.js --post-comments --dry-run
node reviews\scripts\run-daily-review.js --post-comments

REM Post comments scoped to a single story only
node reviews\scripts\run-daily-review.js --post-comments --story 9100

REM Trend analysis over the last 30 saved reviews
node reviews\scripts\run-daily-review.js --trend

REM Review everything since the start of the sprint
node reviews\scripts\run-daily-review.js --synopsis --since 2026-03-18
```

---

### What It Produces

The report is divided into seven sections:

#### Section 1 — Daily Standup
A per-team-member breakdown of everything that happened in the last 24 hours: Git commits (with branch, hash, and message) and ADO story changes (with state and last-changed timestamp). This section is designed to support async standups — each person's activity is self-contained.

#### Section 2 — ADO Comments Rollup
Every comment posted to any work item in the configured area path within the lookback window, grouped by story. Includes the commenter's name, timestamp, and full comment text. Useful for catching decisions, blockers, and questions that were discussed in ADO but not in a standup.

#### Section 3 — Git → ADO Cross Reference
A hygiene check with four validations:
- **Commits with no linked story ID** — flags any commit message that doesn't contain a recognized story ID pattern (9xxx, 8xxx, 7xxx)
- **Referenced stories in unexpected state** — flags stories linked in commits that are in New or Removed state (indicates a commit may have been pushed against the wrong story)
- **Closed stories with no merged PR** — flags ADO stories marked Closed that have no corresponding merge commit in Git
- **Merged PRs with unresolved comments** — for each PR merged within the lookback window, fetches comment threads from ADO and flags any where one or more threads are still in `active` (unresolved) status. System-generated threads (merge conflict notices, policy alerts, vote notifications) are excluded — only developer-written comments are checked.

#### Section 4 — Full Daily Synopsis
The most information-dense section. Contains:

- **Git Activity** — total commits, merged PRs, new branches detected, per-PR diff summary (files changed, lines added/removed) with a plain-English summary of what each PR touched
  - **New branch warnings** — branches created within the lookback window are automatically scanned for four conditions: the linked story is already `Closed`, the story is still in `New` state (not yet Active), the branch name contains only a story ID with no description (e.g. `feature/dev/9085`), or the story's sprint has already ended. Warnings appear in the synopsis and are included in the ADO branch-created comment so the whole team is notified.
  - **Team-wide branch visibility** — the script runs `git fetch` before scanning so that branches pushed by other engineers within the lookback window are always detected, regardless of local sync state.
  - **Multi-Branch Stories** — detects stories with two or more active branches on remote. Branches older than `STALE_BRANCH_CRIT_DAYS` are excluded (already reported as stale in Section 6). Closed and Removed stories are excluded (covered by the closed-story branch warning). Flags stories where an old abandoned branch was never deleted, or where multiple developers are working in parallel without coordination.
  - **Large Commits** — any commit across all active branches where the files changed or total lines changed (added + removed) meets or exceeds the configured warn/critical thresholds. Flagged 🟡 or 🔴 depending on severity. Large commits are hard to code review thoroughly.
  - **File Hotspots** — files that appear in `FILE_HOTSPOT_THRESHOLD` or more separate merged PRs within the lookback window. Most useful when combined with `--since <sprint-start>` to cover the full sprint. Signals potential ownership confusion or unaddressed tech debt.
- **ADO Story Hygiene** — seven checks against all non-Closed stories in the current sprint. Findings are shown as a summary scorecard followed by a detail block for each check with at least one hit:
  - **No git activity** — story is `Active` but no commit in the lookback window references its ID
  - **No branch** — story is `Active` but no remote branch name contains its ID
  - **Unassigned** — story has no assignee
  - **Sprint spillover** — story was moved into this sprint from an earlier sprint (detected via ADO update history)
  - **Reopened** — story previously moved to `Closed` or `Resolved` and was subsequently moved back to `Active` or `New` within the last `STORY_HYGIENE_REOPENED_LOOKBACK_DAYS` days
  - **No acceptance criteria** — story is `Active` or `New` but its Acceptance Criteria field is empty or shorter than `STORY_HYGIENE_AC_MIN_LENGTH` characters (HTML stripped)
  - **No parent feature** — story has no Hierarchy-Reverse link to a parent Feature work item
- **Sprint Burndown** — see [Sprint Burndown](#sprint-burndown) below
- **Stuck Stories** — stories with no ADO activity beyond the configured thresholds, sorted by age with color-coded severity (💬 notice / 🟡 warning / 🔴 critical). Stories assigned to a future sprint (sprint start date has not yet arrived) are excluded — only current and past sprints are evaluated.
- **Stories by State** — all active stories in the area path organized by state (Closed, Active, SIT, UAT, New, On deck) with assignee and last-changed date

#### Section 5 — Automated Code Review
Scans the unified diff of every commit in the lookback window against 17 known anti-patterns. Findings are organized by severity. See [Automated Code Review Patterns](#automated-code-review-patterns) for the full pattern list.

#### Section 6 — All Branch Commit Activity
A table of every commit across all monitored repositories and all configured branches within the lookback window, showing repo, branch, timestamp, author, and hash. Followed by a stale branch report listing branches with no commits beyond the warn/critical thresholds.

#### Section 7 — Pipeline Status
All pipeline runs from the configured `PIPELINE_ADO_ORG`/`PIPELINE_ADO_PROJECT` in the last 24 hours, grouped by result (Succeeded, Failed, Partial). Each run shows the pipeline name, trigger type, branch, timestamp, and duration.

---

### Sprint Burndown

The burndown in Section 4 automatically detects the current sprint — no hardcoded sprint name is needed. It works by:

1. Querying a recently-modified active User Story in the area path to read its `System.IterationPath` and `System.IterationId`
2. Calling the ADO Classification Nodes API to fetch the sprint's start and finish dates
3. Querying all User Stories in that sprint by `IterationId`

**UAT counts as done.** Stories in UAT status are release-ready — they are waiting only for the Tuesday release night at the end of the sprint. The burndown treats `Closed + UAT` together as "done" for all rate and pace calculations. The progress bar uses two characters to distinguish them:

```
█ = Formally Closed
▓ = UAT (ready to ship on release night)
░ = Remaining (SIT / Active / New)
```

**Pace thresholds:**
- ✅ **On pace** — done% is within 5 points of sprint elapsed%
- 🟡 **Slightly behind** — done% is 6–20 points behind sprint elapsed%
- 🔴 **Behind pace** — done% is more than 20 points behind sprint elapsed%

The auto-detect picks up whichever sprint your active stories are currently assigned to. Once stories are moved to the next sprint in ADO, the burndown will reflect the new sprint automatically.

---

### Teams Integration

When `--save` is used (or `--notify` is used standalone), the report is posted to the Teams channel configured in `TEAMS_WEBHOOK_URL` as two separate messages to stay within Teams' message size limits:

- **Message 1** — Sections 1–5 (Standup, Comments, Cross Reference, Synopsis, Code Review)
- **Message 2** — Sections 6–7 (Branch Activity, Pipeline Status)

The webhook is a standard Microsoft Teams Incoming Webhook (configured via Power Automate or the Teams connector). If `TEAMS_WEBHOOK_URL` is blank, Teams posting is silently skipped.

---

## Supporting Module: `config.js`

`config.js` is not meant to be run directly. It is `require()`'d by every other script in this directory and is responsible for:

1. Locating and reading the `.env` file from the same directory
2. Parsing all environment variables
3. Exporting a single configuration object used by all scripts

The exported config object has the following top-level keys:

| Key             | Contents              |
|-----------------|---                    |
| `ado`           | org, project, projectId, team, iterationId, PAT, pre-built `authHeader` |
| `git`           | Array of repo configs (name, path, branches array) |
| `output`        | `reviewsDir` path for saving files |
| `lookbackHours` | Number of hours to look back (default: 24) |
| `staleBranch`   | `warnDays` and `critDays` thresholds |
| `stuckStory`    | `noticeDays`, `warnDays`, `critDays`, `areaPath` |
| `pipeline`      | Separate org/project/PAT/authHeader and `watchIds` array |
| `storyDeepDive` | `lookbackFiles` count for `--story` deep dives |
| `teams`         | `webhookUrl`          |
| `storyIdPattern` | Compiled `RegExp` for extracting story IDs from commit messages |

---

## Standalone Section Scripts

Each of the four section scripts can be run on its own. This is useful when you need a quick focused view without running the full orchestration.

### `standup.js`

Generates a per-person standup report from the last 24 hours of Git commits and ADO story changes. Groups output by team member name so each person's activity is self-contained. Useful for async standups or for reviewing what a specific team member shipped before a sync.

```cmd
node reviews\scripts\standup.js
node reviews\scripts\standup.js --save
```

**Data sources:** Git commit log (all configured repos/branches), ADO WIQL query for User Stories changed in the lookback window.

---

### `crossref.js`

Validates that Git and ADO are in sync. Runs three checks:
- Commits with no linked story ID (extracted using the `storyIdPattern` regex)
- ADO stories referenced in commits that are in an unexpected state
- ADO stories marked Closed with no corresponding merge commit in Git history

The goal is to catch hygiene issues early — commits pushed without a story reference, or stories closed in ADO that have no actual code merged.

```cmd
node reviews\scripts\crossref.js
node reviews\scripts\crossref.js --save
```

**Data sources:** Git commit log, ADO WIQL for closed stories.

---

### `synopsis.js`

Produces a full activity log: commits organized by branch with per-PR diff stats, ADO stories organized by state, and auto-generated observations. The saved file appends to an existing file if one already exists for today (allowing multiple runs per day to accumulate).

```cmd
node reviews\scripts\synopsis.js
node reviews\scripts\synopsis.js --save
```

**Data sources:** Git commit log, ADO WIQL for all User Stories changed in the lookback window with full descriptions and tags.

---

### `codereview.js`

Scans the unified diff of every commit in the lookback window against a library of 17 known anti-patterns. Findings are grouped by severity (BLOCKER → WARNING → INFO) and include the file name, line number, and pattern description. Use `--verbose` to also print the matched code snippet inline.

```cmd
node reviews\scripts\codereview.js
node reviews\scripts\codereview.js --save
node reviews\scripts\codereview.js --verbose
```

**Data sources:** Git diff output for all commits in the lookback window.

---

### Automated Code Review Patterns

`codereview.js` runs **19 checks** in total against every diff — 17 named single-line pattern tests plus 2 multi-line structural checks that require tracking state across consecutive lines. All 19 are derived from issues identified during manual code reviews in March 2026.

#### BLOCKERS — Must fix before merge

| Pattern | What It Catches |
|---|---|
| `CONSTRUCTOR_SUBSCRIPTION` | RxJS `.subscribe()` called inside a constructor — causes memory leaks |
| `LOAD_IN_CONSTRUCTOR` | API calls or data loading triggered from the constructor instead of `ngOnInit` |
| `DIV_INSIDE_SPAN` ⁺ | Block-level `<div>` nested inside an inline `<span>` — invalid HTML |
| `EMPTY_ERROR_HANDLER` | `.catch(() => {})` or empty error callbacks that silently swallow exceptions |
| `FRAME_ANCESTORS_REMOVED` | Deletion of `frame-ancestors` from CSP config — opens clickjacking vulnerability |
| `VAR_DECLARATION` | Use of `var` instead of `const`/`let` — function-scoped variable hoisting issues |

#### WARNINGS — Should fix before merge

| Pattern | What It Catches |
|---|---|
| `ARRAY_ISARRAY_CALLSITE` | Defensive `Array.isArray()` callsite used as a runtime type guard on a config or model value — indicates the type contract is missing at the source. Fix the type (e.g. `string | string[]`) instead of guarding at every call site. |
| `CSP_MISSING_HTTPS` | HTTP (non-HTTPS) URLs added to CSP directives |
| `SUBMIT_BUTTON_DISABLED_FLAG` | Submit buttons disabled via a boolean flag without a matching re-enable path |
| `NULL_GUARD_MISSING` | Accessing nested object properties without null/undefined guards |
| `REDUNDANT_TERNARY` | Ternary expressions that return `true`/`false` literally (just use the condition directly) |
| `ORPHANED_FLAG` | Feature flag variables set but never read, or read but never set |

#### INFO / NITS — Low priority, worth noting

| Pattern | What It Catches |
|---|---|
| `COMMENTED_CODE_BLOCK` ⁺ | Large blocks of commented-out code left in the diff |
| `INLINE_CHANGE_COMMENT` | `// TODO`, `// FIXME`, `// HACK` comments added in the diff |
| `CONSOLE_LOG` | `console.log()` statements left in production code paths |
| `MISALIGNED_BRACE` | Opening braces on their own line (style inconsistency with the rest of the codebase) |
| `MISSING_NEWLINE_EOF` | Files that do not end with a newline character |

#### ⁺ Multi-Line Structural Checks

Two checks above — marked with ⁺ — are **not** single-line regex tests. They run as separate structural passes after the main pattern loop and require tracking state across multiple consecutive diff lines. They cannot be expressed as a single regex because the finding depends on what appeared on previous lines.

**`DIV_INSIDE_SPAN`** (`.html` files only)
Tracks whether the scanner is currently inside an open `<span>` tag that has not yet been closed. If a `<div>` is found on any subsequent added line before the `</span>` is reached, the finding is raised. A single-line test would only catch cases where both tags appear on the same line.

**`COMMENTED_CODE_BLOCK`** (`.ts` and `.html` files only)
Counts consecutive added lines that begin with `//` or `<!--`. The finding only fires when 5 or more appear in a row — isolated comment lines are ignored. The streak resets as soon as a non-comment line is encountered. A single-line test has no way to know how many comment lines preceded the current one.

Both checks reuse the `PATTERNS` array only to retrieve the title and description text for the finding output. Their detection logic runs entirely outside the main pattern loop.

---

## `.env` Variable Reference

### Azure DevOps — Main

| Variable | Required | Default | Description |
|---|---|---|---|
| `AZURE_DEVOPS_PAT` | **Yes** | — | Personal Access Token for authentication |
| `AZURE_DEVOPS_ORG` | No | `https://dev.azure.com/PFCB-IT` | Organization URL |
| `AZURE_DEVOPS_PROJECT` | No | `PFC Systems (2026)` | Project name |
| `AZURE_DEVOPS_PROJECT_ID` | No | — | Project GUID (required for some API calls) |
| `AZURE_DEVOPS_TEAM` | No | `Enterprise_WebApps_Team` | Team name |
| `AZURE_DEVOPS_ITERATION_ID` | No | — | Current sprint GUID (used as fallback if auto-detect fails) |

### Git Repositories

Supports up to N repositories using numbered variable groups. Increment the number for each additional repo.

| Variable | Required | Default | Description |
|---|---|---|---|
| `GIT_REPO_N_NAME` | No | — | Display name shown in reports |
| `GIT_REPO_N_PATH` | **Yes** | — | Absolute path to the local repository |
| `GIT_REPO_N_BRANCHES` | No | `develop,master` | Comma-separated list of branches to monitor |

### Output

| Variable | Required | Default | Description |
|---|---|---|---|
| `REVIEWS_DIR` | No | `C:/Repos/Website 3.0/reviews` | Directory where all report files are saved |

### Analysis Thresholds

| Variable | Required | Default | Description |
|---|---|---|---|
| `LOOKBACK_HOURS` | No | `24` | Hours of history to include in all queries |
| `STALE_BRANCH_WARN_DAYS` | No | `14` | Days since last commit before a branch is flagged 🟡 |
| `STALE_BRANCH_CRIT_DAYS` | No | `30` | Days since last commit before a branch is flagged 🔴 |
| `STUCK_STORY_NOTICE_DAYS` | No | `3` | Days without ADO activity before a story is flagged 💬 |
| `STUCK_STORY_WARN_DAYS` | No | `5` | Days without ADO activity before a story is flagged 🟡 |
| `STUCK_STORY_CRIT_DAYS` | No | `10` | Days without ADO activity before a story is flagged 🔴 |
| `STUCK_STORY_AREA_PATH` | No | — | Scopes stuck story queries to a specific area path — prevents pulling stories assigned to other teams |
| `LARGE_COMMIT_FILES` | No | `10` | Files changed in a single commit before a 🟡 warning is raised |
| `LARGE_COMMIT_FILES_CRIT` | No | `20` | Files changed in a single commit before a 🔴 critical is raised |
| `LARGE_COMMIT_LINES` | No | `200` | Total lines changed (added + removed) in a single commit before a 🟡 warning |
| `LARGE_COMMIT_LINES_CRIT` | No | `250` | Total lines changed in a single commit before a 🔴 critical |
| `FILE_HOTSPOT_THRESHOLD` | No | `3` | Number of separate merged PRs a file must appear in to be flagged as a hotspot |

### ADO Story Hygiene

| Variable | Required | Default | Description |
|---|---|---|---|
| `STORY_HYGIENE_AC_MIN_LENGTH` | No | `50` | Minimum character length (HTML stripped) of the Acceptance Criteria field before a story is flagged as missing AC |
| `STORY_HYGIENE_REOPENED_LOOKBACK_DAYS` | No | `14` | How many days back to scan for stories that were reopened after being Closed or Resolved |

> **Performance note:** The spillover and reopened checks fetch the update history for every non-Closed sprint story (one API call per story). This is acceptable for typical sprint sizes (10–30 stories) but will be slower for large sprints. Disable by removing stories from the check or by extending the lookback to reduce the number of active stories detected.

### Pipeline Monitoring

| Variable | Required | Default | Description |
|---|---|---|---|
| `PIPELINE_ADO_ORG` | No | `https://dev.azure.com/pfcbweb` | ADO org where pipelines live (can differ from main org) |
| `PIPELINE_ADO_PROJECT` | No | `Automation_Website` | ADO project containing pipeline definitions |
| `PIPELINE_ADO_PAT` | No | Falls back to `AZURE_DEVOPS_PAT` | Separate PAT if pipeline org requires different credentials |
| `PIPELINE_WATCH_IDS` | No | *(all pipelines)* | Comma-separated pipeline definition IDs to monitor; leave blank to watch all |

### Story Deep Dive

| Variable | Required | Default | Description |
|---|---|---|---|
| `STORY_DEEP_DIVE_LOOKBACK_FILES` | No | `5` | Number of past daily review files to scan when building the `--story` commit history |

### Microsoft Teams

| Variable | Required | Default | Description |
|---|---|---|---|
| `TEAMS_WEBHOOK_URL` | No | — | Incoming webhook URL for the target Teams channel; if blank, Teams posting is skipped silently |

---

## Output Files

All files are written to the directory configured in `REVIEWS_DIR`.

| File | Generated By | Description |
|---|---|---|
| `daily-review-YYYY-MM-DD.txt` | `run-daily-review.js --save` | Full combined daily review (all 7 sections) |
| `standup-YYYY-MM-DD.txt` | `standup.js --save` | Per-person standup report |
| `crossref-YYYY-MM-DD.txt` | `crossref.js --save` | Git → ADO cross-reference validation |
| `git-synopsis-YYYY-MM-DD.txt` | `synopsis.js --save` | Full activity synopsis (appends on multiple runs) |
| `codereview-YYYY-MM-DD.txt` | `codereview.js --save` | Automated code review findings |

The `--trend` flag reads from `daily-review-*.txt` files in `REVIEWS_DIR` — it requires at least a few saved daily reviews to produce meaningful output.

---

## Extending the Scripts

### Add a New Code Review Pattern

Open `codereview.js` and add an entry to the `PATTERNS` array:

```javascript
{
  id:       'MY_PATTERN_NAME',
  severity: 'BLOCKER',           // BLOCKER | WARNING | INFO
  label:    'Short description of what this catches',
  test:     (line) => /your-regex/.test(line)
}
```

The `test` function receives one line of diff content at a time (already stripped of the leading `+`/`-` diff character). Return `true` to flag the line.

### Adjust Stuck Story / Stale Branch Sensitivity

All thresholds are controlled entirely through `.env` — no code changes needed. Lower the `_DAYS` values to get earlier warnings, raise them to reduce noise.

---

## Troubleshooting

| Symptom                     | Likely Cause | Fix  |
|-----------------------------|---           |---   |
| `HTTP 401` on any ADO call  | PAT expired or belongs to the wrong org | Regenerate the PAT in ADO User Settings → Personal Access Tokens. Verify `AZURE_DEVOPS_ORG` exactly matches the org the PAT was issued for. |
| ADO returns an HTML page instead of JSON (`Parse error` logged) | PAT has insufficient scopes | The PAT needs Work Items (Read & Write), Build (Read), and Code (Read). Recreate with those scopes and update `.env`. |
| WIQL returns 0 stories / burndown section is blank | `STUCK_STORY_AREA_PATH` is scoped too narrowly, or `AZURE_DEVOPS_PROJECT` doesn't match exactly | Copy the area path string directly from ADO → Project Settings → Team Configuration. Leave `STUCK_STORY_AREA_PATH` blank to scope to the entire project. |
| `[DEBUG] HTTP status: 400` from `adoPost` | Malformed JSON Patch body in a story creation script, or Teams message too large | Run with `--dry-run` first to preview output. Check the `[DEBUG]` lines printed to stderr for the raw response from ADO. |
| Teams webhook returns `HTTP 400` | Webhook URL is stale/revoked, or the message body exceeds Teams' size limit | Re-create the webhook in Teams (Apps → Incoming Webhook). The script already splits into two messages — if a single section is extremely large, run it standalone with `--save` to file instead. |
| `execSync` crash or `git: command not found` | Git is not on the system PATH, or a repo path with spaces isn't handled correctly | Verify `git` is accessible by opening a Command Prompt and running `git --version`. In `.env`, use forward slashes in paths (e.g. `C:/Repos/Website 3.0`). |
| `--post-comments` reports "no story IDs found in branch names" | Branch names don't contain a 4 or 5 digit number | Branch names must include the ADO story ID somewhere in the name — e.g. `feature/dev/9085-description`. Branches named `develop`, `master`, `hotfix`, etc. are intentionally skipped. |
| `--trend` reports "No saved daily reviews found" | `REVIEWS_DIR` is misconfigured, or `--save` has never been run | Verify `REVIEWS_DIR` in `.env` points to the correct folder. Run `node reviews\scripts\run-daily-review.js --save` at least once to generate the first file. |
| Sprint burndown is blank / missing from Section 4 | All stories in the sprint are Closed or Removed — the auto-detect query finds nothing to sample | Keep at least one story in Active or New state in the current sprint, or see [Known Limitations](#known-limitations) for the fallback option. |

---

## Architecture

The diagram below shows how data flows from external systems through the scripts to their outputs.

```
  .env
   │
   └──► config.js  ◄──────────────────────────────────────────────────────────┐
              │                                                                │
              ▼                                                                │
   run-daily-review.js  ◄──── all standalone scripts also require config.js ──┘
         │         │
         │         └──────────────────────────────────────────┐
         │                                                     │
    Reads from                                           Writes to
         │                                                     │
    ┌────┴────────────────────┐              ┌────────────────────────────────┐
    │  Azure DevOps REST API  │              │  reviews\  (REVIEWS_DIR)       │
    │  ─ Work Items (WIQL)    │              │  ─ daily-review-YYYY-MM-DD.txt │
    │  ─ Comments             │              │  ─ standup-YYYY-MM-DD.txt      │
    │  ─ Iterations/Sprints   │              │  ─ crossref-YYYY-MM-DD.txt     │
    │  ─ Pull Requests/Threads│              │  ─ git-synopsis-YYYY-MM-DD.txt │
    │  ─ Pipeline Builds      │              │  ─ codereview-YYYY-MM-DD.txt   │
    ├─────────────────────────┤              │  ─ codereview-YYYY-MM-DD.txt   │
    │  Git (local via CMD)    │              └────────────────────────────────┘
    │  ─ git log              │
    │  ─ git diff             │              ┌────────────────────────────────┐
    │  ─ git fetch            │              │  Microsoft Teams               │
    │  ─ git fetch            │              │  (new branch scan)             │
    │  ─ git for-each-ref     │              │  ─ Message 1: Sections 1-5     │
    └────────────────────────-┘              │  ─ Message 2: Sections 6-7     │
                                             └────────────────────────────────┘
```

**Report sections produced by `run-daily-review.js`:**

```
  Section 1 — Daily Standup       ◄── Git log  +  ADO Work Items
  Section 2 — ADO Comments        ◄── ADO Work Item Comments
  Section 3 — Git→ADO Cross Ref   ◄── Git log  +  ADO Work Items
  Section 4 — Synopsis/Burndown   ◄── Git log  +  ADO Work Items  +  ADO Iterations
  Section 5 — Code Review         ◄── Git diff
  Section 6 — Branch Activity     ◄── Git log  +  Git branch
  Section 7 — Pipeline Status     ◄── ADO Pipeline Builds API
  Section 8 — Trend (--trend)     ◄── Saved daily-review-*.txt files (no API calls)
```

The `--trend` flag is the only section that makes no network calls — it reads exclusively from previously saved `.txt` files in `REVIEWS_DIR`.

---

## Sample Output

The examples below are excerpted from a real daily review. Author names have been generalized.

---

**Section 1 — Daily Standup (one team member block)**

```
  Dev A  <dev-a@example.com>
  ───────────────────────────────────────────────────────────────────────────
  GIT COMMITS:
    [a1db267d] reverted loyalty checkbox update for /reservation and /waitlist
               feature/dev-a/9015changes | Apr 6, 8:16 AM

  ADO STORIES:
    [9015] (SIT) Replace Eclub Opt-ins with Loyalty Opt-ins
               Last changed: Apr 6, 8:16 AM
    [9086] (UAT) Dashboard Update Mobile Phone field.
               Last changed: Apr 6, 5:37 AM
```

---

**Section 4 — Sprint Burndown**

```
  SPRINT BURNDOWN — Sprint Seven  (Mar 18, 2026 → Mar 31, 2026)
  ---------------------------------------------------------------------------
  Day 13 of 13  |  0 days remaining  |  100% through sprint

  Stories       :  13 total in sprint
    ✅ Closed       :    5  (38%)  ████████▓▓▓▓▓▓▓▓▓░░░░░  (█=closed ▓=UAT)
    🔵 UAT         :    5  (38%)  ← ready to ship (release night)
    🟡 SIT         :    2  (15%)
    🔴 Active      :    1  ( 8%)
    ────────────────────────────────────────
       Done        :   10  (77%)  ████████▓▓▓▓▓▓▓▓▓░░░░░

  Burn rate     :  0.77 stories/day  (10 done [5 closed + 5 UAT] over 13 days)
  To finish     :  3 stories remaining,  need 1.0/day over 0 days
  Status        :  🔴 Behind pace — 10 done [5 closed + 5 UAT] over 13 days
```

---

**Section 5 — Automated Code Review (one BLOCKER, one WARNING)**

```
  ● BLOCKER  [LOAD_IN_CONSTRUCTOR]
    quick-enrollment.component.ts  line 104
    API calls or data loading triggered from the constructor instead of ngOnInit

  ● WARNING  [NULL_GUARD_MISSING]
    reservations.component.ts  line 312
    Accessing nested object properties without null/undefined guards
```

---

**Section 6 — Stale Branches**

```
  STALE BRANCHES  (no commits in 14+ days)

  🔴  223 days   pfcbconductor   feature/dev/8328-conductor-changes
               Last commit : Dev B  (223 days ago)
               Tip         : [8cab2ae] added hcaptcha verification

  🟡   21 days   pfcbconductor   feature/dev/9013-Story
               Last commit : Dev A  (21 days ago)
               Tip         : [c034c76] 9013 Story Implementation
```

---

**Section 8 — Trend (`--trend`)**

```
  Date         Files   Blockers  Warnings  Nits  Total  Findings
  -----------  ------  --------  --------  ----  -----  --------------------
  Apr 1            12  🔴 2      🟡 3         1      6  ██████████
  Apr 2             9  0         🟡 1         2      3  █████
  Apr 3            14  🔴 1      0            0      1  ██
  Apr 4             0  0         0            0      0  ✓ Clean
  Apr 6            11  🔴 1      🟡 2         1      4  ███████

  5-day totals  :  🔴 4 blockers  |  🟡 6 warnings  |  💭 4 nits
  Daily average :  2.8 findings/day
  Clean days    :  1 of 5  (20%)
  🔴 Trend      :  Worsening  (avg 1.8 → 4.0 over last 2 days)
```

---

## Scheduling

### Windows Task Scheduler

To run the daily review automatically each morning, create a scheduled task using Task Scheduler:

1. Open **Task Scheduler** (search for it in the Start menu)
2. Click **Create Basic Task** in the right panel
3. Name it `Team Daily Review` and click Next
4. Set the trigger to **Daily**, start time `8:00 AM`, click Next
5. Set the action to **Start a Program**, click Next
6. Set the fields as follows:

| Field          | Value    |
|----------------|--------- |
| Program/script | `node`   |
| Add arguments  | `"C:\Repos\Team\reviews\scripts\run-daily-review.js" --save` |
| Start in       | `C:\Repos\Team` |

7. Click Finish, then right-click the task and choose **Run** to verify it works before leaving it scheduled.

> **Note:** `node` must be on the system PATH for Task Scheduler to find it. If the task fails with "program not found," use the full path to `node.exe` (e.g. `C:\Program Files\nodejs\node.exe`) in the Program/script field.

---

### Monday Morning — Catching the Weekend Gap

The default 24-hour lookback window will miss Saturday and Sunday activity when run on Monday morning. Run this after the normal daily review to catch up:

```cmd
REM Monday catch-up — review from Friday morning through now
node reviews\scripts\run-daily-review.js --synopsis --since 2026-04-04

REM Or capture it all and save as a separate file
node reviews\scripts\run-daily-review.js --since 2026-04-04 --save
```

Replace the date with the most recent Friday. The `--since` flag overrides the lookback window for any section flag or the full `--save` run.

---

## Known Limitations

1. **Sprint burndown and story hygiene checks break when all sprint stories are Closed or Removed.**
   Both features auto-detect the current sprint by sampling a recently-changed active User Story. If every story in the sprint has been formally closed, the sample returns empty and both sections are silently omitted. To work around this, keep at least one story in Active or New state until the sprint is officially retired, or set `AZURE_DEVOPS_ITERATION_ID` in `.env` to the sprint's GUID as a hardcoded fallback.

2. **The 24-hour lookback misses weekends on Mondays.**
   Monday's standard run covers only the past 24 hours, leaving Saturday and Sunday invisible. Use `--since YYYY-MM-DD` with the previous Friday's date to cover the full weekend. See [Monday Morning — Catching the Weekend Gap](#monday-morning--catching-the-weekend-gap) above.

3. **`--post-comments` requires a story ID in the branch name.**
   Branches named `develop`, `master`, `hotfix`, or anything without a 4 or 5 digit number are intentionally skipped. If a developer creates a branch without the story ID in the name (e.g. `feature/swetha/login-fix` instead of `feature/swetha/9015-login-fix`), those commits will not be scanned or posted. Branch naming convention must be enforced at PR review time.

   The **multi-branch story check** in Section 4 also relies on story IDs in branch names — branches with no story ID are invisible to that check. Additionally, branches older than `STALE_BRANCH_CRIT_DAYS` (default 30 days) are excluded from the multi-branch check since they are already flagged in the Section 6 stale branch report.

4. **Teams message size cap can cause HTTP 400 on large code review days.**
   The script already splits the report into two Teams messages to stay under the size limit. However, on days with a large number of code review findings across many files, message 1 (which includes Section 5) can still be rejected. If this happens, run `--codereview --save` separately to write findings to file, and use `--save` for the rest of the report.

5. **`--trend` requires previously saved daily review files — it makes no API calls.**
   The trend section is built entirely by parsing `daily-review-*.txt` files from `REVIEWS_DIR`. If `--save` was never used, or the reviews folder was cleared, `--trend` will report "No saved daily reviews found." There is no way to backfill historical trend data — it accumulates only from future `--save` runs.

6. **Multi-repo Git queries run sequentially, not in parallel.**
   Each configured repository runs its own `git log` and `git diff` via a separate synchronous command. On machines where repositories are on a network drive, or repos are very large, this can add several seconds to the total runtime. This is a deliberate simplicity tradeoff — the scripts use no async concurrency for Git operations.

7. **Story ID pattern is fixed to 4+ digit IDs in the 7xxx–9xxx range.**
   The `storyIdPattern` regex in `config.js` is designed to match the current ADO story numbering. If story numbers roll into a new range (e.g. 10000+) or a different project with IDs below 7000 is added, the pattern must be updated in `config.js`. The regex also appears in `codereview.js` and `crossref.js` — search for `storyIdPattern` to find all usages.

---

## `--post-comments` Walkthrough

This flag automates posting three types of structured comments directly to ADO stories:

| Comment Type | When It Posts |
|---|---|
| **New Branch** | A branch containing a 4 or 5 digit story ID was created within the lookback window |
| **PR Merge** | A pull request linked to a story was merged within the lookback window |
| **Code Review** | An active branch containing a 4 or 5 digit story ID has code review findings in its recent commits |

All three types run on every `--post-comments` invocation. Use `--story <ID>` to restrict all three to a single story (see [Scoping to One Story](#scoping-to-one-story) below).

It is most useful before a sprint release to give developers a summary of any issues found in their branch's recent commits.

### Step 1 — Branch Naming Convention

The feature works by extracting story IDs from branch names. Any branch with a 4 or 5 digit number anywhere in its name is included. The recommended naming pattern is:

```
feature/<author>/<story-id>-short-description
```

Examples that **will** be picked up:
```
feature/dev-a/9085-sms-unsubscribe
feature/swetha/9015changes
Feature/Sai/9085-StoryImplementation
```

Examples that **will not** be picked up (no story ID):
```
develop
master
feature/swetha/login-fix
hotfix-button-alignment
```

> **Branch naming warning:** If a branch name contains only a story ID with no description after it (e.g. `feature/swetha/9085`), a `no-description` warning is raised in Section 4 of the report. The recommended convention is `feature/<author>/<story-id>-brief-description`.

### Step 2 — Dry Run First

Always run with `--dry-run` before posting to see exactly what would be submitted to each story:

```cmd
node reviews\scripts\run-daily-review.js --post-comments --dry-run
```

The dry-run output shows each story ID found, which branches map to it, and the full text of the comment that would be posted:

```
  --post-comments (DRY RUN)
  ─────────────────────────────────────────────────────────────────────────
  Story #9085  ←  [Website 3.0] Feature/Sai/9085-StoryImplementation

  Would post comment:
  ──────────────────
  Automated Code Review — Apr 6, 2026

  Branch: Feature/Sai/9085-StoryImplementation
  Commits scanned: 3  |  Files reviewed: 7

  WARNINGS (1)
  • NULL_GUARD_MISSING  —  reservations.component.ts line 88
    Accessing nested object properties without null/undefined guards

  INFO (1)
  • CONSOLE_LOG  —  sign-up.component.ts line 204
    console.log() statement in production code path
  ─────────────────────────────────────────────────────────────────────────
  1 story would be updated. Run without --dry-run to post.
```

### Step 3 — Live Run

Once the dry-run output looks correct, run without `--dry-run`:

```cmd
node reviews\scripts\run-daily-review.js --post-comments
```

Each story that receives a comment will show a confirmation line:

```
  ✓ Posted to #9085  (1 warning, 1 info)
  --post-comments: 1 of 1 stories updated.
```

The comment appears in the ADO story's Discussion tab and is visible to everyone on the team.

After all posting is complete, a POST-COMMENTS SUMMARY table is printed to the console (not posted to ADO):

```
  ╔══════════════════════════╦════════╦═════════╦════════╗
  ║ Type                     ║ Posted ║ Skipped ║ Errors ║
  ╠══════════════════════════╬════════╬═════════╬════════╣
  ║ PR Merge                 ║      1 ║       0 ║      0 ║
  ║ Code Review              ║      2 ║       1 ║      0 ║
  ║ New Branch               ║      1 ║       0 ║      0 ║
  ╠══════════════════════════╬════════╬═════════╬════════╣
  ║ TOTAL                    ║      4 ║       1 ║      0 ║
  ╚══════════════════════════╩════════╩═════════╩════════╝
```

"Skipped" means a story was found but had no findings or no matching activity — nothing to post. "Errors" means the ADO API call failed for that story.

### Step 4 — Scoping to One Story

To post comments for a single story only (useful when reviewing a specific branch or re-posting after a fix), combine `--post-comments` with `--story`:

```cmd
node reviews\scripts\run-daily-review.js --post-comments --story 9100
```

All three comment types (new branch, PR merge, code review) are still evaluated — but only activity linked to story `9100` will be posted. The POST-COMMENTS SUMMARY at the end reflects only that story's results.

### Step 5 — When Nothing Is Posted

There are cases where `--post-comments` skips posting for a given comment type:

| Message | Meaning |
|---|---|
| `no story IDs found in branch names` | No active branches have a 4 or 5 digit number in their name — check branch naming convention |
| `no findings on any story branch` | Branches were found but the code review scanner found no issues in their recent commits |
| `no merged PRs with linked story IDs` | No merged PRs with story IDs were found in the lookback window |
| `no new branches detected` | No branches with a 4 or 5 digit story ID were created within the lookback window |

---

## `create-story.js` — ADO Work Item Creator

`stories/create-story.js` is a standalone helper that creates a User Story (or any work item type) in Azure DevOps from a JSON definition file. It is used to create tech debt stories, bug reports, and feature requests directly from the command line without opening the ADO browser UI.

### Usage

```cmd
REM Preview what would be created (no API call made)
node reviews\scripts\stories\create-story.js --file reviews\scripts\stories\my-story.json --dry-run

REM Create the work item in ADO
node reviews\scripts\stories\create-story.js --file reviews\scripts\stories\my-story.json
```

Run from the repository root. The `--file` path can be relative or absolute.

### JSON Schema

Create a `.json` file in `reviews/scripts/stories/` using the schema below. Only `title` is required — all other fields fall back to defaults from `config.js` or `.env`.

```json
{
  "title":              "Story title (required)",
  "description":        "<h2>HTML description...</h2>",
  "acceptanceCriteria": "<h3>Acceptance Criteria...</h3>",
  "assignedTo":         "Brian Hackett",
  "iterationPath":      "PFC Systems (2026)\\Sprint Eight",
  "areaPath":           "PFC Systems (2026)\\Enterprise_WebApps_Team",
  "tags":               "tech-debt; angular; typescript",
  "parentFeatureId":    9099,
  "workItemType":       "User Story",
  "techDebtSeverity":   "Warning",
  "requirementLinks": [
    { "url": "https://...", "comment": "Link label shown in ADO" }
  ]
}
```

### Field Reference

| Field | ADO Field | Notes |
|---|---|---|
| `title` | `System.Title` | **Required.** Plain text. |
| `description` | `System.Description` | HTML. Use `<h2>`, `<h3>`, `<pre><code>`, `<ul>` etc. |
| `acceptanceCriteria` | `Microsoft.VSTS.Common.AcceptanceCriteria` | HTML. Use GIVEN/WHEN/THEN format per team standard. |
| `assignedTo` | `System.AssignedTo` | Display name. Default: `STORY_ASSIGNED_TO` env var → `Brian Hackett`. |
| `iterationPath` | `System.IterationPath` | Full path e.g. `PFC Systems (2026)\Web App Team Sprints\Sprint Eight`. |
| `areaPath` | `System.AreaPath` | Defaults to `<project>\<team>` from config. |
| `tags` | `System.Tags` | Semicolon-separated. e.g. `tech-debt; angular`. |
| `parentFeatureId` | Hierarchy-Reverse relation | Links story to a parent Feature. Default: `CREATE_STORY_DEFAULT_PARENT_ID` env var → Feature `9099`. |
| `workItemType` | URL parameter | Defaults to `User Story`. Can be `Bug`, `Task`, etc. |
| `techDebtSeverity` | `Custom.TechDebtSeverity` | One of: `Critical`, `Warning`, `Architectural Need`, `Nitpick`. See below. |
| `requirementLinks` | `Custom.RequirementsLink` | Array of `{ url, comment }` objects — rendered as an HTML list in the Requirements Link field visible on the story form. |

### Tech Debt Severity Values

| Value | When to Use |
|---|---|
| `Critical` | Causes bugs, data loss, or a security risk — needs immediate action |
| `Warning` | Code works but has a bad pattern or inconsistent type contract — should be fixed soon |
| `Architectural Need` | Structural issue affecting maintainability or scalability |
| `Nitpick` | Minor style or convention issue — low urgency |

### `.env` Defaults for Story Creation

These optional environment variables set defaults for stories created with this script:

```env
STORY_ASSIGNED_TO=Brian Hackett
STORY_ITERATION_PATH=PFC Systems (2026)\Web App Team Sprints\Sprint Eight
CREATE_STORY_DEFAULT_PARENT_ID=9099
```

If a value is set in the JSON file it always overrides the environment variable default.
