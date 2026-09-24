# Configuration

Every skill in this directory reads its organisation-specific values from here.
Fill this in once, then find and replace the `{{PLACEHOLDER}}` tokens across the
skill files, or leave them and let the agent resolve them from this file.

Nothing else should need editing.

## Issue tracker

| Placeholder | Meaning | Example |
|---|---|---|
| `{{PROJECT_KEY}}` | Project key tickets are filed under | `ACME` |
| `{{JIRA_HOST}}` | Tracker host | `acme.atlassian.net` |
| `{{AC_FIELD_ID}}` | Custom field holding acceptance criteria. Leave blank if your tracker has no dedicated field, and the skills will put acceptance criteria in the description instead. | `customfield_10001` |

## Source control

| Placeholder | Meaning | Example |
|---|---|---|
| `{{WORKSPACE}}` | Workspace or organisation that owns the repos | `acme-eng` |

## Services

Name the repos the skills route work to. If you have fewer than five, delete the
rows you do not need and remove the matching lines from `ARCHITECTURE.md`.

| Placeholder | Meaning |
|---|---|
| `{{FRONTEND_REPO}}` | The user-facing application |
| `{{CORE_SERVICE}}` | The main backend, where most business logic lives |
| `{{IDENTITY_SERVICE}}` | Users, roles, permissions, schedules |
| `{{CUSTOMER_SERVICE}}` | Customer or account records |
| `{{ENGINE_SERVICE}}` | Any specialised engine, for example a solver or recommender |
| `{{INTEGRATION_SERVICE}}` | The data integration or sync layer |

## Test environment

| Placeholder | Meaning | Example |
|---|---|---|
| `{{STAGE_URL}}` | Where browser tests run | `https://stage.acme.com` |
| `{{AUTH_PROVIDER}}` | Identity provider the test accounts log in through | `Auth0` |
| `{{SPECIALIST_ROLE}}` | A non-admin domain role used in test cases | `technician` |

Test account credentials belong in `.qa-auth/e2e-accounts.json`, which must be
git-ignored. Never put credentials in a skill file.

## Documentation space

Only needed by the QA skills, which write their output to a wiki.

| Placeholder | Meaning |
|---|---|
| `{{QA_SPACE_ID}}` | The space QA pages live in |
| `{{TICKET_PARENT_ID}}` | Parent page that per-ticket pages hang under |
| `{{LIBRARY_PARENT_ID}}` | Parent page for the canonical test case library |
| `{{AREA_ID_1}}` to `{{AREA_ID_5}}` | One page per test area, for example auth, booking, calendar |

If you do not use a wiki, point these at a directory instead and have the skills
write markdown. They do not depend on any wiki-specific feature beyond tables
and collapsible blocks.

## Data warehouse

| Placeholder | Meaning |
|---|---|
| `{{DWH_DIR}}` | Directory holding a read-only warehouse query helper and its schema guide |

Entirely optional. Every skill that touches it is written to skip silently when
it is absent. If you do use one, it needs:

- A `db.py` exposing a `run(sql)` that refuses anything but a single `SELECT`.
- A `SCHEMA_GUIDE.md` covering the table map, join keys, soft deletes, and above
  all the sync lag. Several skills specifically warn against reading a missing
  row as proof that something does not exist, and that warning only works if the
  schema guide documents how stale the mirror can be.
