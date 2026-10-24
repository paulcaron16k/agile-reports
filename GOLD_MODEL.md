# The Gold Agile model

A generic model for Agile metrics: one shape for issues and their history, whatever tool they were
tracked in, so that cycle time, lead time, throughput and flow measured on GitHub can be set beside the
same measures from another Agile tool, such as Jira. Each tool is a *source*. This document defines the
model's two tables, `agile_issues` and `agile_events`, and maps the
[`gh-agile`](https://github.com/paulcaron16k/gh-agile) export onto them.

Every claim about GitHub below was measured against a live project, not read from documentation.

---

## Silver in, gold out

`gh-agile` exports **silver**: `github_issues`, `github_events` and `github_prs`, which are GitHub's
facts under GitHub's names. Its contract is
[`github_catalog.json`](https://github.com/paulcaron16k/gh-agile/blob/main/github_catalog.json) —
every stream, every column, typed and nullable-or-not — and it is the input a transform is entitled to
assume. `gh-agile` writes no `agile_*` file, and this repository writes nothing back to GitHub.

**Gold** is this model. Getting from one to the other is a transform, not a rename, and it does five
things:

1. **Renames.** GitHub's vocabulary is not the model's: `number` is `process_id`, `created_at` is
   `created_date`. The events grain was built close to the model's vocabulary; the issues grain was not.
2. **Converts units.** Every duration in silver is an integer count of seconds. A gold field states its
   unit, and a transform that skips the conversion does not fail — a factor of 3,600 between seconds
   and hours simply reads as a wrong number on a dashboard.
3. **Derives.** A stable `process_guid`, and the time-in-status measures, from the status events.
4. **Resolves people and teams.** Silver carries GitHub logins. Turning a login into a person's
   identifier, and placing an issue in a team, needs a directory the transform owns: GitHub has no
   team edge on an issue.
5. **Leaves what GitHub cannot supply null.** A field filled with a guess reads, downstream, exactly
   like a measured one.

---

## `agile_events`

One row per thing that happened to an issue. GitHub's equivalent of a changelog is the issue
**timeline** — some fifty distinct typed events rather than one stream — and the status event carries
more than a changelog usually does:

```
AddedToProjectV2Event            2026-08-14T16:58:59Z
ProjectV2ItemStatusChangedEvent  2026-08-14T16:59:00Z  (none) -> Todo
LabeledEvent
ProjectV2ItemStatusChangedEvent  2026-08-14T16:59:32Z  Todo -> In Progress
ClosedEvent
```

`ProjectV2ItemStatusChangedEvent` has `previousStatus`, `status`, `actor`, `createdAt` **and
`wasAutomated`**, which tells a person moving a card from a rule doing it.

| Field | Meaning | From GitHub |
|---|---|---|
| `occurred_at` | When it happened, UTC | `createdAt` on the timeline event |
| `event_category` | `state_change`, `activity` or `data_change` | See the three rows below |
| `state`, `prior_state` | The status after and before a `state_change` | `status`, `previousStatus` |
| `state_changed` | Whether the status actually moved | `status != previousStatus` |
| `actor_name`, `ext_actor_id` | Who did it, in the source tool's terms | `actor.login` |
| — `state_change` | | `ProjectV2ItemStatusChangedEvent` |
| — `activity` | | `AssignedEvent`, `LabeledEvent`, `MilestonedEvent`, `ClosedEvent`, `ReopenedEvent`, `SubIssueAddedEvent`, `CrossReferencedEvent`, and ~40 more |
| — `data_change`, with `data_from` / `data_to` | A field's old and new value | `IssueFieldChangedEvent` for GitHub's issue fields only |

**`data_change` is the one gap.** A change to a Projects v2 *custom* field — points, sprint, priority
— leaves no event at all, so GitHub supplies no `data_from` / `data_to` for them. `gh-agile` documents
the gap and the mitigation, dated snapshots diffed against each other, whose resolution is the snapshot
interval. A row synthesised that way should say so; it is not an exact timestamp.

| Field kind | Change history from GitHub |
|---|---|
| Status | **Yes** — a dedicated event, with the previous value |
| Assignee, labels, milestone, title, state, parent | **Yes** — one typed event each |
| Projects v2 custom fields | **No** — from snapshots, at their resolution |

---

## `agile_issues`

One row per work item.

| Field | Meaning | From GitHub | Notes |
|---|---|---|---|
| `ext_process_id` | The source tool's identifier | `Issue.id` (node ID) | Stable across renames and transfers |
| `issue_key` | The identifier people say out loud, `KEY-number` | `issue_key` in silver | GitHub has no key; `gh-agile` keeps one per repository. Falls back to `owner/repo#number` |
| `state_id`, `state_name` | Where the item is in its workflow | Project `Status` | Map onto `new\|ready\|in_progress\|in_review\|in_qa\|blocked\|done` |
| `issue_type_id` | Epic, story, bug, … | `type:` label | |
| `created_date`, `updated_date`, `closed_date` | | `createdAt`, `updatedAt`, `closedAt` | `updated_date` is the replication key |
| `project_id` | The board | Project number + owner | A number alone is not an identity |
| `epic_id`, `parent_id` | The item above this one | `Issue.parent.id` | GitHub has one parent edge, not two |
| `assignee_ext_id` | Who holds it | first of `assignees` | The model expects one and GitHub allows many; keep the rest in `data` |
| `reporter_ext_id` | Who filed it | `Issue.author.login` | |
| `story_points` | The estimate | Project number field | |
| `priority_id` | | Project single-select | |
| `labels` | | `labels.nodes[].name` | |
| `sprint_id` | The iteration it is in | Project iteration field | |
| `subtask_count` | Direct children | `subIssuesSummary.total` | |
| `lead_time` | Boarded → done | `AddedToProjectV2Event` → done | **Not** created → done: an issue open a year and boarded yesterday has one day of lead time by this definition. Say so wherever it is reported |
| `cycle_time` | Work started → done | first `In Progress` → `Done` | |
| `blocked_time`, `review_time`, `qa_time` | Time in each status | Sum of intervals per status | Each status event carries its previous value, so intervals are exact |
| `status_change_count`, `reopen_count`, `was_blocked` | | Counted from status events | |
| `data` | Everything unmapped | | Extra assignees, any custom field |

**Unfillable from GitHub, and left null:** `business_value`, `severity_id`, `released_date` and anything
tied to a release — GitHub has no release object attached to an issue the way some trackers attach a
fix version. `team_id` needs the transform's own directory (step 4 above).

---

## Initiatives are an attribute, not a level

`gh-agile` has a level above epic — `type:initiative`, with epics as its sub-issues. The model does
**not** gain a fourth hierarchy level for it. An initiative arrives as an attribute on every issue
beneath it:

1. **An attribute is additive; a level is a schema change** every consumer of `agile_issues` absorbs.
2. **It answers the questions anyway.** Burndown, cycle-time distributions and scope reports group
   by initiative; none of them traverses a hierarchy.
3. **Nothing is lost.** The initiative itself is an ordinary silver row, and `parent_key` records the
   edge, so the level can be built later from data already exported.

| Silver | Gold | Note |
|---|---|---|
| `initiative_slug` | an attribute on the issue | The join key. Group by it |
| `parent_key` | the parent edge, any level | A key, so it joins to `issue_key` in one column |
| `parent_type` | which level the parent is | `initiative` above an epic, `epic` above a story |
| `epic_key` | the epic a story belongs to | Empty when the parent is not an epic |
| `epic_number` | — | **Prefer `epic_key`.** GitHub numbers issues per repository, so two repositories on one board both have a `#12` |

That last row applies to the model as well: **reference a parent by key, never by number**, or a board
spanning repositories makes the reference ambiguous.

### Progress is aggregated, not copied

Silver carries `sub_issues_done` over `sub_issues_total` per container, because that is what GitHub
counts and what boards render. Gold has no corresponding column. "How far along is this initiative" is
a `GROUP BY` over `agile_issues`, filtered by `initiative_slug` or joined on `parent_key`. A copied
column would be a second definition of the same number, and the two would disagree the first time an
issue closed between two exports.

**Do not sum `sub_issues_done` across levels.** It counts direct children only, so adding an
initiative's tally to its epics' tallies produces a figure that reconciles to neither.

---

## Tasks are below the model

`gh-agile` allows `type:task` below a story, for story-internal housekeeping filed as issues. **No task
reaches this model, because none reaches the export.** A task carries no points, no sprint and no
board, and is counted by nothing that counts stories, so a row for one would change no metric. A grain
exists to be measured.

This is a boundary, not an omission. The tasks are still in GitHub, and the decision reverses without
rewriting anything built on it.
