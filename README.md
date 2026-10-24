# agile-reports

Agile reporting built on the [`gh-agile`](https://github.com/paulcaron16k/gh-agile) export.

## What this repository holds

`gh-agile` is the Agile tool: boards, sprints and issues on GitHub, and an export of them. This
repository is what is done with that export — **KPIs**, and the model they are expressed in.

- [`GOLD_MODEL.md`](GOLD_MODEL.md) — a generic Agile metrics model, `agile_issues` and
  `agile_events`, and how `gh-agile`'s silver export fills it. Generic so that a measure taken on
  GitHub can be set beside the same measure from another Agile tool, such as Jira.

The boundary runs both ways: `gh-agile` writes no `agile_*` file, and nothing here writes to GitHub.
The export's contract is `gh-agile`'s
[`github_catalog.json`](https://github.com/paulcaron16k/gh-agile/blob/main/github_catalog.json).

## This repository has no project of its own

Its issues live on **`paulcaron16k/projects/4`**, the same board as `gh-agile`, under the key
**`REPORTS`**. That is the point rather than an accident: one team responsible for several
repositories sees all of their work on one scrum board and one kanban board.

`.github/agile.json` holds a single property:

```json
{ "config_from": "paulcaron16k/gh-agile" }
```

`read_config` follows that pointer and resolves the real profile — the project number and owner, the
keys, the boards, the sprint cadence — through the API, so it works for a private repository and
cannot rot into a dead link the way a relative path would. Nothing about the board is duplicated
here, so nothing here can disagree with it.

## Working in it

Run `gh agile` from this clone and the board it reaches is `#4`:

```bash
gh agile board show "Sprint board"     # the whole team, every repository
gh agile board show "Team: REPORTS"    # this repository's slice
gh agile epic "..."                    # filed here, boarded there, titled REPORTS-<n>
gh agile validate
```

`gh agile sync` from *any* of the team's clones sweeps every repository in the profile's keys, which
is how issues opened in the web UI reach the shared board.

## Why the key is on the repository

GitHub numbers issues per repository, so two repositories on one board both have a `#12`. The key
disambiguates: `REPORTS-12` and `AGILE-12` are different issues, and either can be said out loud.
`project.issue_title_prefix` writes that identifier into the title, which is what makes a shared
board readable at a glance.
