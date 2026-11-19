# Spike 0001: iCloud CalDAV, read-only

**Goal:** retire the unknowns ADR-0001 marks **[spike]** before the cache and
recurrence design is frozen. It is throwaway exploration code under `spike/`,
not product code. Its outputs are findings written into this document and
synthetic fixtures for the test suite.

## Ground rules

- **Read-only, enforced in code.** The spike uses a fetch wrapper that throws
  on any method other than `PROPFIND`, `REPORT`, `GET`, or `OPTIONS`. The
  wrapper runs before any request leaves the process.
- **Credentials via 1Password only:**
  `op run --env-file=spike/.env.op -- pnpm tsx spike/<step>.ts`, where
  `spike/.env.op` holds only `op://` references (`ICLOUD_APPLE_ID`,
  `ICLOUD_APP_PASSWORD`). The script exits non-zero if either is empty.
  Nothing is printed except redacted values.
- **Real data stays local:** raw responses go to `spike/out/` (gitignored).
  Findings in this doc use counts, shapes, and redacted names. Fixtures
  derived from real events are rewritten with synthetic titles, places, and
  UIDs before they are committed.
- **Gentle on iCloud:** requests are sequential, with at most 1 per second.
  Stop on the first 429 and record it.

## Steps

Each step is a bead under the spike epic, and each records findings here.

| # | Question | Method | Done when |
|---|----------|--------|-----------|
| S0 | Auth and discovery work | tsdav `createDAVClient` (Basic auth) → `current-user-principal` → `calendar-home-set`. Record the partition host (`pNN-caldav.icloud.com`) and any redirects. | Principal and home URLs resolved; a wrong password yields a clean 401 that we record |
| S1 | Which collections exist | `fetchCalendars`; dump displayname, resourcetype, `supported-calendar-component-set` (VEVENT vs VTODO/Reminders), ctag, color, `current-user-privilege-set` | Table of collections compared with the calendars you expect to see |
| S2 | Shared calendars | Identify calendars shared *to* this Apple ID among S1 results (owner/`CS:shared` props, privilege set). You confirm which family calendars you expect; we try reading one you know is **not** shared and expect a 403/404 | Answer: whether shared-to-me calendars are visible, and with what privileges; whether unshared ones are unreachable |
| S3 | Subscribed (webcal) calendars | Look for `CS:subscribed` resourcetypes / `CS:source` hrefs in the home set. If they're absent, list the subscriptions you have in Calendar.app and check whether their ICS URLs are fetchable directly | Answer plus a recommendation: ignore them, fetch the ICS URL in the broker, or out of scope |
| S4 | Sync primitives | For one calendar: read `getctag` and `sync-token`; run `sync-collection` with an empty token, then the returned token (expect no changes); you edit one event in Calendar.app; run the token again; also test an invalid token | Which of ctag / sync-collection / ETag-diff iCloud honors; how a changed or deleted resource shows up; how token expiry looks |
| S5 | Recurrence and time zones | You name 3–5 real recurring events that include a **moved** instance, a **canceled/deleted** instance, an all-day multi-day event, and ideally a DST-crossing series. Fetch the raw `.ics`, expand a 90-day window with ical.js, compare against Calendar.app, and compare against a server-side `<C:expand>` calendar-query | ical.js expansion matches Calendar.app on every case; we know whether `<C:expand>` works on iCloud and how cancellations are encoded (EXDATE vs RECURRENCE-ID with `STATUS:CANCELLED`); scrubbed fixtures exist for each case |
| S6 | Rate limits and failures | Time a full-sync pass over every calendar; record latency, payload size, any 429/503 and `Retry-After`; test a bad password (401 body/headers) | Numbers to set the sync cadence (ADR D8) and the backoff policy |

## What I need from you

1. The 1Password **vault/item** that holds the Apple ID and app-specific
   password (or let me pick a name and you create the item). It also needs a
   service account token in the environment where the spike runs (otherwise
   `op` waits silently for an interactive approval).
2. The list of calendars you **expect** to see (S1), which ones are shared to
   you by someone else (S2), and any webcal subscriptions you rely on (S3).
3. For S4, one throwaway event you're happy for me to ask you to edit.
4. For S5, a few recurring events, by title, with known moved and canceled
   instances.

## Findings

_To be filled in per step._
