# Annual maintenance

The lists in this repo decay: people rename their GitHub account, change institution, remove the email
from their ORCID profile, or never accept the team invitation. None of that produces a notification, so
it has to be checked deliberately. Work through this list once a year and record the date below.

Related: issue [#29](https://github.com/codecheckers/codecheckers/issues/29) (ping all codecheckers
regularly to keep the list updated).

| Run | Done by | Notes |
| --- | --- | --- |
| 2026-08 | @nuest (assisted) | first run; findings and fixes in the commit for this file |

## 1. Handles that no longer resolve or have been renamed

GitHub redirects renamed accounts on the web, but `codecheckers.csv` keeps the old string and
`@`-mentions of it stop working. `GET /users/{handle}` follows the rename and returns the *current*
login, so any mismatch is a rename; a 404 means the account is gone or was mistyped from the start.

```bash
python3 - <<'PY'
import csv, json, subprocess
for f in ('codecheckers.csv', 'institutional-codecheckers.csv'):
    for r in csv.DictReader(open(f)):
        h = r['handle'].strip().lstrip('@')
        p = subprocess.run(['gh','api','users/'+h,'--jq','.login'],
                           capture_output=True, text=True)
        if p.returncode:
            print(f, h, 'DELETED-OR-404')
        elif p.stdout.strip() != h:
            print(f, h, '-> @' + p.stdout.strip())
PY
```

Fix the row in place, keeping the canonical spelling returned by the API. Cross-check against the
author of the person's registration issue if the new handle is not obviously the same person.

## 2. `see ORCID page` contacts that no longer resolve to an email

Roughly a third of the list uses `see ORCID page` instead of an address. That is only valid while the
profile actually exposes a public email — people do set it back to private.

```bash
python3 - <<'PY'
import csv, json, urllib.request
for r in csv.DictReader(open('codecheckers.csv')):
    if r['contact'].strip().lower() != 'see orcid page':
        continue
    o = r['ORCID'].strip()
    req = urllib.request.Request('https://pub.orcid.org/v3.0/%s/person' % o,
                                 headers={'Accept': 'application/json'})
    try:
        d = json.load(urllib.request.urlopen(req, timeout=30))
    except Exception:
        print(r['handle'], o, 'ORCID-404'); continue
    if not d['emails']['email']:
        urls = [u['url']['value'] for u in d['researcher-urls']['researcher-url']]
        print(r['handle'], o, 'no public email; urls:', urls or 'none')
PY
```

For each hit: if a linked personal/institutional page has an address, put that in `contact`; otherwise
ask the person (see the *missing email* template in [`registration-runbook.md`](registration-runbook.md)).
Note the ORCID API cannot distinguish "no email" from "email set to private" — either way there is
nothing reachable for us.

## 3. Reconcile the lists with the GitHub teams

```bash
python3 - <<'PY'
import csv, subprocess
listed = set()
for f in ('codecheckers.csv', 'institutional-codecheckers.csv'):
    listed |= {r['handle'].lstrip('@').strip().lower() for r in csv.DictReader(open(f))}
team = subprocess.check_output(
    ['gh','api','--paginate','orgs/codecheckers/teams/codecheckers/members','--jq','.[].login'],
    text=True).split()
in_team = {t.lower(): t for t in team}
print('listed but not in team:', sorted(h for h in listed if h not in in_team))
print('in team but not listed:', sorted(t for l, t in in_team.items() if l not in listed and l != 'team-codecheck'))
PY

gh api orgs/codecheckers/invitations --jq '.[] | "\(.login)\t\(.created_at)"'          # still pending
gh api orgs/codecheckers/failed_invitations --jq '.[] | "\(.login)\t\(.failed_reason)"'  # expired/declined
```

Most "listed but not in team" entries are **expired invitations** — GitHub cancels them after 7 days
and the person never noticed. Re-invite and ping them on their registration issue:

```bash
gh api --method PUT orgs/codecheckers/teams/codecheckers/memberships/<login> -f role=member
```

"In team but not listed" is usually an institutional codechecker who never filed a registration issue →
add them to `institutional-codecheckers.csv` and to the `institutional-codecheckers` team.

## 4. Institutions of institutional codecheckers

People move. Check that each entry in `institutional-codecheckers.csv` still codechecks for the
institution recorded, and that the `institutional-codecheckers` GitHub team matches the file:

```bash
gh api --paginate orgs/codecheckers/teams/institutional-codecheckers/members --jq '.[].login'
```

Where someone has left the collaboration, decide whether to drop the row or move them to
`codecheckers.csv` as a volunteer — ask them, do not assume.

## 5. Row hygiene

Cheap to check, easy to let rot:

- six columns in every row of `codecheckers.csv`, two in `institutional-codecheckers.csv`;
- exactly one leading `@` on every handle, no stray whitespace, no `", "` between fields;
- ORCID present, bare (no URL), and matching `\d{4}-\d{4}-\d{4}-\d{3}[\dX]`;
- `fields` and `languages` non-empty;
- no duplicate handles or ORCIDs;
- the ORCID's name still plausibly matches the `name` column (catches copy-paste of the wrong ID).

## 6. Ping the codecheckers

Issue #29: contact everyone once a year — confirm their data is current, ask whether they are still
available, and remind them of the register. This is also the moment to ask people whose contact or
handle could not be repaired in steps 1–2.

## 7. Availability

Cross-check the [register](https://github.com/codecheckers/register/) for who is currently assigned,
so recommendations from `codecheckers.csv` do not go to people already busy with several checks.
