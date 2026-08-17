# Codecheckers Team

This repo collects all information about people conducting CODECHECKs as part of the [CODECHECK community](https://codecheck.org.uk/guide/community-workflow).

## Find a codechecker

You can take a look at the codecheckers table, [`codecheckers.csv`](codecheckers.csv) to find a suitable codechecker.
GitHub even provides a nice [search function](https://help.github.com/en/github/managing-files-in-a-repository/rendering-csv-and-tsv-data) for the file.
Consider skills earlier in the list to be more advanced, later ones to be less strong.
If you have a good candidate, please check the codechecker is currently not busy with too many CODECHECKs already (see assigned issues in the [CODECHECK register](https://github.com/codecheckers/register/)).

Alternatively, you can @-mention the codecheckers team with [`@codecheckers/codecheckers`](https://github.com/orgs/codecheckers/teams/codecheckers) in the issue for managing the codecheck and ask around for interested codecheckers by adding `@codecheckers/codecheckers` to an issue comment.

Finally, you can ask the author for recommendations, start an open call for codecheckers on Twitter, et cetera.

## Early career researcher (ECR) status

The last two columns of [`codecheckers.csv`](codecheckers.csv) record whether someone is an early career researcher, i.e., within eight years of the award of their PhD and not yet transitioned to an independent researcher.
We track it because some programmes and funders ask us to involve ECRs, and because it is a moving target: nobody's ECR status is permanent, so a plain yes/no goes stale silently.

Instead of a boolean, we record **when the status ends** and **when we last established it**:

| Column | Meaning |
| --- | --- |
| `ecr_until` | `YYYY-MM` - the month the eight-year window closes. Somebody is currently an ECR if this is in the future. |
| | `open` - is an ECR, but no end date can be determined yet, e.g., because the PhD is still ongoing. |
| | `expired` - is not an ECR, but no date is available to say since when. |
| | `NA` - unknown, we have never established it. **`NA` never means "no".** |
| `ecr_checked` | `<YYYY-MM>;<source URL>` - when the value was last established and what it is based on, either the person's ORCID profile (which carries the PhD date) or their registration issue (where they told us themselves). `NA` if never established. |

So "who is currently an ECR?" is `ecr_until` being `open` or a month in the future, and "whose entry needs a fresh look?" is `ecr_until` being `NA`, or an `ecr_checked` date that is more than a year old.
Note that the eight-year window is a guideline, not a rule: career breaks such as parental leave normally extend it, so a date in `ecr_until` may be adjusted by hand - please just tell us.

## Institutional codecheckers

Not everybody in the [Codecheckers Team](https://github.com/orgs/codecheckers/teams/codecheckers) is a volunteer.
Some people conduct CODECHECKs as part of their job, e.g., as research software engineers or as part of an institutional reproducibility programme, and they are members of the organisation so they can take on and submit checks for their institution.
They are recorded in [`institutional-codecheckers.csv`](institutional-codecheckers.csv) with their GitHub handle and the institution they codecheck for - the file exists so that org membership without a corresponding entry in `codecheckers.csv` is documented rather than confusing.

Institutional codecheckers do not go through the volunteer sign-up below; they are usually onboarded as part of setting up the collaboration with their institution.
Someone can be in both lists, e.g., when they registered as a volunteer and later also codecheck as part of their role.
Please do not contact institutional codecheckers with requests for volunteer checks - they pick up the work assigned within their institution.

## Sign up

If you want to [get involved](https://codecheck.org.uk/get-involved) as a codechecker, we need to run through the following steps:

1. Codechecker (you!) opens an issue **[using this link](https://github.com/codecheckers/codecheckers/issues/new?assignees=nuest&labels=registration&template=codechecker-registration.md&title=Register+as+codechecker)** (with an issue template)
2. Community manager makes sure all required information is there
3. Community manager invites the new codechecker to the [Codecheckers Team](https://github.com/orgs/codecheckers/teams/codecheckers) (Note: the team page is not public)
4. Communtiy manager welcomes the new codechecker
5. Communtiy manager saves information in the "database" and closes the issue with the commit

------

[About CODECHECK](https://codecheck.org.uk/)
