---
name: QA Test Cases
user-invocable: true
description: Generate and classify test cases for a completed development ticket ({{PROJECT_KEY}}) and write them to Confluence. Use when the user wants test cases for a ticket — generated from acceptance criteria, diff behaviours, and (if available) code-review findings — classified by Priority / Layer / Destructive / Execution (Auto vs Manual) and mapped to the canonical App Test Case Library. Writes the ticket's Confluence QA page (cases only) and enriches the area library. Optionally appends the Confluence page link to the Jira issue Description with `--to-jira`. One of the three QA sub-skills sequenced by `qa-orchestration`; runs standalone too.
arguments:
  - name: ticket_id
    description: The Jira ticket ID to generate cases for (e.g., {{PROJECT_KEY}}-1861). Space/comma-separated list also accepted when run standalone.
    required: true
  - name: to-jira
    description: "If set (--to-jira), also append the Confluence ticket-page link to the END of the Jira issue Description (managed block). Default OFF."
    required: false
  - name: no-library
    description: "If set (--no-library), skip App Test Case Library enrichment (write the ticket page only)."
    required: false
---

# QA Test Cases — Generation, Classification & Library Enrichment

## Purpose
Generate + classify test cases for a completed ticket, write them to the ticket's **Confluence QA page** (cases only), and **enrich the canonical App Test Case Library** (merge-by-ID). This is the test-case half of the QA system, split out of the old `qa-tickets-auto`. Code review lives in **qa-code-review**; browser execution in **qa-execute**; sequencing in **qa-orchestration**.

Writes **no local output files** — its deliverables are the Confluence pages. Standalone, it may download the ticket's image attachments to a scratch dir to analyse them (see Context bundle). Never opens a browser. The cases marked `Execution = Auto` are later run by **qa-execute**.

## Language
- Write in simple, plain language. Short sentences, common words.
- No filler, no preamble, no marketing phrasing. Every line must add information.
- One idea per line; prefer bullets over paragraphs.

## Flags
- `--to-jira` (default **off**): in addition to the Confluence pages (always written), append the ticket-page link to the **end of the Jira issue Description** as a managed block. See "Jira write-back".
- `--no-library` (default off): skip library enrichment; write the ticket page only.

## Confluence coordinates (`QA` space)
- Cloud `{{JIRA_HOST}}` · Space `QA` (id `{{QA_SPACE_ID}}`).
- **Ticket QA** parent `{{TICKET_PARENT_ID}}` — per-ticket pages are children titled `{{PROJECT_KEY}}-xxxx — <summary>`.
- **App Test Case Library** parent `{{LIBRARY_PARENT_ID}}`, area pages:
  | Area | id | prefix | Feature mapping |
  |------|----|--------|-----------------|
  | User Service | `{{AREA_ID_1}}` | `USER` | offboarding/deactivation, activation, roles, shops, work-schedules, settings |
  | Booking & Appointments | `{{AREA_ID_2}}` | `BOOK` | booking, slot suggestions, reschedule, meeting lifecycle |
  | Calendar | `{{AREA_ID_3}}` | `CAL` | calendar views, navigation, event display, filtering |
  | Customer | `{{AREA_ID_4}}` | `CUST` | customer CRUD, search, identity/merge, address, consent |
  | Auth | `{{AREA_ID_5}}` | `AUTH` | login/SSO, session, permissions/role boundaries |
Confluence write tooling — the cases table **must be full-width**, and the reliable recipe depends on the tool:
- **Default path (`mcp__atlassian-remote__*` — the canonical Atlassian MCP; prefer it over the deprecated `claude_ai_Atlassian` connector).** It accepts `html`/`markdown`/`adf` only (no `storage`). **Write the cases table with `contentFormat: "adf"`** and set the table node's `attrs.layout = "full-width"`, with **no fixed column widths** (no `colwidth`/`<colgroup>`/`width`): this MCP treats fixed widths as incompatible with full-width and silently drops the layout → centered/narrow. Status lozenges = ADF `status` nodes (`{"type":"status","attrs":{"color":"blue|purple|green|yellow|red","text":"FE"}}`).
- **Never write the cases table via `html`.** The html format has **no** way to carry full-width — a `<table>` (even `<table data-layout="full-width">`) always lands at `default` layout, and an `html` readback won't reveal it. Writing tables via `html` is the recurring narrow-table bug. **Verify via an `adf` read, never `html`**: every `table` must be `full-width` with zero `colwidth`.
- **Heads-up — downstream updates can re-flatten this table.** qa-execute later adds a "Latest run" pointer to this same ticket page; if that update round-trips the body through `html` it strips the layout you just set. qa-execute is instructed to re-assert `full-width` on write, but if you see a ticket page go narrow after a run, that update is the culprit — re-apply via `adf`.
- Storage tooling (only if a token-based `mcp__atlassian__confluence_*` tool is actually present): `content_format: "storage"`, `<table data-layout="full-width">`, no `<colgroup>`; lozenges/widths persist.

## Context bundle
When sequenced by `qa-orchestration`, use the passed **context bundle** (ticket + AC + comment digest + analysed screenshots + diffs + the qa-code-review findings) verbatim. Standalone, gather the ticket + AC + PR diffs **plus the comment digest and the downloaded-and-analysed screenshots** yourself (see qa-code-review's "Context bundle" steps 1–4 **and step 6**); code-review findings are then unavailable — generate from AC + comments + screenshots + diffs (skip only the review-derived negative cases).

## Phase 1: Generate & classify
Generate cases from these sources — don't invent generic ones: (1) one happy-path case per acceptance criterion; (2) a case per meaningful new behavior/branch in the diffs; (3) the highest-risk edge cases and any reported bug (e.g. a 500) — as explicit negative cases; (4) **the comment digest** — every bug, repro, or clarified/changed expected behaviour raised in the ticket's Jira comments becomes a case (a negative case for a reported bug; a corrected happy-path case when a comment changes the expected result); (5) **the analysed screenshots** — a bug-repro screenshot becomes a negative case reproducing that state, and a mockup/expected-UI screenshot sharpens the matching case's Expected (assert the labels, empty-state text, colours, and layout the image shows). When code-review findings are supplied, add a negative case per newly-found failure mode.

For **each** case set:
- **Priority** — P1 (blocking) / P2 (important) / P3 (nice).
- **Layer** — `Frontend` ({{FRONTEND_REPO}} UI) or `Backend` ({{CORE_SERVICE}}/{{IDENTITY_SERVICE}}/app/etc. API/DB/cross-service).
- **Destructive?** — `Yes` if the steps persist a change to shared state (confirm deactivation; cancel/reschedule/edit/save a meeting; change {{SPECIALIST_ROLE}}; create/delete records; send notifications; DB writes). `No` if read-only / UI-state-only (open & observe a modal, assert a list/exclusion/rendering, a validation error that does NOT submit, navigation, filtering, search display).
- **Execution** — **`Auto`** only if `Layer = Frontend` AND `Destructive? = No` (these are run by Playwright via qa-execute). Otherwise **`Manual`** — i.e. every Backend case, and every destructive Frontend case, must be run by hand. **This is the automation safety boundary**: only non-destructive frontend cases are ever automated; everything else is explicitly `Manual`.
- **Acting role** — which user performs the case. Default **admin** (the `e2e-admin` test account). If it needs `{{SPECIALIST_ROLE}}`, `booker`, or `admin-{{SPECIALIST_ROLE}}`, state that in Preconditions (e.g. "logged in as booker") so qa-execute picks the matching E2E account.
- **Library mapping** — pick the area (feature→area table), read that area page, and tag the case `NEW` (no match), `IMPROVES <AREA>-TC-<n>` (refines an existing case), or `REUSES <AREA>-TC-<n>` (an existing canonical case applies unchanged — normal and valuable; it links the ticket to existing coverage).

Each case: working ID (`TC-01`…), Title, Priority, Layer, Destructive?, Execution, Library mapping, Preconditions, numbered Steps (concrete, real UI labels/paths), Expected.

**Reconsider existing cases:** if the ticket already has a Confluence page from a prior run, decide — based on any supplied code-review findings — whether to add/change/remove cases. Summarize the delta.

## Phase 2: Confluence sync
**a) Ticket QA page (test cases ONLY).** Find an existing child of `{{TICKET_PARENT_ID}}` titled with this ticket ID; if present `confluence_update_page` (idempotent — replace the body, keep history; skip the write if identical), else `confluence_create_page` (`space_key: "QA"`, `parent_id: "{{TICKET_PARENT_ID}}"`, title `<TICKET_ID> — <summary>`). The code review is NOT on this page — it's the dated sub-page written by qa-code-review (shown as a child).
- Minimal header: status + acting role + a note that code-review results are in the dated child page(s); state the Execution split outright and **list the Manual case IDs** (e.g. "Manual cases (run by hand): TC-01, TC-04…").
- Columns: **ID · Title · Pri · Layer · Destructive · Execution · Library · Preconditions · Steps · Expected** (no Result column — run results live on the dated Execution Run sub-page written by qa-execute).
- **Full-width layout always.** Storage tooling: `<table data-layout="full-width">` with a `<colgroup>` of sized columns. `atlassian-remote` (adf) tooling: table node `attrs.layout = "full-width"` and **no** fixed column widths (see the tooling note above — fixed widths drop the layout on this MCP).
- Render Layer/Destructive/Execution as **status lozenges**: Layer `FE` (blue) / `BE` (purple); Destructive a red `yes` (blank if not); **Execution `Auto` (green) for non-destructive FE, `Manual` (amber/yellow) otherwise**. Acting role noted in Preconditions when not the default admin.
- The generator `.qa-auth/gen-qa-adf.js` (and the batch helper) emits ready-to-push **storage** with this exact column set — prefer it over hand-authoring; push its output with `content_format: "storage"`.

**b) Enrich the library** (skip if `--no-library`). For each case tagged `NEW`/`IMPROVES`, open the mapped area page and **merge by ID** (never blind-append):
- `NEW` → next free `<PREFIX>-TC-<n>` (read page, max existing n, +1) and add a row.
- `IMPROVES <ID>` → update that row in place.
- `REUSES <ID>` → do NOT modify the library; keep the case on the ticket page tagged with the canonical `<ID>`.
Update the ticket page's Library column with the final canonical IDs. Area pages use the same full-width all-inline table with the same lozenges: **ID · Title · Pri · Layer · Destructive · Execution · Preconditions · Steps · Expected · Source ticket**.

## Jira write-back (only when `--to-jira`)
The Confluence pages are always written; this is additive — and it edits the **Description**, not a comment.
- **Resolve to the parent issue.** If `ticket_id` is a **sub-task**, act on its parent task/story/bug. Never touch a sub-task.
- **Managed block at the END of the Description.** Read the issue's current description (`jira_get_issue`). Append (or replace, if already present) a single auto-managed block at the very end, in **markdown**, identified by its signature line `_QA — auto-managed (do not edit below)_`:
  > `----`
  > `_QA — auto-managed (do not edit below)_`
  > `QA test cases (N cases · P1×a / P2×b / P3×c · M Auto / K Manual): [<TICKET> — QA page](<ticket page url>)`
  Everything above the `----` rule is left untouched. On re-run, locate the existing block by that signature line and **replace only that block** (never stack duplicates); it always stays at the very end. Update via `jira_update_issue` (description field), `contentFormat: "markdown"`.
- **Working link, no raw markers.** Use markdown link syntax `[text](url)` — NOT Jira wiki syntax `[text|url]` (renders as literal text under markdown). Do NOT use an HTML-comment marker (`<!-- ... -->`) — it shows as visible literal text in Jira; the italic signature line is the idempotency key instead.

## Final output (stdout)
One line: `<TICKET> → <#cases> cases (<#Auto> Auto, <#promoted NEW+IMPROVES>) → <ticket page url>` (+ `· jira-description-linked` when `--to-jira`).

## Data warehouse (optional, read-only)
A read-only mirror of the **production** data warehouse may be available at `{{DWH_DIR}}/` (`<BASE>` = `QA_BASE_DIR` or CWD). **Read `{{DWH_DIR}}/SCHEMA_GUIDE.md` first** (schema map, `<table prefix>` tables, join keys, gotchas — incl. that **the DWH syncs on a delay, so a missing/recent row may just be unsynced, not absent**). Query from that dir: `python -c "from db import run; print(run('select ...'))"` (SELECT-only). If absent or a query errors, **skip it silently** — cases must still be generated from AC + diffs.

Use it to **ground cases in reality** so steps and preconditions are concrete: the actual `STATUS`/type enum values that occur (e.g. real `APP_MEETING.STATUS` and `APP_MEETING_TYPE.NAME` values), realistic data distributions, and **edge cases that genuinely exist in prod** (a meeting with a null field, a user with overlapping appointments, an archived row) — these make the strongest negative cases. **Two hard rules:** (1) the DWH is PROD, the run target is **stage** — never tell qa-execute to use a prod ID as stage test data; describe the *shape*, let execution pick a stage entity. (2) **Never paste real customer PII** (names, emails, addresses from `APP_CUSTOMER`/`APP_USERS`) into Confluence or Jira — use placeholders or IDs.

## Guidelines
- Only non-destructive frontend cases get `Execution = Auto` — non-negotiable (the automation safety boundary). Everything else is explicitly `Manual`.
- Merge-by-ID into the library; never create duplicate canonical cases.
- Concise; skip "all good" sections.
- Project key `{{PROJECT_KEY}}`; cloud `{{JIRA_HOST}}`. (Feature→repo/area mappings as in qa-code-review's architecture block.)
