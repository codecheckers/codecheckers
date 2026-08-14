# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

There is no code, build, or test suite here. This is the "database" of the [CODECHECK](https://codecheck.org.uk/) community: a single CSV of people who have volunteered to conduct CODECHECKs, plus the GitHub issue template used to onboard them. All work is data curation and issue handling.

## The data files

`codecheckers.csv` — the volunteer list, columns: `name,handle,ORCID,contact,fields,languages`

- Rows are append-only in registration order; do not sort or reorder.
- `handle` is the GitHub handle with a leading `@` (a few historic rows are missing it — leave them alone unless fixing that row).
- `ORCID` is the bare ID, not a URL.
- `contact` is either an email address or the literal string `see ORCID page` (only valid if that profile actually exposes a public email — see the second snippet in `not-a-bot.md` for the reply when it doesn't).
- `fields` and `languages` are comma-separated lists inside one CSV field, so they must be double-quoted whenever they contain a comma. Per the README, entries within `languages` are ordered most-to-least proficient.

`institutional-codecheckers.csv` — columns: `handle,institution`

People who codecheck as part of their job rather than as volunteers. They are org/team members without a volunteer registration issue, and this file documents why. Same `@handle` convention; no ORCID or contact is collected, since the institution is the point of contact. Overlap with `codecheckers.csv` is allowed and expected (e.g. `@yiquintero` is in both). Do not onboard someone here via the volunteer registration workflow, and do not silently move a row between the two files — ask.

[`annual-maintenance.md`](annual-maintenance.md) holds the yearly upkeep checklist (renamed handles, dead `see ORCID page` contacts, list/team reconciliation) with runnable scripts; follow it when asked to check or clean up the lists, and record each run in its table.

When auditing team membership against the lists, check **both** files before reporting someone as missing, and note that `@team-codecheck` is the org's bot account. The `institutional-codecheckers` GitHub team is the closest thing to an authoritative source for the institutional list; `check-pub` is a separate team for the CHECK-PUB project and does not imply either kind of membership.

## Registration workflow

New codecheckers open an issue via `.github/ISSUE_TEMPLATE/codechecker-registration.md`, which asks them to paste their row as CSV. The community manager (`nuest`) then: verifies the data, invites the person to the GitHub `codecheckers` team, replies with the welcome snippet from `not-a-bot.md`, and appends the row.

**Never post a comment or any other message others will read until the full draft has been shown to the user and approved** — no exceptions, not even when told to "handle it" or "run the sequence", and not because a near-identical message was approved earlier. Invitations, CSV edits and commits do not need that pause; text other people will see always does.

**When asked to process a registration issue, follow [`registration-runbook.md`](registration-runbook.md)** — it has the exact `gh api` calls, the ORCID email verification, the validation checks, the message templates with their observed variants, and the points at which to stop and ask the human.

Commits that add a codechecker follow the established message form, which closes the issue:

```
add @handle, closes #NN
```

Take the row content from the issue but normalize it to the column conventions above rather than pasting verbatim — submissions frequently omit the `@`, include an ORCID URL, or leave quoting off multi-value fields.
