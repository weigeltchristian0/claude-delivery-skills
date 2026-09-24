# Repo catalog

Index every repo a symptom could route to, so triage picks a target before it
starts reading code.

Fill this in for your organisation. Group by domain and keep one line each. The
goal is routing, not documentation: enough to rule a repo in or out.

## Core application repos

| Repo | Role |
|---|---|
| `<frontend>` | The user-facing app |
| `<core-service>` | The main backend |
| `<identity-service>` | Users, roles, permissions |

## Supporting services

| Repo | Likely role |
|---|---|
| `<name>` | <one line> |

## Infrastructure

Rarely where application bugs live, usually where deploy and config bugs do.

## Trust note

Mark which entries are verified and which are inferred from naming. Before
deep-diving an unverified repo, read its README to confirm what it actually
does.
