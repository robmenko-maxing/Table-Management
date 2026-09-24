# Table Management: Project Plan

Status: Draft v0.1 (discovery complete, awaiting SDLC reconciliation with Flightdeck AGENTS.md)
Owner: Rob Menko
Planning and UX: Fable. Implementation: Opus agents (xhigh effort). Every Opus deliverable gets an adversarial code review before merge.

## 1. Purpose

A Rails app that helps Cru staff run the seating side of fundraising events (banquets, dinners, vision nights). It answers three questions on event night: who is coming, where do they sit, and have they arrived.

## 2. Discovery answers

| Topic | Decision |
| --- | --- |
| Users | Event staff/admins, table hosts (captains), door check-in volunteers. Guests do not log in. |
| Guest data source | A separate tool still in development. We pull from its API on a schedule and on demand. |
| Event size | Medium: 15 to 60 tables, roughly 100 to 500 guests. Several events per year, possibly several ministries or cities. |
| Sign-in | Cru SSO (Okta / The Key) via OpenID Connect. |
| Seating UX | Table board with drag-and-drop. Tables shown as cards, guests dragged between them. No room geometry in v1. |
| Stack | Rails 8, Hotwire (Turbo, Stimulus, Turbo Streams), Tailwind, Postgres, Solid Queue/Cache/Cable. |
| Infrastructure | Terrabloks defaults. Document whatever it produces. |
| Project tracking | Flightdeck. Epics and stories below are written to load straight into it. |

## 3. Personas and jobs

**Event coordinator (staff/admin).** Builds the event, sets up tables and hosts, watches RSVP and seating progress, fixes problems fast. Works on a laptop, often the week before and the afternoon of the event.

**Table host (captain).** A volunteer or donor who fills one table with their own invitees. Wants to see their table, who has said yes, and open seats. Uses the app a few times a year on a phone, so it must be simple and forgiving.

**Check-in volunteer.** Stands at the door with a phone or tablet. Needs to find a guest in two taps and tell them their table number. Big text, fast search, works with a spotty connection.

## 4. Key screens (UX plan)

1. **Events list.** Upcoming and past events, one card each with date, venue, and a progress strip (invited, yes, seated, checked in).
2. **Event overview.** Counts and warnings: unseated confirmed guests, tables over capacity, tables with no host, guests with dietary notes, last sync time.
3. **Seating board.** The core screen. A horizontal board of table cards (Kanban style). Each card shows table name, host, capacity meter, and guest chips. An "Unseated" column on the left. Drag a guest chip to a table. Live updates to everyone on the page through Turbo Streams. Filter by RSVP, party, or search. Keyboard fallback: select a guest, pick a table.
4. **Guest list.** Searchable table with RSVP status, party/household, table, dietary notes, source ID, and sync status. Bulk assign and bulk unassign. Manual add and edit for walk-ins.
5. **Table hosts portal.** A host sees only their own table(s): guests, RSVP status, open seats, plus a way to add or remove their invitees (if the source tool allows it; otherwise read-only with a note).
6. **Check-in mode.** Full-screen, mobile-first. One search box, large result rows, tap to check in, shows the table number in very large type. Undo within a few seconds. Shows a running count.
7. **Print and export.** Table list by table, alphabetical guest list with table numbers, place cards, and a CSV export.
8. **Admin.** Organization members and roles, event archiving, integration settings for the source tool, sync history.

Design principles: mobile first for check-in and hosts, desktop first for the seating board. Plain language, high contrast, no hover-only actions. Every destructive action is undoable or confirmed.

## 5. Domain model (first cut)

```
Organization        name, slug
User                okta_uid, email, name           (from The Key)
Membership          user, organization, role        role: admin | staff | host | checkin
Event               organization, name, starts_at, venue, status, settings(jsonb)
Table               event, name, number, capacity, sort_order, host_guest_id
Guest               event, external_id, first_name, last_name, email, phone,
                    rsvp_status, party_key, dietary_notes, notes,
                    is_host, checked_in_at, checked_in_by_id, source
SeatAssignment      guest (unique), table, seat_number, assigned_by_id
HostAccess          table, user                     (lets a host log in and see their table)
SyncRun             event, source, started_at, finished_at, status, stats(jsonb), error
```

Rules the model must enforce:

- A guest has at most one seat assignment per event.
- A table cannot exceed capacity unless an admin overrides (the board shows the warning either way).
- Guests sharing a party_key are flagged when split across tables.
- Deleting a guest is soft (discarded_at) so sync can reconcile.
- All queries are scoped to the current organization.

## 6. Integration with the source tool (pull model)

Since the source tool is still being built, we isolate it behind one boundary:

- `Sync::Source` interface: `fetch_guests(event_external_id, since:)` returns normalized guest records.
- One adapter per source. First adapter is `Sync::CsvSource` so staff can work before the API exists. Second adapter is `Sync::RemoteApiSource` once the tool exposes an endpoint.
- `Sync::Reconciler` upserts by `external_id`, never overwrites local-only fields (table, check-in, notes), and records changes in `SyncRun.stats`.
- Runs on a Solid Queue schedule (every 15 minutes during the week of the event, hourly otherwise) and on demand from the event overview.

Open items to settle with that tool's owners: auth method, pagination, how RSVP status is represented, and whether hosts add guests there or here.

## 7. Milestones and epics (for Flightdeck)

**M0 Foundation**
- Rails 8 app skeleton, Postgres, Tailwind, Hotwire, Solid Queue/Cache/Cable.
- Terrabloks infrastructure (app, database, secrets, TLS, CI deploy).
- Okta / The Key OIDC sign-in, Organization, Membership, roles, authorization policy layer.
- CI: RuboCop, Brakeman, bundler-audit, system tests, adversarial review gate.

**M1 Events and guests**
- Event CRUD, Table CRUD, Guest CRUD, soft delete.
- CSV import through the Sync boundary. Sync history page.
- Event overview counts and warnings.

**M2 Seating board**
- Kanban board, drag-and-drop (Stimulus + SortableJS), Turbo Stream broadcasts.
- Capacity and party warnings. Keyboard fallback. Bulk assign.

**M3 Check-in mode**
- Mobile-first search and check-in, undo, live counts, works on flaky connections (optimistic UI, retry).

**M4 Host portal**
- HostAccess, restricted views, host invite email (magic link into The Key flow if possible).

**M5 Remote sync adapter**
- `Sync::RemoteApiSource`, scheduled runs, conflict rules, alerts on failed runs.

**M6 Print, export, polish**
- PDF table lists and place cards, CSV export, accessibility pass, performance pass.

## 8. SDLC (to be reconciled with flightdeck/AGENTS.md)

The intended process, pending the AGENTS.md file which was not reachable from this session:

1. **Plan (Fable).** Story written in Flightdeck with acceptance criteria and UX notes. UI stories include a wireframe description or mock.
2. **Implement (Opus, xhigh effort).** One agent per story on a feature branch. Tests required. No story merges without passing CI.
3. **Adversarial review (Fable).** A second agent reviews with a red-team prompt: find bugs, missing authorization, N+1 queries, race conditions, and untested paths. Findings go back to the implementer. Repeat until clean.
4. **Merge and track.** Story moved to done in Flightdeck with the PR link and review summary.
5. **Deploy.** Terrabloks-managed pipeline to staging, then production after smoke test.

## 9. Open questions

- What is the source tool called, and when will its API be available for a staging test?
- Do table hosts have The Key accounts, or do we need a separate invite flow?
- Should check-in work fully offline (service worker) or is a "retry when back online" queue enough?
- Are place cards and printed table lists needed for the first event, or can they wait for M6?
- Does any ministry need more than one organization in v1, or is one org enough for launch?

## 10. Blockers noted during planning

- The Terrabloks and Flightdeck connectors exist in the org but are not connected to this session.
- The referenced `~/htdocs/flightdeck/AGENTS.md` is not in this cloud container and no flightdeck repo is attached.
- A `/frontend-design` skill was requested but is not installed; the UX plan above was done directly.
