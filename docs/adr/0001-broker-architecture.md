# ADR-0001: Calendar broker architecture

- **Status:** Draft. Each open question is tracked as a `decision` bead and is
  ratified only with an explicit go-ahead. Sections that depend on the spike
  (`docs/spike-0001-icloud-caldav.md`) are marked **[spike]**.
- **Date:** 2026-11-19

## Context

Several local projects need calendar data. `meal-planner` reads it through
`native/ekreader`, a signed Swift helper on macOS EventKit. That path depends on
a TCC "Calendars" grant, which OS upgrades void (macOS 27.2 did; the planner
degraded silently for weeks). Every new EventKit consumer would inherit the same
problem.

All calendars live in iCloud under one Apple ID. iCloud exposes them over
CalDAV (`caldav.icloud.com`) with an app-specific password. That password
**cannot be scoped**: it grants full read/write to every calendar and to
contacts. So any access control has to be enforced by whatever holds it.

Settled by the brief: iCloud CalDAV is the source; access is read-only first
with a designed-in path to CRUD; it is self-hosted on the LAN (R630 preferred,
Mac container as fallback); we build it rather than adopt `ox-mcp`.

## Decision

### D1. Shape: one broker process, the only holder of the iCloud credential

```
 agents / bots ──bearer token──▶ ┌──────────────── broker ────────────────┐
 (MCP clients)                   │ auth+scope ─▶ tools ─▶ query engine     │
                                 │                         │  (expand RRULE)│
 meal-planner ──bearer token──▶  │                         ▼               │
                                 │ /health ◀── sync loop ─▶ SQLite cache   │
                                 └───────────────│─────────────────────────┘
                                                 ▼ app-specific password
                                         caldav.icloud.com
```

- Clients never see the app-specific password. A leaked client token gives at
  most that client's scopes and calendars, and it can be revoked alone.
- The broker never touches CardDAV/contacts.
- The core (sync, cache, query, recurrence expansion) knows nothing about MCP.
  Transports are thin adapters over it, which keeps D2's REST question cheap
  to answer either way.

### D2. Transport: MCP over Streamable HTTP — **decided 2026-11-19: MCP-only**

The official `@modelcontextprotocol/sdk` server with the Streamable HTTP
transport, in **stateless** mode (no session affinity; each request carries its
bearer token). That serves LLM agents directly.

Open question: deterministic consumers like meal-planner can either
(a) use the MCP client SDK and call tools (meal-planner already depends on the
SDK), or (b) call a small REST/JSON endpoint (`GET /v1/events?...`) that runs
the same core with the same auth.
**Decided: (a) MCP-only for v1.** One surface to secure and test. Tools
return `structuredContent` with a declared `outputSchema`, so programmatic
callers get typed JSON, not prose. Add REST later only if a consumer can't
reasonably speak MCP.

### D3. Auth and scopes — **decided 2026-11-23**

- **Per-client static bearer tokens**, not MCP's OAuth 2.1 flow. Every client
  is a headless bot on the LAN; an OAuth authorization server adds a lot of
  moving parts and no security for that case. The SDK's bearer-auth hook lets
  us adopt OAuth later without changing tools.
- Tokens are random 256-bit values. The broker stores only a **SHA-256 hash**
  per client, in a client registry file
  (`clients.json`: `id`, `tokenHash`, `scopes`, `calendars`). The plaintext
  lives in each client's own 1Password item. Comparison is constant-time.
- **Scopes:** `calendar:read`, and later `calendar:write` (which implies read).
  Every tool declares the scope it needs, and the tool list a client sees is
  filtered to what it holds. A write tool simply doesn't exist for a read-only
  client.
- **Calendar allowlist (optional, per client):** if set, the client can't see
  calendars outside it at all, in `list_calendars` or any query. It is a hard
  filter applied in the core, not in each tool.
  Open sub-question: key the allowlist by **calendar ID** (the stable CalDAV
  collection URL; renaming a calendar doesn't widen or break access) or by
  **display name** (what meal-planner's config already uses).
  **Decided:** store IDs. The config tool accepts names and resolves
  them to IDs once, and `list_calendars` shows both.
- Unauthenticated: only `/health`, which reveals no calendar names or content.

### D4. Event schema — **decided 2026-11-23 (with D5)**

One JSON shape returned by every read tool. Each **occurrence** of a recurring
event is a separate item, so consumers never see RRULEs unless they ask.

```ts
interface BrokerEvent {
  // identity
  calendarId: string;          // stable collection id (URL path)
  calendarName: string;        // display name at time of read
  uid: string;                 // iCalendar UID (series id)
  recurrenceId: string | null; // ISO instant of the original occurrence; null for non-recurring
  // when
  allDay: boolean;
  start: string;               // allDay: "YYYY-MM-DD"; else ISO-8601 with offset, e.g. "2026-11-19T18:00:00-06:00"
  end: string;                 // exclusive; same format as start
  timeZone: string | null;     // IANA TZID of the source event, null for floating/all-day
  // what
  title: string;               // SUMMARY, "" if absent
  status: "confirmed" | "tentative" | "canceled"; // missing STATUS -> "confirmed"
  location: string | null;
  description: string | null;
  url: string | null;
  // write-pathway handles (present now so v2 is additive)
  href: string;                // resource URL of the containing .ics
  etag: string;                // for If-Match on update/delete
}
```

- **meal-planner mapping** is mechanical: `start/end -> Date`; all-day dates
  are interpreted in the reference timezone (D6); the rest map 1:1.
- Attendees/organizer/alarms/attachments are left out of v1. They are
  additive later and are the riskiest fields for privacy.
- `etag` belongs to the whole resource, so every occurrence of a series shares
  it. That is correct: an If-Match write replaces the whole resource.

### D5. Query semantics

- `list_events({ start, end, calendarIds?, calendarNames?, includeCanceled? })`:
  returns occurrences that **overlap** `[start, end)`. An event with
  `end > start_q && start < end_q` is included, so in-progress events count.
  All-day events overlap the dates they cover in the reference timezone.
- Canceled occurrences are **included by default** (meal-planner treats status
  as data, not a filter). An EXDATE'd occurrence doesn't exist at all; it is
  not reported as canceled.
- Results are sorted by `start`, then `calendarId`, then `uid`. A window wider
  than a configured maximum (default 366 days) is rejected.
- Every response carries `asOf` (time of the last successful sync covering the
  returned calendars) and `stale: boolean`. Freshness is visible per call, not
  only on `/health`.

### D6. Time zones — **decided 2026-11-19: `America/New_York`**

All-day and floating events have no zone, so the broker needs a **reference
timezone** to place them on the timeline for overlap queries and Date
conversion. **Decided:** one broker-wide IANA zone in config, default
`America/New_York`, with no per-client override in v1. Zoned events are
converted using their own VTIMEZONE (registered with ical.js per resource),
falling back to the IANA database by TZID. Output offsets are in the event's
own zone.

### D7. Recurrence — **[spike]** client-side expansion

- Expand **in the broker** with `ical.js`: RRULE/RDATE, EXDATE, and
  RECURRENCE-ID overrides (moved instances, per-instance status changes such as
  a canceled occurrence), honoring each resource's VTIMEZONE.
- We don't rely on CalDAV `<C:expand>`. iCloud support for it is unverified and
  reported to be inconsistent. The spike will compare it against our own
  expansion, but the cache stores **raw resources** and expands at query time
  either way, so arbitrary windows work with no re-sync.
- Correctness is pinned by a fixture suite (synthetic `.ics`) covering weekly
  with EXDATE, moved instance, canceled instance, `THISANDFUTURE` splits, DST
  transitions, all-day multi-day, floating times, and `UNTIL`/`COUNT`.

### D8. Cache and sync — **open: `decision` bead "sync cadence and retention"**

- **SQLite** cache (`node:sqlite`, built into Node ≥ 22.13, so there is no
  native module to compile; fall back to `better-sqlite3` if it misbehaves).
  Tables: `calendars` (id, name, color, ctag, sync_token, last_synced_at,
  last_error) and `resources` (calendar_id, href, etag, ics, uid). It holds raw
  iCalendar; expansion happens on read.
- **Sync loop:** every *N* minutes per calendar, check the `getctag`. If it
  changed, run `sync-collection` (RFC 6578) with the stored sync-token. If the
  token is rejected, do a full ETag diff (`calendar-query` hrefs+etags, then
  multiget only what changed). `tsdav` covers all three. **[spike]**: confirm
  which ones iCloud honors.
- Recommendation: *N* = 5 minutes, with jitter, exponential backoff on 429/5xx
  and a hard floor of 1 minute. Keep **all** resources (calendars are small).
  Expansion is bounded by the query window, not by storage.
- If iCloud is unreachable, reads keep serving the cache, with `stale: true`.
- Auth failure (revoked app-specific password, 401) is its own state. Retrying
  won't fix it, so it goes straight to `/health` as `auth_failed`.

### D9. Observability — **decided 2026-11-19: healthchecks.io push**

- `GET /health`: `200 {status:"ok", lastSuccessfulSyncAt, ageSeconds}`, or
  `503` with `status: "stale" | "auth_failed" | "starting"` once the age passes
  a threshold. It is meant for an external uptime checker that pages a human,
  because a log line paged nobody in meal-planner.
- Recommendation: stale threshold = 30 min (6 missed syncs). Structured JSON
  logs to stdout. Never log tokens, passwords, or event content beyond
  ids/counts.
- **Paging: healthchecks.io dead-man's switch.** After each successful full
  sync pass the broker pings a healthchecks.io check URL (kept in 1Password).
  If the pings stop, for any reason (stale sync, `auth_failed`, crashed
  broker, host down), healthchecks.io alerts a human. On `auth_failed` the
  broker also pings the check's `/fail` endpoint so the alert fires at once.
  The ping carries no calendar data. `/health` stays as the local diagnostic.

### D10. Secrets

The app-specific password and Apple ID come from a 1Password service account
(`op run` with `op://` refs in the deploy env file). The client registry holds
only token hashes, so it isn't secret. Startup fails fast and loudly if the
credential is empty, so a missing secret is never treated as a sync error.

### D11. Hosting — **decided 2026-11-19: R630 container, plain HTTP on the LAN**

- A single OCI container (Node 22 slim), with the SQLite file on a volume.
  Run it on the **R630** (preferred, no macOS dependency) with a restart
  policy. The same image runs on the Mac container host as fallback.
- **Transport: plain HTTP on the trusted LAN.** Accepted risk: anything on
  the LAN segment can sniff a client token. Mitigations: tokens are scoped
  and read-only, each one can be revoked alone, and the broker is not
  exposed beyond the LAN (no port forward). Revisit TLS (Caddy + internal CA)
  or Tailscale **before `calendar:write` ships** (D12).

### D12. CRUD pathway (not built in v1)

Adding writes is additive, with no architectural change:

- New tools `create_event`, `update_event`, `delete_event` that require
  `calendar:write` and respect the calendar allowlist.
- Every update/delete sends **`If-Match: <etag>`**. A 412 comes back to the
  caller as a conflict with the current version; there is no blind overwrite.
  Creates use `If-None-Match: *` and a broker-generated UUID UID.
- Writes go **straight to iCloud**, then trigger an immediate sync of that
  calendar. The cache is never a source of truth.
- An **append-only audit log** (SQLite table: who, tool, calendar, uid, before
  etag, after etag, outcome) plus a `dryRun` flag that returns the iCalendar
  that would be sent.
- Recurring-series edits (this instance, this and future, all) need their own
  ADR before write tools ship.

## Consequences

- **Good:** no TCC, signing, or macOS in the calendar path; one place to
  enforce read-only; fast cached reads; staleness is visible and pageable.
- **Cost:** we own recurrence/timezone correctness. That is the largest risk,
  and the fixture suite in D7 is how we contain it.
- **Cost:** another long-running service to deploy and monitor.
- **Accepted risk:** client tokens cross the LAN in cleartext (D11). That is
  tolerable for read-only scopes, and must be revisited before write.
- **Risk:** iCloud CalDAV behavior (shared and subscribed calendars, sync-token
  support, rate limits) is undocumented. The spike retires these before the
  cache design is frozen.

## Alternatives considered

- **Keep EventKit (`ekreader`) per project:** rejected; the TCC fragility is
  the reason this project exists.
- **Adopt `Rheopyrin/ox-mcp`:** rejected; it is mail-first, immature,
  single-user, with no per-client auth or scoping. It may be read for CalDAV
  reference.
- **Give each client the app-specific password and a CalDAV library:**
  rejected; it can't be scoped, so every bot would hold full read/write to
  calendars and contacts.
- **No cache (proxy every read to iCloud):** rejected; it is slower, risks rate
  limits, and is unavailable whenever iCloud is.
