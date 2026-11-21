# Feature: Racket Sports & General Fitness Tracker (WitnessTheFitness)

## Overview

A personal Laravel application (built with Laravel Boost agentic AI rules) that automatically tracks racket sports
play (tennis, padel, and other racket sports added later) and, in a later phase, combines it with wearable health
data (heart rate, sleep, exercise) into one place with graphical statistics. It replaces a manually-maintained
spreadsheet that was abandoned because keeping it up to date took too much ongoing effort — the core value of this
app is that session data is captured **automatically**, with no manual logging required for day-to-day use.

This is a single-user, personal tool. Architecture and technology choices beyond "Laravel + Laravel Boost" are
explicitly out of scope for this document.

## Phasing

- **Phase 1** (this spec's primary focus): automated capture of racket sports sessions (bookings, cancellations,
  player changes) from Gmail/Playtomic/David Lloyd, including scores where Playtomic's own API already provides
  them, plus a statistics dashboard (including win/loss where score data exists).
- **Phase 2** (lighter/placeholder requirements below): integrate Matchspace (box leagues, club championships) to
  enrich sessions with richer score/result data, correlating Matchspace results against David Lloyd bookings.
  Access method (API vs email vs manual/screenshot ingestion) is still to be researched.
- **Phase 3** (lighter/placeholder requirements below): ingest wearable device data (heart rate, sleep, exercise)
  and present it alongside Phase 1/2 data. Details are pending — the user will supply the wearable brand/app and
  an example historical spreadsheet; Phase 3 requirements should be revisited once that information is available.

## User Stories

- As the player, I want my court bookings to be captured automatically from confirmation emails/APIs, so I don't
  have to manually log every session like I did with my old spreadsheet.
- As the player, I want cancellations to be detected and excluded automatically, so my stats aren't inflated by
  sessions that never happened.
- As the player, I want changes to who's playing (added/removed players) to be reflected, so "who I played with"
  stays accurate.
- As the player, I want a dashboard showing how much I play, where, and with whom, and how that trends over time,
  so I can see patterns without maintaining a spreadsheet.
- As the player, I want to review and correct anything the system wasn't sure about, so bad data doesn't silently
  pollute my history.
- As the player, I want my historical spreadsheet data imported, so I don't lose the record I already built up.

## Functional Requirements

### Session Capture

> **Research findings (API feasibility):** Investigated whether Playtomic and David Lloyd expose APIs an
> individual member could use, beyond email parsing.
> - **Playtomic** does publish an official third-party API (`thirdparty.playtomic.io`), but it is **club/venue-scoped**:
>   credentials are issued to a venue via Playtomic Manager (Settings → Developer Tools), not to an individual
>   player, and booking history is capped at the last 3 months with a ~1 call/minute rate limit
>   ([Playtomic API guide](https://helpmanager.playtomic.com/hc/en-gb/articles/38836515997073)). As a club
>   member, the user has no route to these credentials. Separately, developers have reverse-engineered the
>   *mobile app's* traffic (via mitmproxy) to find **booking/payment** endpoints
>   (`POST /v1/payment_intents`, `PATCH /v1/matches/{id}/registrations`, etc. —
>   [write-up](https://dev.to/hadriel33/how-i-reverse-engineered-playtomics-mobile-payment-api-to-build-a-padel-court-booking-bot-5ehb)),
>   but no source found documents a "my match history" or score endpoint reachable this way, and
>   authentication for that player-facing flow is undocumented in what was found. A `lingolux/playtomic`
>   GitHub repo turned up in search is unrelated (a different "Playtomic" leaderboard project), not a usable
>   client.
> - **David Lloyd**: no public API, developer program, or reverse-engineered client/scraper was found anywhere
>   (GitHub search, app-traffic write-ups, forums). Booking is app-only per David Lloyd's own help pages.
> - **Conclusion**: for Phase 1, Gmail email parsing is the only *confirmed* capture mechanism for both
>   Playtomic and David Lloyd. FR-CAPTURE-002 (API-first) and FR-SCORE-001 (Playtomic API score capture) remain
>   in this spec as the better path *if* a viable player-level endpoint is found, but should be treated as
>   unconfirmed/stretch until someone captures their own Playtomic app traffic (login + match-history screens)
>   to check — see Open Questions.

**FR-CAPTURE-001**: Email-based booking detection
While Gmail access is connected, when a new booking confirmation email arrives from a recognized venue/platform
(e.g. Playtomic, David Lloyd), the system shall parse it and create a session record (sport, date, time, duration,
venue, players) with status `pending_review` or `confirmed` depending on parse confidence.

**FR-CAPTURE-002**: API-first capture where available
Where a player-accessible API is confirmed available for a venue or platform, the system shall prefer that API as
the source of booking data for that venue over parsing its emails. As of this spec, no such player-accessible API
has been confirmed for either Playtomic (its official API is club/venue-scoped, not member-accessible) or David
Lloyd (none found) — see the Research findings note above — so this requirement currently has no venue it applies
to in practice and is retained for if/when one is confirmed.

**FR-CAPTURE-003**: Email fallback (Phase 1's actual sole mechanism)
The system shall rely on Gmail email parsing to capture bookings for any venue without a confirmed
player-accessible API — which, per current research, means Gmail parsing is the mechanism for both Playtomic and
David Lloyd, not a fallback used only occasionally.

**FR-CAPTURE-004**: Confidence-based auto-confirmation
When a parsed booking (from email or API) has all required fields recognized with high confidence (sport, date,
time, duration, venue, known players), the system shall mark it `confirmed` automatically without requiring manual
review.

**FR-CAPTURE-005**: Low-confidence routing
When a parsed booking has an ambiguous or missing field (e.g. unrecognized player name, unclear duration or
cancellation status), the system shall mark it `pending_review` and add it to the review queue instead of
auto-confirming it.

### Cancellations & Player Changes

**FR-CANCEL-001**: Cancellation detection
When a cancellation email or API event is received that references a previously captured session, the system
shall mark that session as `cancelled` and exclude it from all statistics.

**FR-CANCEL-002**: Cancellation of a past session
While a session's date/time is in the past and it is marked `confirmed`, when a cancellation reference for it is
received, the system shall route it to the review queue rather than silently cancelling it, since a cancellation
notice for a session that already happened is itself ambiguous.

**FR-PLAYER-001**: Player roster updates
When a booking-modification email/event changes the list of players for an existing session, the system shall
update that session's player roster to the latest known state and retain the prior roster in history (not silently
overwritten/lost).

**FR-PLAYER-002**: Player identity matching
When parsing player names from a booking or modification email, the system shall match each name against a known
players list (including aliases); any name that cannot be confidently matched shall cause the session to be routed
to the review queue (per FR-CAPTURE-005) so the user can map it to an existing player or create a new one.

### Scores & Results

**FR-SCORE-001**: Where a post-event score is available for a booking — whether from a (currently unconfirmed)
Playtomic API, or parsed from a post-match result email/notification Playtomic sends the player — the system
shall store that score against the corresponding session. Which of these sources actually exists and what it
returns is unconfirmed (see Research findings above and Open Questions) and needs checking before this is built.

**FR-SCORE-002**: The system shall display the recorded score, where available, on a session's detail view.

**FR-SCORE-003**: Where no score is available for a session, the system shall display it without a score rather
than treating the absence of a score as an error.

### Review Queue

**FR-REVIEW-001**: Review queue listing
The system shall provide an in-app review queue listing every session in `pending_review` status, showing the
parsed data and the reason it needs review (e.g. "unmatched player: Dave", "ambiguous cancellation").

**FR-REVIEW-002**: Review resolution
While a session is in the review queue, when the user confirms, corrects, or discards it, the system shall update
the session's status accordingly (`confirmed`, `confirmed` with corrected data, or `discarded`) and remove it from
the queue.

**FR-REVIEW-003**: No other notification
The system shall surface pending review items only within the in-app queue; it shall not send emails or other
notifications about them.

### Statistics & Dashboard

**FR-STATS-001**: Sessions & hours
The system shall display total sessions played and total hours played, broken down by sport and filterable by
time period (week, month, year, all-time).

**FR-STATS-002**: Venue breakdown
The system shall display a breakdown of sessions and hours played per venue/location.

**FR-STATS-003**: Partner breakdown
The system shall display a breakdown of sessions and hours played with each playing partner.

**FR-STATS-004**: Trends
The system shall display a chart of play frequency/duration over time and shall surface streak information (e.g.
longest run of consecutive weeks played, longest gap without playing).

**FR-STATS-005**: Only confirmed sessions counted
The system shall exclude sessions with status `cancelled`, `pending_review`, or `discarded` from all statistics
and charts; only `confirmed` sessions shall be counted.

**FR-STATS-006**: Win/loss record
Where sessions have a recorded score, the system shall calculate and display a win/loss record, overall and
broken down by partner, venue, and sport; sessions without a recorded score shall be excluded from win/loss
calculations without affecting the other (non-score) statistics.

**FR-STATS-007**: Friendly vs Competitive breakdown
The system shall allow filtering and breakdown of sessions and stats (FR-STATS-001 to FR-STATS-004, FR-STATS-006)
by the Friendly/Competitive flag (FR-TYPE-001).

### Session Type (Friendly / Competitive)

**FR-TYPE-001**: The system shall record a Friendly/Competitive flag against every session.

**FR-TYPE-002**: When a session originates from a box league or club championship context (e.g. Matchspace, once
integrated in Phase 2), the system shall default its flag to Competitive.

**FR-TYPE-003**: When a session originates from a regular booking (Gmail/Playtomic/David Lloyd) with no league or
championship context, the system shall default its flag to Friendly.

**FR-TYPE-004**: The user shall be able to override the Friendly/Competitive flag on any session, via the review
queue or the session detail view.

### Sport Extensibility

**FR-SPORT-001**: Open-ended sport field
The system shall store `sport` as a data value rather than a hardcoded enum/type, so a new racket sport (e.g.
squash, badminton) can be recorded without a schema change.

### Historical Backfill

**FR-BACKFILL-001**: Spreadsheet import
The system shall support a one-off import of the user's existing spreadsheet into session history, mapped to the
same session model used for ongoing capture. The source format (`court_time_tracker.xlsx`, single "Court Time"
sheet, one row per session, blank rows as week separators) maps as follows:

| Spreadsheet column | Session field |
|---|---|
| `Date` | session date (`Day` is derivable from it and not stored separately) |
| `Activity` | sport |
| `Duration (hrs)` | duration |
| `Venue` | venue |
| `Playing Partners` | partner names, parsed into individual players where real names are given |
| `Notes` | free-text note; also the source for score extraction (FR-BACKFILL-003) |
| `Week Of` | not stored; it is a derived grouping, not session data |

**FR-BACKFILL-002**: Group/drop-in session type
Where `Playing Partners` contains a non-person value (e.g. "Social session", "TBC (intermediate group)",
"Intermediate group") rather than named individuals, the system shall import the row as a `group` session — a
distinct type from a named-partner session — contributing to sport/venue/time-based stats but not to any
individual partner breakdown, with the original text retained as a note. The same `group` type shall be available
for ongoing (non-backfill) capture where a booking has no identifiable individual partners.

**FR-BACKFILL-003**: Score extraction from historical notes
During spreadsheet import, when a `Notes` value matches a recognizable result pattern (e.g. "Won 7-6", "Lost 4-6
6-3"), the system shall extract it into the session's score field (FR-SCORE-001) and set the Friendly/Competitive
flag to Competitive where the note indicates a league/championship match (e.g. "Box league match"). Where the
pattern is ambiguous, the system shall leave the note as plain text and route the session to the review queue for
manual confirmation rather than guessing.

**FR-BACKFILL-004**: Historical email scan
The system shall support an on-demand scan of historical Gmail messages to retroactively capture past bookings,
and shall detect and flag likely duplicates against already-imported spreadsheet sessions (same date/venue/players)
for review rather than double-counting them.

### Phase 2 (placeholder — Matchspace integration; access method still to be researched)

**FR-MATCHSPACE-001** (tentative): The system shall ingest box league and club championship results from
Matchspace. The access method is unresearched — candidates are an API, email notifications, or (worst case)
manual screenshot upload with the system extracting players/score from the image. The chosen method shall be
confirmed before implementation.

**FR-MATCHSPACE-002** (tentative): Where a Matchspace result correlates with an existing David-Lloyd-sourced
session (matching date, venue, and players), the system shall enrich that session with the Matchspace score
rather than creating a duplicate session.

**FR-MATCHSPACE-003** (tentative): Where no matching booking session exists for a Matchspace result, the system
shall create a new session sourced from Matchspace alone, with its Friendly/Competitive flag defaulted to
Competitive (per FR-TYPE-002).

### Phase 3 (placeholder — wearable integration; device: Xiaomi Smart Band 10, app: Mi Fitness, Android)

Mi Fitness has no official public API. Data only leaves it via the phone's platform health store once sync is
enabled in the app — on Android (the user's platform) that is **Health Connect**; on iPhone it would be Apple
Health. This means Phase 3 access is indirect: Mi Fitness → Health Connect → whatever reads Health Connect.
Note: some Xiaomi bands measure HRV but Xiaomi's own policy blocks HRV from being shared out via this route, so
HRV may not be available even if everything else works.

The user switches the band/app into a sport-specific exercise mode (e.g. "Tennis", "Padel") before playing, so
each recorded exercise entry should already carry a matching sport label and a start/end time close to the
actual session — see FR-WEARABLE-003/004 for how this is used to link wearable data to racket-sport sessions.

**FR-WEARABLE-001** (tentative): While Mi Fitness is configured to sync to Health Connect, the system shall
ingest daily heart rate, sleep, and exercise summary data made available there, on an ongoing basis.

**FR-WEARABLE-002** (tentative): The system shall display wearable-derived fitness metrics alongside racket-sport
session statistics on a combined view.

**FR-WEARABLE-003** (tentative): The user manually switches Mi Fitness into a specific exercise mode (e.g.
"Tennis", "Padel"/training mode) when playing, so each wearable exercise entry carries both a start/end time and
an exercise-type label matching the sport. Where a wearable exercise entry's type matches a confirmed
racket-sport session's sport and its time range overlaps that session, the system shall treat it as a high-
confidence match and link them automatically (rather than requiring manual confirmation), so effort/heart-rate
data can be viewed per session.

**FR-WEARABLE-004** (tentative): Where a wearable exercise entry's time overlaps a confirmed session but its
exercise-type label does not match the session's sport (or is a generic/unlabelled workout), the system shall
still offer it as a candidate link but require manual confirmation rather than auto-linking.

**FR-WEARABLE-005** (tentative): Where HRV or other data is unavailable via Health Connect due to Xiaomi's
sync restrictions, the system shall omit that metric rather than erroring.

## Non-Functional Requirements

### Performance
- Personal-scale data volume (expected low hundreds of sessions/year); no high-throughput requirements.
- Email/API polling and parsing may run as background/queued jobs; the dashboard itself should load without
  waiting on live Gmail/API calls.

### Security & Access
- Single-user application; the app itself shall still require authentication to view any data (specific
  mechanism is an architecture decision, out of scope here).
- Gmail access shall use the minimum scope/query needed to read booking-related emails (read-only; not full
  mailbox access where avoidable).
- Phase 2 health data (heart rate, sleep) is sensitive personal data; it shall not be exposed to or shared with
  any third party, and access shall be subject to the same authentication as the rest of the app.

### Data Retention
- Session history (confirmed, cancelled, discarded) is retained indefinitely by default, as it forms the user's
  personal historical record — this was the original goal of the spreadsheet it replaces.
- The user shall be able to manually delete or correct any individual session.

### Reliability
- Venue email templates may change over time; the parser should fail safe — an unparseable email is routed to the
  review queue (or a dead-letter list) rather than silently dropped or mis-recorded.

## Acceptance Criteria

### AC-001: Booking email captured automatically
Given Gmail access is connected and a booking confirmation email arrives from a recognized venue
When the system processes it and all fields are recognized with high confidence
Then a `confirmed` session is created with the correct sport, date, time, duration, venue, and players
And it appears in the statistics without any manual action

### AC-002: Cancellation excludes a session from stats
Given a `confirmed` upcoming session exists
When a matching cancellation email/event is received before the session's date
Then the session is marked `cancelled`
And it no longer counts toward totals, trends, venue, or partner stats

### AC-003: Player change updates the roster
Given a `confirmed` session with an existing player roster
When a booking-modification email changes the players
Then the session's roster reflects the new players
And the prior roster remains visible in the session's history, not deleted

### AC-004: Ambiguous parse goes to review, not silently recorded or dropped
Given a booking email contains a player name that doesn't match any known player
When the system parses it
Then the session is created with status `pending_review`
And it appears in the review queue with the reason "unmatched player"
And it is excluded from statistics until resolved

### AC-005: Resolving a review item
Given a session is in the review queue
When the user confirms it with a corrected player mapping
Then the session status becomes `confirmed`
And it is removed from the review queue
And it is included in statistics going forward

### AC-006: Cancellation of an already-played session is not silently applied
Given a `confirmed` session whose date/time is in the past
When a cancellation reference for that session is received
Then the session is routed to the review queue (not auto-cancelled)
And the reason shown is "cancellation received for past session"

### AC-007: Spreadsheet backfill import
Given the user uploads/imports their historical spreadsheet
When the import runs
Then a `confirmed` session is created for each valid row (date, duration, venue, partners mapped to players)
And rows that can't be mapped cleanly are reported to the user rather than imported incorrectly

### AC-008: Historical email scan avoids double-counting
Given the spreadsheet has already been imported for a date range
When a historical Gmail scan finds a booking email for a session already present from the import
Then the scanned result is flagged as a likely duplicate for review rather than creating a second session

### AC-009: Dashboard filtering
Given confirmed sessions exist across multiple sports, venues, and partners
When the user filters the dashboard by time period, sport, venue, or partner
Then totals, breakdowns, and trend charts update to reflect only the matching sessions

### AC-010: Win/loss record from a scored session
Given a confirmed Playtomic session has a post-event score recorded
When the dashboard calculates win/loss record
Then that session is included in the overall, partner, venue, and sport win/loss breakdowns
And a confirmed session with no recorded score is excluded from win/loss stats but still counted in
sessions/hours/venue/partner stats

### AC-011: Friendly/Competitive default and override
Given a session is created from a regular David Lloyd booking with no league/championship context
When the session is saved
Then its flag defaults to Friendly
And the user can change it to Competitive from the session detail or review queue, after which dashboard
breakdowns by Friendly/Competitive reflect the change

## Error Handling

| Error Condition | Handling |
|---|---|
| Unparseable/unrecognized email format | Routed to review queue (or dead-letter list) with raw email reference; not dropped, not mis-recorded |
| Gmail access/token expired or revoked | Background capture pauses; dashboard shows a clear "Gmail not connected" state until reauthorized |
| Venue API unavailable/rate-limited | Falls back to email parsing for that venue (per FR-CAPTURE-003) where possible; otherwise queues retry |
| Duplicate booking detected (API + email, or backfill + live capture) | Flagged for review rather than auto-merged or double-counted |
| Unmatched player name | Session routed to review queue (FR-PLAYER-002); user maps to existing or new player |
| Cancellation referencing unknown/already-deleted session | Logged and surfaced in review queue as an orphaned cancellation, not silently ignored |
| Spreadsheet import row with missing/invalid data | Row skipped and reported in an import summary, not silently imported with blanks |

## Implementation TODO

### Backend
- [ ] Define `Session`, `Player`, `Venue` (and alias/match) data models and migrations, with `sport` as a free
      value, a nullable `score` field, and a Friendly/Competitive flag
- [ ] Session status lifecycle: `pending_review` → `confirmed` / `discarded`; `confirmed` → `cancelled`
- [ ] Friendly/Competitive default logic (booking source vs league/championship source) + manual override
- [ ] Gmail connection (MCP) + read-only query/polling for booking-related emails
- [ ] Per-venue email parser(s) (starting with Playtomic, David Lloyd formats) with confidence scoring
- [ ] Playtomic API client (booking read) where available
- [ ] David Lloyd API client (booking read) if an API exists; otherwise document as email-only
- [ ] Cancellation matching logic against existing sessions
- [ ] Player-change/roster-update handling with history retention
- [ ] Review queue backend (list, confirm, correct, discard actions)
- [ ] Duplicate-detection logic (API vs email, backfill vs live, backfill vs historical scan)
- [ ] Spreadsheet import job per FR-BACKFILL-001/002/003 (column mapping, group-session handling, note-based
      score/competitive extraction with regex patterns for common result formats)
- [ ] Historical Gmail scan job (on-demand, bounded by date range)
- [ ] Stats aggregation queries (by sport/venue/partner/time period, trends, streaks, win/loss, Friendly vs
      Competitive)
- [ ] Simple authentication for app access
- [ ] Matchspace integration (Phase 2) — access method TBD; correlation logic against David Lloyd sessions

### Frontend
- [ ] Dashboard with filterable charts (sessions/hours, venue, partner, trends/streaks)
- [ ] Review queue UI (list + confirm/correct/discard actions)
- [ ] Session list/detail view (including roster history)
- [ ] Backfill/import flow UI (spreadsheet upload, historical scan trigger, import summary)

### Testing
- [ ] Parser unit tests per venue email template, using sample emails (booking, cancellation, modification)
- [ ] Cancellation flow tests (upcoming vs past session)
- [ ] Player-change/roster history tests
- [ ] Duplicate-detection tests (backfill vs email/API overlap)
- [ ] Stats aggregation correctness tests
- [ ] Spreadsheet import tests once format is known

## Out of Scope (for this spec)

- System/infrastructure architecture and technology choices beyond Laravel + Laravel Boost
- Multi-user accounts/login for other players (single-user only)
- Creating/modifying real bookings via any API (read-only capture only)
- League standings/rank tracking over time (only raw scores + win/loss record are in scope; standings were
  considered but not selected)
- Full Matchspace (Phase 2) implementation details — blocked on researching its access method
- Full Phase 3 wearable requirements — to be detailed once the wearable brand/app and its data-access options are
  known

## Open Questions

- [ ] Wearable is a Xiaomi Smart Band 10 via Mi Fitness on Android — confirmed there is no official Mi Fitness
      API; data only reaches Health Connect (on-device) once sync is enabled. Health Connect itself has no cloud
      REST API — reading it normally requires an Android app with the relevant permissions. How a server-side
      Laravel app gets this data (e.g. a small companion Android app/shortcut that exports and uploads it) is a
      real open question for whoever scopes Phase 3's architecture, not just a data-mapping detail.
- [x] Does David Lloyd expose any public/partner API? **Researched: no.** No public API, developer program, or
      reverse-engineered client/scraper was found (GitHub, app-traffic write-ups, forums). It is email-only in
      practice for Phase 1.
- [x] Does Playtomic expose a player-accessible API? **Researched: not confirmed.** Its official third-party API
      is club/venue-scoped (requires Playtomic Manager credentials issued to a venue, not a member) and limited
      to 3 months of history. Reverse-engineered mobile-app traffic write-ups document booking/payment endpoints
      but no documented "my match history" or score endpoint. Treat as email-only for Phase 1 like David Lloyd.
- [ ] Would capturing one's own Playtomic app traffic (e.g. with mitmproxy, as the reverse-engineering write-ups
      did) reveal a player match-history or score endpoint? Unresearched — would need the user to actually do
      this capture; carries some ToS risk the write-ups didn't address, so worth a deliberate decision before
      attempting it, and isn't required since Gmail parsing already covers Phase 1.
- [ ] Does Playtomic's post-match notification email contain the score, as an alternative route to FR-SCORE-001
      without needing any API? Unresearched — needs a real example email to check.
- [ ] Does Matchspace expose an API or send emails, or will results need manual screenshot upload + extraction?
      (needs researching — flagged by the user as unresearched; worst case is screenshot ingestion)
- [ ] How far back should the historical Gmail scan look (all-time vs a bounded number of years)?
- [ ] What authentication mechanism should gate access to the app? (flagged as an architecture decision for later)
- [ ] Data retention/consent policy specifics for Phase 3 health data (currently assumed: retained indefinitely,
      never shared with third parties, same auth as rest of app) — confirm once Phase 3 is scoped.
