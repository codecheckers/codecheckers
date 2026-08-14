# Codechecker registration runbook

Operational companion to the "Sign up" section of [`README.md`](README.md) and the message snippets in
[`not-a-bot.md`](not-a-bot.md). It describes how to process a registration issue end to end with the
`gh` CLI, and which steps require explicit human approval.

Everything below was verified against the live API in the `codecheckers/codecheckers` repo.

**Supervision rule:** every read/validation step may run unattended. Every step that writes — team
invitation, issue comment, commit/push, issue close — is proposed to the human first and only executed
after explicit approval.

> **Strict rule, no exceptions: never post a comment or any other message other people will read until
> the full draft has been shown to the community manager and approved.** This holds even when the
> instruction is "handle the registration" or "run the sequence", and even when a near-identical
> message was approved minutes earlier — approval of one message is never approval of the next.
> Render the complete text, stop, and wait. Batch the drafts if several are due, and call out anything
> in them that involved a judgement call. The other steps of the workflow do not need this pause.

## 0. Preconditions

```bash
gh auth status                                   # must be logged in as an org owner (e.g. nuest)
gh api -i user 2>&1 | grep -i x-oauth-scopes     # must include admin:org (or write:org)
```

Read-only work needs `repo, read:org`. The docs suggest team-membership *writes* need `admin:org`, but
in practice the `gh` OAuth token with `gist, read:org, repo, workflow` **did** succeed at
`PUT …/teams/…/memberships/…` when the account is an organization owner (verified 2026-08-14). If you
do get a 403, widen the scopes once, interactively — this is a browser flow, so the human must run it,
e.g. by typing `! gh auth refresh -h github.com -s admin:org` in the session:

```bash
gh auth refresh -h github.com -s admin:org
```

Other facts about the repo that the procedure relies on:

- default branch `master`, **not** protected → a direct push is possible, no PR needed;
- the account used must have `push` permission (`gh api repos/codecheckers/codecheckers --jq .permissions`);
- team slug is `codecheckers` (org has `admins`, `check-pub`, `codecheckers`, `institutional-codecheckers`);
- registration issues carry the `registration` label and are assigned to `nuest`.

## 1. Pick up the issue

```bash
gh issue list --state open --label registration --json number,title,author,createdAt,comments \
  --jq '.[] | "\(.number)\t\(.author.login)\t\(.createdAt)\tcomments:\(.comments|length)"'
gh issue view <N> --comments
```

Note the *author login* — it is the ground truth for the handle, which registrants often mistype.
If the issue already has maintainer comments, read them: it may be a follow-up to a request for
missing information, in which case the requested data is usually in a **comment**, not in the issue
body, and sometimes the registrant edited the original post instead of replying.

## 2. Extract the data row

The template asks for one CSV line inside a fenced block. Robust extraction:

```bash
gh issue view <N> --json body,comments --jq '.body' \
  | python3 -c "import sys,re; s=re.sub(r'<!--.*?-->','',sys.stdin.read(),flags=re.S); \
print('\n'.join(re.findall(r'\`\`\`(?:csv)?\s*(.*?)\`\`\`', s, flags=re.S)))"
```

Strip the HTML comments first — the template's instructions and its `Christina Codechecker` example
both live in comments and would otherwise be mistaken for data.

Target schema (see [CLAUDE.md](CLAUDE.md) for the column conventions):

```
name,handle,ORCID,contact,fields,languages
```

Normalisation to apply before validating — all of these occur regularly in real submissions:

| Observed | Fix |
| --- | --- |
| `Karla Lozano Gonzalez, @KarlaLG91, 0000-…` (spaces after commas) | strip whitespace around every field |
| `@cherylisabella ` / `valerieorozco988` | trim, ensure exactly one leading `@` |
| `"…, HPC, "` (trailing comma/space inside a quoted list) | trim the list items |
| `https://orcid.org/0000-…` | reduce to the bare ID |
| unquoted multi-value `fields`/`languages` | re-quote so the row still has 6 columns |
| `R(expert),Python(intermediate)` | acceptable as-is; do not invent spacing changes |

Never paste the row verbatim; always re-emit it through a CSV writer so quoting is correct.

## 3. Validate

Run all checks, then present a single pass/fail summary to the human.

**a) Six columns present.** The most common defect is a missing `contact` column (issue #77 submitted
5 fields), or the untouched template line still containing `name,@GitHub-handle,…` (issue #72).

**b) Handle resolves and matches the issue author.**

```bash
gh api users/<handle> --jq '{login,name,blog,email}'   # 404 = wrong handle
```

Store the **canonical `login`** returned by the API, not the spelling from the issue. Handles that no
longer resolve are a real historical problem in the CSV (`patrickeneche`, `subhanaliweb` are both 404
today — accounts renamed after registration). If the handle differs from the issue author, ask.

**c) ORCID exists.**

```bash
curl -s -o /dev/null -w '%{http_code}\n' -H 'Accept: application/json' \
  https://pub.orcid.org/v3.0/<ORCID>/person        # 200 = exists, 404 = bad ID
```

**d) `contact` — the ORCID email check.** This is the single most frequent reason a registration is
held up. If the submitted contact is the literal `see ORCID page`, that claim **must be verified**:

```bash
curl -s -H 'Accept: application/json' https://pub.orcid.org/v3.0/<ORCID>/person \
  | python3 -c "import json,sys; d=json.load(sys.stdin); \
print('emails:', [e['email'] for e in d['emails']['email']]); \
print('urls:', [(u['url-name'], u['url']['value']) for u in d['researcher-urls']['researcher-url']])"
```

- non-empty `emails` → accept `see ORCID page`;
- empty `emails` but a researcher URL → open that page and look for an address there, matching the
  established practice ("I didn't find an email via your ORCID profile **or personal page linked on
  it**"). Report what was checked;
- nothing found → this is the *ask-for-email* branch, [Step 4](#step-4-if-information-is-missing).

The ORCID public API needs no authentication. Note it returns `emails.email: []` both for "no email"
and for "email set to private", which is fine — either way there is no reachable address.

**e) Sanity of `fields` / `languages`.** Lower case by convention, comma-separated, most-proficient
first. An empty `languages` was historically chased up (issue #3).

**f) Already a member?** Idempotency check before inviting:

```bash
gh api orgs/codecheckers/teams/codecheckers/memberships/<login>   # 200 {state, role} or 404
gh api orgs/codecheckers/memberships/<login> --jq '{state,role}'  # org-level, 404 if not a member
grep -in '<login>' codecheckers.csv                                # already listed?
```

Someone can be in the team but not the list (issue #68) — that changes the wording of the reply.

**g) Optional questionnaire.** The ECR / motivation / time answers are not stored in the CSV, but they
inform the reply: registrants have twice been gently corrected on the ECR question ("Actually, you are
an early career researcher as a graduate student", #61; "as you are still a student … you are most
definitely an ECR", #74). Surface these answers to the human.

## 4. If information is missing

Do **not** invite or commit. Post a request for the missing piece and leave the issue open. Pick the
matching template from [Step 8](#8-message-templates), confirm with the human, then:

```bash
gh issue comment <N> --body-file <draft.md>
```

Track it for a follow-up: the established habit is a friendly ping after a few weeks ("@rkugyelka
Friendly ping - would be great to get you on the list.", #7; #47; #72), and, if a registration stays
incomplete for a long time, closing it with an explicit invitation to reopen (#47).

## 5. Draft and approve the reply

Compose the welcome from the canonical snippet in `not-a-bot.md`, then **always ask the human whether
to amend it for this person**, offering the specific hooks found while validating:

- prior relationship or event ("already contributing through the Rotterdam CHECK-NL event", #59;
  "as part of the Amsterdam UMC collaboration", #75; "congrats on already completing your first
  check", #71);
- existing codechecker profile URL on codecheck.org.uk (#62);
- ECR correction (#61, #74);
- apology for delay when the issue has been open for months (#30, #44, #46);
- greeting by first name ("Hi Marcel!", "Hello Yen-Chung,") vs. no salutation — both are in use;
- thanking a third party who referred them, if someone else commented on the issue (#23);
- a direct answer if the registrant raised a question or objection in their issue (#16).

Present the full rendered draft, not a diff, and offer: send as is / amend / skip.

## 6. Invite to the team

```bash
gh api --method PUT orgs/codecheckers/teams/codecheckers/memberships/<login> \
  -f role=member --jq '{state,role}'
```

- `state: pending` → the person is not yet an org member and has been emailed an invitation;
  `state: active` → they were already in the org, membership is immediate.
- The call is idempotent; re-running on an active member is harmless.
- 403 means missing scope (see [Step 0](#0-preconditions)) or IdP team synchronisation;
  422 means the target is an organization, not a user.

Only needed if the person is not in the org at all and you want to add them without a team:
`PUT /orgs/codecheckers/memberships/<login>` with `-f role=member`. Pending org invitations are
visible via `gh api orgs/codecheckers/invitations` and, per team, `…/teams/codecheckers/invitations`.

## 7. Post, commit, close

Order matters: comment first (so the issue notification and the commit arrive together), then commit
with `closes #N`, which closes the issue on push to `master` — no separate close call.

```bash
gh issue comment <N> --body-file <draft.md>

# append the normalised row (python3 csv writer, not echo, to get quoting right)
git add codecheckers.csv
git commit -m "add @<login>, closes #<N>"
git push origin master
```

Then verify, and only fall back to an explicit close if the push did not do it (e.g. the commit
message lacked the keyword, or the work happened on a branch):

```bash
gh issue view <N> --json state,stateReason
gh api --method PATCH repos/codecheckers/codecheckers/issues/<N> \
  -f state=closed -f state_reason=completed        # fallback only
```

Rows are **appended** in registration order — never sort or reorder the file.

For an incomplete registration abandoned by the author, close with `-f state_reason=not_planned`
and say it can be reopened at any time.

## 8. Message templates

`not-a-bot.md` holds the two canonical snippets. 53 welcome comments and ~20 information requests were
reviewed; these are the variants that recur.

**Welcome (canonical, unchanged since 2021 — copy verbatim from `not-a-bot.md`):** thanks + `:heart:`,
"I invited you to [the team] and added you to [the list]. Welcome!", then the "we will get in touch …
community CODECHECK … reviewer … editor" paragraph. Keep the community link as
`https://codecheck.org.uk/guide/community-workflow` (older comments use the outdated
`…/guide/community-process`).

**Welcome, already in the team** (#68) — replace the invitation sentence:
"You were already in [the team](…) and I now added you to [the list](…). Welcome!"

**Welcome, short form** (#67, #71, #75) — first two paragraphs only, dropping the "we will get in
touch" boilerplate. Used for people already engaged with the project.

**Missing email** (`not-a-bot.md` snippet 2; #27, #31, #36, #48, #49, #77) — "Unfortunately I didn't
find an email via your ORCID profile or personal page linked on it - can you please check? Thanks! We
simply want another option to reach you besides GitHub. I'll add you to the list then asap, just ping
me here." Mention explicitly what was checked when the linked page was inspected too (#31).

**Incomplete data** (#47): "Can you please update your data to include all fields from the template?"
followed by the header line in a `csv` block.

**Template not filled in at all** (#72): "It seems like something went wrong with the registration - it
only contains the template but not your information. Please ping me here when you had a chance to
update your details in the first post above."

**Friendly ping** (#7, #47, #72): one short line, `@handle` + "friendly reminder"/"friendly ping".

**Closing an abandoned registration** (#47): state what is missing, close, offer to reopen.

Tone markers to preserve across all of them: first-person singular from the community manager, an
emoji or two (`:heart:`, `:tada:`, `:muscle:`), "Thanks for your interest in becoming a codechecker!",
and a concrete next step for the registrant.

## 9. Maintenance / audit

The recurring checks live in [`annual-maintenance.md`](annual-maintenance.md) — run them once a year.
The quick reconciliation below is the one worth repeating whenever you touch the list:

The list and the team drift apart, so an occasional reconciliation is useful (currently 67 CSV rows
vs. 55 team members; 17 listed handles are not in the team, and 5 team members are not listed —
partly renamed accounts, partly people who never accepted or later left, plus the `team-codecheck`
bot):

```bash
python3 - <<'PY'
import csv, subprocess
listed = set()
for f, col in (('codecheckers.csv', 'handle'), ('institutional-codecheckers.csv', 'handle')):
    listed |= {r[col].lstrip('@').strip().lower() for r in csv.DictReader(open(f))}
team = subprocess.check_output(
    ['gh','api','--paginate','orgs/codecheckers/teams/codecheckers/members','--jq','.[].login'],
    text=True).split()
in_team = {t.lower(): t for t in team}
print('listed only:', sorted(h for h in listed if h not in in_team))
print('team only  :', sorted(t for l, t in in_team.items() if l not in listed and l != 'team-codecheck'))
PY
```

Check both files — an unlisted org member may be an institutional codechecker rather than a missing
volunteer row, and `@team-codecheck` is the org's bot account.

Handles that 404 on `gh api users/<h>` have been renamed or deleted and need a manual fix in the CSV.

## 10. API reference

| Purpose | Route | Token requirement |
| --- | --- | --- |
| List/read registration issues | `GET /repos/{o}/{r}/issues[/{n}]` (`gh issue list/view`) | repo read |
| Post reply | `POST /repos/{o}/{r}/issues/{n}/comments` (`gh issue comment`) | repo write |
| Close issue | `PATCH /repos/{o}/{r}/issues/{n}` `state=closed`, `state_reason=completed\|not_planned\|duplicate` | push or triage |
| Read team membership | `GET /orgs/{org}/teams/{slug}/memberships/{user}` → `{state,role}` | `read:org` |
| **Add to team (invites if needed)** | `PUT /orgs/{org}/teams/{slug}/memberships/{user}` `role=member\|maintainer` | `admin:org` |
| Remove from team | `DELETE /orgs/{org}/teams/{slug}/memberships/{user}` → 204 | `admin:org` |
| List team members | `GET /orgs/{org}/teams/{slug}/members` (`--paginate`) | `read:org` |
| Org membership state | `GET /orgs/{org}/memberships/{user}` | `read:org` |
| Is org member (204/404) | `GET /orgs/{org}/members/{user}` | `read:org` |
| Pending invitations | `GET /orgs/{org}/invitations`, `GET /orgs/{org}/teams/{slug}/invitations` | `admin:org` |
| Verify GitHub handle | `GET /users/{user}` | none |
| Update CSV without a local clone | `PUT /repos/{o}/{r}/contents/codecheckers.csv` (needs `sha`) | repo write |
| ORCID record exists | `GET https://pub.orcid.org/v3.0/{orcid}/person` (`Accept: application/json`) | none |
| ORCID public email | `GET https://pub.orcid.org/v3.0/{orcid}/email` | none |

`gh api` notes: `-f k=v` sends strings, `-F k=v` converts `true/false/null`/numbers, any field switches
the method to POST unless `--method` is given, `--jq` filters the response, `--paginate` follows all
pages, `-i` shows headers (useful for the 204/404 checks and for reading `X-OAuth-Scopes`).
