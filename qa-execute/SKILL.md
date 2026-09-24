---
name: QA Execute
user-invocable: true
description: Run the browser-testable QA cases for a ticket against stage with Playwright. Use this whenever the user wants to execute, run, or verify the automated browser tests for an {{PROJECT_KEY}} ticket, run the Playwright cases for a ticket, or check a ticket's frontend QA on stage. Reads the ticket's QA page in the Confluence `QA` space, runs ONLY the cases marked `Execution = Auto` (non-destructive frontend cases), and writes PASS/FAIL results to a dated Execution Run sub-page. Companion to `qa-test-cases` (which generates and classifies the cases); prefer this skill for the execution half. Optionally posts a results comment to the Jira issue with `--to-jira`.
arguments:
  - name: ticket_id
    description: The Jira ticket ID whose marked cases to run (e.g., {{PROJECT_KEY}}-1861)
    required: true
  - name: role
    description: "Which E2E test account to log in as: admin (default), {{SPECIALIST_ROLE}}, booker, or admin-{{SPECIALIST_ROLE}}"
    required: false
  - name: to-jira
    description: "If set (--to-jira), also post a NEW dated results comment on the Jira issue (one per run). Default OFF."
    required: false
---

# QA Execute — Browser Test Runner

## Purpose
Execute the **browser-testable** test cases for a ticket against stage and record the results. The cases come from the ticket's page in the Confluence `QA` space, written there by `qa-test-cases`. This skill runs only the cases that skill marked `Execution = Auto` — which by construction are **non-destructive frontend** cases, so they are safe to run unattended.

This skill is the execution half of the QA system; `qa-test-cases` is the generation/classification half, `qa-code-review` is the review half, and `qa-orchestration` sequences all three. See `qa-skills-architecture.html`.

## Language
- Write in simple, plain language. Short sentences, common words.
- No filler, no preamble, no marketing phrasing. Every line must add information.
- One idea per line; prefer bullets over paragraphs.

## Arguments & flags
- `ticket_id` (required): the Jira ticket ID (e.g., `{{PROJECT_KEY}}-1861`).
- `role` (optional): which E2E account to use.
- `--to-jira` (optional, default **off**): in addition to the Confluence run sub-page (always written), post a NEW dated results comment on the Jira issue (one per run). See "Jira write-back".

## Confluence coordinates
- Cloud: `{{JIRA_HOST}}` · Space `QA` (id `{{QA_SPACE_ID}}`)
- **Ticket QA** parent page id `{{TICKET_PARENT_ID}}` — ticket pages are children, titled `{{PROJECT_KEY}}-xxxx — <summary>`.
- **Execution Run sub-pages** are children of the ticket page (same relationship code-review sub-pages have), titled `{{PROJECT_KEY}}-xxxx · Execution Run — <DATE>`. One per calendar date; re-running the same day updates that date's page.

## Stage target & accounts
- `{{STAGE_URL}}` (override with `TEST_BASE_URL`).
- **Auth uses dedicated E2E test accounts** logging in through the **custom {{AUTH_PROVIDER}} email/password form** (the scriptable path, not corporate SSO). Credentials live in `<BASE>/.qa-auth/e2e-accounts.json` (git-ignored — never put them in this skill or in Confluence). The file maps roles → `{email, password, roles}`: `admin` (default), `{{SPECIALIST_ROLE}}`, `booker`, `admin-{{SPECIALIST_ROLE}}`. The `role` argument (or a case's required role) picks which account to use; the persistent profile keeps the session between runs.

---

## Workflow

### Phase 1: Locate & read the ticket's QA page
1. Find the ticket page: list children of `{{TICKET_PARENT_ID}}` with `mcp__atlassian__confluence_get_page_children` (parent `{{TICKET_PARENT_ID}}`) and match the title, or `mcp__atlassian__confluence_search` (CQL: `space = QA AND title ~ "<TICKET_ID>"`). Read it with `mcp__atlassian__confluence_get_page` using `convert_to_markdown: false` (raw storage XML) so the table round-trips cleanly.
2. Parse the full-width cases table and select **only rows where `Execution = Auto`**. Each row already holds that case's Preconditions/Steps/Expected inline (no expands to chase). If there are no `Execution = Auto` rows, write nothing, report "no browser-testable cases for <ticket>", and stop. (Legacy pages may still use a `Browser = Yes` column — treat `Browser = Yes` as equivalent to `Execution = Auto`.)

### Phase 2: Authenticate (auto-login with the test account)
Pick the account: the `role` argument if given, else the role the cases require, else `default` (admin) from `e2e-accounts.json`.

1. Read `<BASE>/.qa-auth/e2e-accounts.json` and get the chosen account's `email`/`password`. (`<BASE>` = `QA_BASE_DIR` or CWD.)
2. Navigate to `TEST_BASE_URL`. If you land in the app **already signed in as the right account**, skip to Phase 3 (persistent profile reuse). If signed in as a *different* account than needed, log out first.
3. **Log in via the {{AUTH_PROVIDER}} custom form** (this is the supported, scriptable path for these dedicated test accounts — not corporate SSO):
   - From the app login page click **Login** to reach the {{AUTH_PROVIDER}} sign-in screen.
   - In the **"Sign in with your email and password"** section: type the account email into the Email field, the password into the Password field, click **Sign in**. (Do NOT use the corporate SSO button.)
   - Wait until redirected back to the app (`{{STAGE_URL}}`, no longer on `/login` or a the identity provider host).
4. **Fallbacks if auto-login fails** (missing creds file, unexpected MFA/captcha, or still on the login page):
   - **Interactive run:** leave the login page open and ask the user to finish signing in (as the chosen `role` account), then re-check and continue.
   - **Headless run** (`claude -p`/cron): mark every selected case `BLOCKED — auto-login failed (<reason>)`, write back (Phase 4), and stop. Never hang.

Treat the password as a secret: use it only to fill the form; never echo it to stdout, the report, or Confluence.

### Phase 3: Execute each marked case
For each selected case, in order:
1. Follow its Steps with the Playwright MCP (`mcp__plugin_playwright_playwright__*`): navigate, snapshot (preferred over screenshot for finding elements), click, type, wait.
2. **Last-resort repo fallback (only when stuck).** If — and only if — you cannot find your way (the target element is absent from the snapshot, a control is genuinely ambiguous, or the expected text/route is unclear), consult the frontend source at `<BASE>/{{FRONTEND_REPO}}` with `Grep`/`Glob`/`Read` to resolve the stable selector (e.g. an `id`/`data-testid`), the exact route, or the expected label/validation string, then continue. **Before trusting the repo, make sure it's on the ticket's code** — the working tree is often left on an unrelated branch or a stale/detached HEAD; if a feature file from the diff is missing or the symbol you need isn't there, `git -C "<BASE>/{{FRONTEND_REPO}}" fetch origin && git checkout <branch> && git pull --ff-only` onto the PR's branch (or just fall back to snapshot-only). Guardrails: the live page always wins — stage runs a deployed build that may differ from the repo, so a repo hint that doesn't match the live DOM is discarded, not forced. If `<BASE>/{{FRONTEND_REPO}}` is absent on this host, skip the fallback and proceed snapshot-only (behavior unchanged). Never use the repo to expand scope — it only helps you locate/verify what the case already describes.
3. **Defense in depth — never mutate shared state.** These cases are supposed to be non-destructive, but if a step would actually persist a change (confirm a deactivation, cancel/reschedule/save a meeting, change {{SPECIALIST_ROLE}}, delete), **do NOT perform that step.** Mark the case `SKIPPED — step is destructive (should not have been marked Execution=Auto)` and flag it; this is a classification bug in `qa-test-cases` worth reporting. Opening/observing modals, asserting lists/exclusions, validation errors that don't submit, navigation, and filtering are all fine.
4. Compare observed vs. the case's Expected → **PASS** / **FAIL** (with the observed difference). A thrown error or unexpected state is **FAIL**, never a reason to abort the rest of the run.
5. **Screenshot every case.** After judging each case — **PASS or not** — capture a screenshot with the Playwright MCP to `<BASE>/qa-tickets-reports/<DATE>/screenshots/<TICKET>-<CASE_ID>.png` (`<BASE>` = `QA_BASE_DIR` or CWD; `<DATE>` = system date `YYYY-MM-DD`) as a visual record of the final state. Every executed case gets one, not just failures. Same behavior locally and on the VM. This local file is the **source for the upload-and-embed** onto the run sub-page in Phase 4.

### Phase 4: Write results to a dated run sub-page + pointer on the ticket page
Do NOT fill the ticket page's Result column anymore. Instead:

1. **Find or create the dated run sub-page.** Title `<TICKET_ID> · Execution Run — <DATE>` (`<DATE>` = system date `YYYY-MM-DD`). List the ticket page's children with `mcp__atlassian__confluence_get_page_children` (parent = the ticket page id) and look for that exact title:
   - exists (same-day re-run) → `mcp__atlassian__confluence_update_page` (auto-bumps the version);
   - absent → `mcp__atlassian__confluence_create_page` with `parent_id` = the ticket page id, `space_key` = `QA`.
   Author the body in **storage format** and pass `content_format: "storage"` — storage preserves status lozenges, `colgroup` column widths, and `data-layout="full-width"`; markdown would flatten them. (Snippets: lozenge = `<ac:structured-macro ac:name="status"><ac:parameter ac:name="title">PASS</ac:parameter><ac:parameter ac:name="colour">Green</ac:parameter></ac:structured-macro>`; header panel = `<ac:structured-macro ac:name="tip"><ac:rich-text-body>…</ac:rich-text-body></ac:structured-macro>`.)
   - **If only the `mcp__atlassian-remote__*` MCP is available** (no `storage` format — only `html`/`markdown`/`adf`): write the run-results table with `contentFormat: "adf"`, set the table node's `attrs.layout = "full-width"`, and add **no** fixed column widths (fixed widths make this MCP drop the full-width layout → narrow table). Status cells = ADF `status` nodes. **Verify via an `adf` read, not `html`** — the html readback strips `attrs.layout`. Note: no connected Atlassian MCP has an attachment-upload tool, but the screenshot embed still works — step 2b uploads via a **direct Confluence REST call** (not the MCP), so always keep the `[[SHOT:…]]` markers and run the embed pass.
2. **Run sub-page body:**
   - A run-header panel: `<DATE>` · tally `N executed · P PASS · F FAIL · B blocked` · target user (name + `{{IDENTITY_SERVICE}}/<id>`) · acting role · auth note (which e2e account was used; note any e2e failure).
   - A **full-width table** (`<table data-layout="full-width">` — always span the page, never default/centered) of **only the executed browser cases**, columns `ID · Title · Steps · Expected · Result`. Result cell = `PASS` / `FAIL: <observed>` / `BLOCKED: <reason>` / `SKIPPED: <reason>`. End **every** case's Result cell with a literal marker `[[SHOT:<TICKET>-<CASE_ID>.png]]` (every executed case is screenshotted, Phase 3 step 5) — step 2b replaces each with the embedded image. No separate Screenshot column.

2b. **Attach & embed the screenshots (every executed case) — via a direct Confluence REST call.** No connected Atlassian MCP exposes an attachment-upload tool, so upload directly to the Confluence REST API. Authenticate with the **same Atlassian credentials the MCPs already use** — email + API token, read at runtime from the user-scope `mcpServers` env in `~/.claude.json` (e.g. `jira-dev-status`'s `JIRA_EMAIL` / `JIRA_API_TOKEN`); Basic auth = base64(`email:token`). **Never hardcode the token in this skill or on the page** — read it from config each run. Sequence:
   - **Upload** each PNG: `POST https://{{JIRA_HOST}}/wiki/rest/api/content/<run-page-id>/child/attachment` with header `X-Atlassian-Token: nocheck` and multipart body `file=@<absolute path>`. Re-posting the same filename just adds a version — safe on re-runs. (curl: `curl.exe -s -X POST -H "Authorization: Basic <b64>" -H "X-Atlassian-Token: nocheck" -F "file=@<path>" <url>`.)
   - **Embed:** `GET .../content/<run-page-id>?expand=body.storage,version`, replace each `[[SHOT:<file>]]` marker (or any `<code><file></code>` reference) in the Result cells with `<ac:image ac:width="900"><ri:attachment ri:filename="<file>" /></ac:image>`, then `PUT .../content/<run-page-id>` with the updated `body.storage.value` and `version.number + 1`. (An `ac:image` only renders once its attachment exists, so upload before this embed pass.) The page body can still be authored/updated via the MCP; only the attachment upload requires the direct REST call.
   - **⚠ UTF-8 — any direct REST round-trip of a page body must be byte-safe.** Use `curl.exe` or Python `urllib` (both send/receive raw UTF-8). **Do NOT use PowerShell 5.1 `Invoke-RestMethod`/`Invoke-WebRequest` to read-then-write a page body** — when the response has no charset it decodes UTF-8 as Latin-1, so em-dashes (`—`) and middots (`·`) silently become mojibake (`â`, `Â·`) on write-back. If you must use PowerShell, force UTF-8 on the read (decode `RawContentStream` as UTF-8) and send the body as UTF-8 bytes. Always re-read after writing and confirm no `â`/`Â·` artifacts appear.
3. **Main ticket page pointer.** Add/replace a single lightweight panel near the top: `Latest run: <PASS x/y> — <DATE> → <link to the run sub-page>`. Leave the case table's content untouched (the ticket page has no Result column — results live only on the run sub-page). If a prior "Latest run" panel exists, replace it in place (idempotent); make no other change.
   - **⚠ This update flattens the cases table unless you do it in ADF.** A full-page update re-sends the *entire* body. On the `mcp__atlassian-remote__*` MCP, reading the body as `html` and writing it back as `html` **silently drops the cases table's `attrs.layout = "full-width"` and re-injects per-cell `colwidth` → the carefully-built full-width table from qa-test-cases collapses to a narrow/centered table.** This is the #1 cause of "tables went narrow again" on ticket pages.
   - **Do this instead (atlassian-remote):** read the ticket page with `contentFormat: "adf"`; edit only the pointer panel node; **before writing, re-assert `attrs.layout = "full-width"` on every `table` node and delete any `colwidth` from every cell**; write back with `contentFormat: "adf"`. Then **verify with an `adf` read** that every table is `full-width` with no `colwidth` (an `html` read cannot show this). Storage tooling (`content_format: "storage"`, if a token-based `confluence_*` tool is present): read with `convert_to_markdown: false` and keep `<table data-layout="full-width">` intact with no `<colgroup>`.
4. **Never lose results.** If sub-page create/update fails, still emit the full results to stdout (Phase 5) and note the failure; do not abort.

### Phase 4b: Jira write-back (only when `--to-jira`)
The Confluence run sub-page is always written; this is additive.
- **Resolve to the parent issue.** If `ticket_id` is a **sub-task**, find its parent task/story/bug (`jira_get_issue` → parent) and act on the parent. Never comment on a sub-task.
- **New results comment every run.** Post a NEW bot comment on each run, identified by its leading signature `_QA Execution Run_` (italic). NEVER edit or replace a previous run's comment — run history stays in the comment stream, matching the dated run sub-pages. The run date goes in parens after the tally, NOT inside the signature. Do NOT use a raw HTML-comment marker (`<!-- ... -->`) — it renders as visible literal text in Jira's ADF.
- **One line (compact), NO code.** `_QA Execution Run_ — <emoji> **P/N PASS** (<DATE>): <one short plain-language clause>; failed: <case IDs + one-line observation in plain words, omit if none>. [Run details →](<run sub-page url>)`. Emoji ✅ when all pass, ⚠️ when any FAIL/BLOCKED. No status transitions, no labels.
- **Formatting rules (the markdown→Jira conversion is fragile):** (1) NO backticks/inline code anywhere — no selectors, URLs-as-code, file/method names; each backtick token renders as a code lozenge and shreds the one-liner into fragments. (2) Keep the italic signature EXACTLY `_QA Execution Run_` — putting parentheses or anything else inside the underscores breaks the italics and shows literal `_…_`. (3) Plain business language only.
- **Working link.** Post with `contentFormat: "markdown"` and markdown link syntax `[text](url)` — NOT Jira wiki syntax `[text|url]` (does not render under markdown).

### Phase 5: Report to stdout
Print one line per executed case (`<CASE_ID> → PASS/FAIL/BLOCKED/SKIPPED`) and a tally, then the run sub-page URL, so a scheduled run's log is self-explanatory.

---

## Data warehouse (optional, read-only — limited use here)
A read-only mirror of the **production** DWH may exist at `{{DWH_DIR}}/` (see `{{DWH_DIR}}/SCHEMA_GUIDE.md`; query via `python -c "from db import run; ..."`, SELECT-only). **Important limitation for this skill: the DWH is PROD; the run target is stage (`{{STAGE_URL}}`), a separate database.** So the DWH **cannot** verify the effect of a stage action and **must not** be used to pick stage test entities — a prod ID won't reliably exist on stage. Its only legitimate use here is read-only *interpretation*: understanding what a realistic value/shape looks like when a case's Expected is ambiguous (alongside the repo fallback in Phase 3 step 2). Also note **the DWH syncs on a delay** — a missing/recent row may just be unsynced, so never read absence as proof an entity doesn't exist. It is never required; if absent, behaviour is unchanged. Never write to it (the guard blocks it anyway), and never paste prod PII into the run sub-page.

## Guidelines
- **Only run `Execution = Auto` rows** (legacy `Browser = Yes` is equivalent). Never opportunistically test destructive or backend cases.
- **Read-only on shared stage** beyond the non-destructive steps themselves. When in doubt about whether a step persists, skip it and flag.
- **Concise** — one line per case in the summary; detail only on FAIL.
- Idempotent: re-running on the same date updates that date's run sub-page (and refreshes the ticket page's "Latest run" pointer); a new date creates a new run sub-page, preserving history. It never duplicates rows; the ticket page is only ever touched to refresh the "Latest run" pointer (it has no Result column).
- Repo fallback is a last resort only (Phase 3 step 2): use it when the browser snapshot can't get you there, never proactively, and never to widen scope. The live page wins over any repo hint.