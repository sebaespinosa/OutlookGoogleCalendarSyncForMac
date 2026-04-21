# OGCS-Mac — Implementation Plan

A cross-platform Python rewrite of [OutlookGoogleCalendarSync](https://github.com/phw198/OutlookGoogleCalendarSync) for macOS and Linux. The original Windows/.NET codebase is preserved under [old_repo/](old_repo/) for reference only; no .NET code, assets, or build system are reused.

---

## Top-level choices (fixed for all phases)

**Language**: Python 3.11+ (managed via [uv](https://github.com/astral-sh/uv) or pyenv).
Rationale: mature Google Calendar + Microsoft Graph SDKs, cross-platform, easy to run as a launchd/systemd daemon, OS keychain access via `keyring`, simple distribution through `pipx` or Homebrew.

**Architecture**: single Python package `ogcs_sync` exposing:
- a CLI (`ogcs`) built on Typer,
- a core library (calendar adapters, sync engine, config),
- a scheduler (APScheduler) for the daemon mode.

**Outlook target**: **Microsoft Graph API only** — Microsoft 365 / Outlook.com / Exchange Online. No desktop-Outlook integration. No AppleScript.

**Base folder**: the root of this repo. New Python code lives at the top level; the old .NET source stays untouched in [old_repo/](old_repo/).

**What we reuse from the C# codebase** (as reference, not code):
- [old_repo/src/OutlookGoogleCalendarSync/SettingsStore/Calendar.cs](old_repo/src/OutlookGoogleCalendarSync/SettingsStore/Calendar.cs): the profile schema (direction, date window, filters, color mapping, reminder DND, obfuscation rules).
- [old_repo/src/OutlookGoogleCalendarSync/Sync/Calendar.cs](old_repo/src/OutlookGoogleCalendarSync/Sync/Calendar.cs) & [old_repo/src/OutlookGoogleCalendarSync/Sync/Engine.cs](old_repo/src/OutlookGoogleCalendarSync/Sync/Engine.cs): the sync lifecycle (fetch → match → diff → apply → record).
- [old_repo/src/OutlookGoogleCalendarSync/Google/CustomProperty.cs](old_repo/src/OutlookGoogleCalendarSync/Google/CustomProperty.cs) + [old_repo/src/OutlookGoogleCalendarSync/Outlook.Graph/O365CustomProperty.cs](old_repo/src/OutlookGoogleCalendarSync/Outlook.Graph/O365CustomProperty.cs): how to stamp cross-system IDs on events so they can be re-matched across runs. This is the critical trick.
- [old_repo/src/OutlookGoogleCalendarSync/Google/GoogleRecurrence.cs](old_repo/src/OutlookGoogleCalendarSync/Google/GoogleRecurrence.cs) + [old_repo/src/OutlookGoogleCalendarSync/Outlook.Graph/O365Recurrence.cs](old_repo/src/OutlookGoogleCalendarSync/Outlook.Graph/O365Recurrence.cs): recurrence mapping logic between RRULE (Google) and MS Graph's `PatternedRecurrence`.
- [old_repo/src/OutlookGoogleCalendarSync/Google/EventColour.cs](old_repo/src/OutlookGoogleCalendarSync/Google/EventColour.cs): color mapping approach.
- [old_repo/src/OutlookGoogleCalendarSync/Obfuscate.cs](old_repo/src/OutlookGoogleCalendarSync/Obfuscate.cs): regex obfuscation rules.

**What we drop entirely**: all WinForms, Squirrel, COM interop, Outlook classic client, Windows Registry, WMI, System.Management, Microsoft.Office.Interop.*, app.manifest, Program.cs bootstrapping, Telemetry/ErrorReporting to the original GCP endpoint.

---

## Phase 0 — Project scaffolding (0.5–1 day)

**Goal**: empty but installable package with test/lint/CI in place.

**Deliverables**:
- Project laid out at the repo root (alongside [old_repo/](old_repo/)). Suggested package name: `ogcs_sync`; console script name: `ogcs`.
- `pyproject.toml` with `build-system = hatchling` (or `uv`), console script `ogcs = ogcs_sync.cli:app`.
- Package skeleton:
  ```
  ogcs_sync/
    __init__.py
    cli.py              # Typer app, subcommand stubs
    config/             # config loading/validation
    adapters/           # google.py, microsoft.py, base.py
    sync/               # engine.py, matcher.py, differ.py, planner.py
    model.py            # Event dataclass, Calendar dataclass
    storage/            # state.py (SQLite), credentials.py (keyring)
    scheduler/          # daemon.py
    logging_setup.py
    paths.py            # XDG / macOS paths
  tests/
  old_repo/             # untouched reference code
  PLAN.md               # this file
  ```
- Tooling: `ruff` (lint+format), `mypy --strict` on `ogcs_sync/`, `pytest` + `pytest-cov`.
- CI: GitHub Actions matrix on macos-latest + ubuntu-latest, Python 3.11/3.12/3.13.

**Spec — paths module** (important, used by every later phase):
- Config dir: `$XDG_CONFIG_HOME/ogcs` (default `~/.config/ogcs`) on Linux; `~/Library/Application Support/ogcs` on macOS.
- Data dir (state DB, logs): `$XDG_DATA_HOME/ogcs` or `~/Library/Application Support/ogcs/data`.
- Cache dir: `$XDG_CACHE_HOME/ogcs` or `~/Library/Caches/ogcs`.

**Acceptance**: `pipx install -e .` works on both macOS and Linux; `ogcs --help` prints; `pytest` green with one placeholder test.

---

## Phase 1 — Authentication (2–3 days)

**Goal**: user can authenticate against Google and Microsoft from the CLI; tokens persist in the OS keychain.

**Deliverables**:
- `ogcs auth google` — runs Google OAuth 2.0 installed-app flow (loopback redirect on `http://localhost:<random>`). Uses `google-auth-oauthlib`. Scopes: `https://www.googleapis.com/auth/calendar`.
- `ogcs auth microsoft` — runs MSAL public-client flow. Two options:
  - Preferred: interactive browser + loopback redirect (`PublicClientApplication.acquire_token_interactive`).
  - Fallback: device-code flow for headless Linux boxes.
  - Scopes: `Calendars.ReadWrite`, `offline_access`, `User.Read`.
- `ogcs auth status` — prints authenticated identity for each provider (`userinfo` / `/me`).
- `ogcs auth revoke <provider>` — clears tokens locally.

**Spec — credential storage**:
- Use `keyring` library. Service names: `ogcs.google`, `ogcs.microsoft`. Username: the account email.
- Store the full token JSON (access + refresh + expiry) as a single secret.
- On Linux without a D-Bus session, fall back to `keyrings.alt` filesystem backend with a strong warning.
- Never write tokens to the config file.

**Spec — OAuth client credentials**:
- The original project bundles its own Google client ID/secret (see [old_repo/src/OutlookGoogleCalendarSync/Google/ApiKeyring.cs](old_repo/src/OutlookGoogleCalendarSync/Google/ApiKeyring.cs)) which is obfuscated but public. For a personal tool, the cleanest path is: **each user registers their own Google Cloud project + Azure app registration** and pastes the client ID/secret into a config file. Provide a `docs/SETUP.md` with click-by-click instructions.
- Don't check in any client secrets.

**Acceptance**: both `ogcs auth google` and `ogcs auth microsoft` complete end-to-end on a fresh Mac; restarting the terminal and running `ogcs auth status` still shows authenticated users; tokens survive a reboot.

---

## Phase 2 — Calendar adapters (read-only) (3–4 days)

**Goal**: internal `Event` model + adapters that can list calendars and fetch events in a date range from both providers. No writes yet.

**Deliverables**:
- `model.py`:
  ```python
  @dataclass
  class Event:
      source: Literal["google", "microsoft"]
      source_id: str
      icaluid: str | None
      summary: str
      description: str | None
      location: str | None
      start: datetime           # timezone-aware
      end: datetime
      all_day: bool
      timezone: str             # IANA tz
      attendees: list[Attendee]
      reminders: list[Reminder]
      availability: Literal["free","busy","tentative","oof"]
      privacy: Literal["default","public","private","confidential"]
      color: str | None         # provider-native color id
      recurrence: Recurrence | None
      is_recurring_instance: bool
      recurring_master_id: str | None
      custom_properties: dict[str, str]   # round-trips through provider
      etag: str | None
      last_modified: datetime
  ```
- `adapters/base.py`: `CalendarAdapter` ABC with `list_calendars()`, `list_events(calendar_id, start, end)`, `get_event(calendar_id, event_id)`.
- `adapters/google.py`: uses `google-api-python-client`. Extended props live under `event.extendedProperties.private`.
- `adapters/microsoft.py`: uses `msgraph-sdk` (or direct httpx for lighter deps). Extended props live under `singleValueExtendedProperties` with a namespaced GUID.
- CLI: `ogcs calendars list --provider google|microsoft`, `ogcs events list --provider ... --calendar <id> --from 2026-04-01 --to 2026-05-01 [--json]`.

**Spec — custom properties**:
- Pick a namespace string: `com.sebaespinosa.ogcs`.
- Always write these on copied events:
  - `ogcs:source_system` = "google" or "microsoft"
  - `ogcs:source_id` = the original provider's event ID
  - `ogcs:source_etag` = etag/lastmod at copy time
  - `ogcs:profile` = profile name
- On the next sync these are the primary matching key.

**Spec — pagination**:
- Always paginate fully. Google: `pageToken` + `maxResults=2500`. Graph: `$top=500` + `@odata.nextLink`.

**Acceptance**: `ogcs events list --provider google --calendar primary --from … --to …` returns a JSON dump of normalized events; same for microsoft. Unit tests cover: all-day, timezone conversion, recurring master, single instance override, events with attendees.

---

## Phase 3 — Configuration & profiles (1 day)

**Goal**: user-editable config defining one or more sync profiles.

**Deliverables**:
- `config/schema.py` using **pydantic v2**.
- File: `$CONFIG_DIR/config.toml`.
- CLI: `ogcs config init` (interactive wizard), `ogcs config show`, `ogcs config validate`, `ogcs config edit` (opens `$EDITOR`).

**Spec — config schema** (informed by [old_repo/src/OutlookGoogleCalendarSync/SettingsStore/Calendar.cs](old_repo/src/OutlookGoogleCalendarSync/SettingsStore/Calendar.cs)):

```toml
[general]
default_profile = "personal"

[oauth]
google_client_id_env = "OGCS_GOOGLE_CLIENT_ID"
google_client_secret_env = "OGCS_GOOGLE_CLIENT_SECRET"
microsoft_client_id = "…"

[[profile]]
name = "personal"
direction = "google_to_microsoft"   # or microsoft_to_google, bidirectional
source_google_calendar = "primary"
source_microsoft_calendar = "AQMk…"
days_past = 1
days_future = 60
simple_match = false                 # if true, match only on signature
disable_delete = true
confirm_delete = false               # CLI-only: only meaningful in interactive runs
merge_items = true                   # don't overwrite pre-existing dest events on first sync

  [profile.attributes]
  sync_description = true
  sync_location = true
  sync_reminders = false
  sync_attendees = false
  max_attendees = 200
  sync_colors = false

  [profile.filters]
  exclude_free = false
  exclude_tentative = false
  exclude_private = false
  exclude_all_day = false
  exclude_subject_regex = ""
  exclude_google_color_ids = []
  exclude_microsoft_categories = []

  [profile.overrides]
  force_private = false
  force_availability = "busy"      # or unset

  [profile.reminder_dnd]
  enabled = false
  start = "22:00"
  end = "06:00"

  [[profile.color_map]]
  google = "5"
  microsoft = "preset1"

  [[profile.obfuscation]]
  pattern = '\b(Acme Corp|Internal)\b'
  replacement = "REDACTED"
  applies_to = "summary"   # or "description", "both"
  direction = "to_google"  # which direction the rule applies on

[schedule]
enabled = true
interval = "15m"
catch_up_on_start = true
```

**Acceptance**: `ogcs config init` walks through calendar selection (hits both providers), writes valid config, `ogcs config validate` is green, schema round-trips via pydantic without data loss.

---

## Phase 4 — Matching, diffing, and plan generation (3–5 days)

**Goal**: given a profile, produce a declarative sync plan (`create[]`, `update[]`, `delete[]`) without touching either calendar.

**Deliverables**:
- `sync/matcher.py`:
  - Primary match: `dest.custom["ogcs:source_id"] == source.source_id` where both belong to the same profile.
  - Fallback signature match (for first run or events where the custom property was wiped): hash of `(normalized_summary, start_utc, end_utc)` — this is what the C# code calls "simple match" (see `SimpleMatch` in [old_repo/src/OutlookGoogleCalendarSync/SettingsStore/Calendar.cs](old_repo/src/OutlookGoogleCalendarSync/SettingsStore/Calendar.cs)).
  - Returns three buckets: `matched_pairs`, `orphans_source` (to create), `orphans_dest` (candidates for delete).
- `sync/differ.py`: given a matched pair, returns a list of `FieldChange(field, old, new)` respecting which attributes the profile syncs.
- `sync/planner.py`: composes a `SyncPlan` dataclass:
  ```python
  SyncPlan(profile, create=[Event…], update=[(dest_event, patch_dict)…], delete=[Event…], skipped=[…], warnings=[…])
  ```
- `sync/filters.py`: applies the profile's exclusion rules and obfuscation to a source event *before* it enters matching/planning.
- CLI: `ogcs sync plan --profile personal [--json]` — prints a human-readable summary and an optional machine-readable plan.

**Spec — recurrence handling**:
- Only copy the *master* event + its exceptions. Don't treat expanded instances as individual events.
- Map Google RRULE strings → MS Graph's `PatternedRecurrence`. Port the logic from [old_repo/src/OutlookGoogleCalendarSync/Google/GoogleRecurrence.cs](old_repo/src/OutlookGoogleCalendarSync/Google/GoogleRecurrence.cs) and [old_repo/src/OutlookGoogleCalendarSync/Outlook.Graph/O365Recurrence.cs](old_repo/src/OutlookGoogleCalendarSync/Outlook.Graph/O365Recurrence.cs). This is the single gnarliest piece of logic in the project — budget time for it.
- Use the `icalendar` library for RRULE parsing/generation.

**Spec — timezone discipline**:
- Always normalize to aware `datetime` in UTC for comparisons.
- Preserve the original IANA timezone string when writing, because the two services handle DST differently for recurrences.

**Acceptance**: given a hand-crafted pair of source/dest event lists, `plan --json` output matches fixtures for: first-run (all creates), steady-state (no-op), an edit on source, a deletion on source, an attendee-only change (should be no-op when `sync_attendees=false`), a recurring series with one modified instance.

---

## Phase 5 — Plan execution (one-way) (2–4 days)

**Goal**: execute a plan in `google_to_microsoft` or `microsoft_to_google` direction.

**Deliverables**:
- `sync/executor.py` with `apply_plan(plan, dry_run=False)`.
- Write paths in both adapters: `create_event`, `update_event(id, patch)`, `delete_event(id)`.
- Retry with exponential backoff (`tenacity`) on 429 / 5xx from either API.
- Rate limiter (e.g. 5 req/sec Graph, 10 req/sec Google — conservative) via `aiolimiter` or a simple token bucket.
- After each successful create, write the cross-system custom properties on the new event.
- CLI: `ogcs sync run --profile personal [--dry-run] [--yes]`.
  - Without `--yes`, deletes require interactive confirmation when `confirm_delete=true`.

**Spec — idempotency & safety**:
- Lockfile at `$STATE_DIR/<profile>.lock` using `filelock`. Reject concurrent runs on the same profile.
- Before deleting any event, verify it still has the expected `ogcs:source_system` / `ogcs:profile` custom properties. This prevents deleting user-authored events that happen to match by signature.
- On the first run for a new profile, bias toward creates + merges; require `--allow-delete` to actually delete anything.
- Track sync run in SQLite (`$DATA_DIR/state.db`): profile, started_at, finished_at, created/updated/deleted counts, errors.

**Acceptance**: a plan with 20 creates / 5 updates / 2 deletes applies cleanly; rerun of `sync run` produces an empty plan; `--dry-run` does not mutate either calendar; a forced 429 causes a retry then success.

---

## Phase 6 — Two-way sync (2–3 days)

**Goal**: `direction = "bidirectional"` works correctly and doesn't loop.

**Deliverables**:
- Extend planner to produce two paired plans (A→B and B→A) from a single matching pass.
- Conflict resolution: configurable `bidirectional_tie_breaker = "last_modified_wins" | "prefer_google" | "prefer_microsoft"`.
- Deletion detection: requires remembering what was synced last time. Store a minimal `seen_events` table in SQLite keyed by `(profile, source_system, source_id, last_etag)`.
  - If an event was seen last run but is absent from both sides now → nothing.
  - If absent from one side only → propagate delete to the other (subject to `disable_delete` / `confirm_delete`).
- Guard against echo loops: an event created by us on the other side carries our `ogcs:source_id` custom property, so on the next run it matches instead of being treated as a new orphan.

**Acceptance**: end-to-end test with a clean pair of calendars: create on Google → appears in MS; edit on MS → propagates to Google; delete on Google → propagates to MS; the very next run is a no-op. Crucially: editing on *both* sides between runs resolves by configured tie-breaker with no duplicates.

---

## Phase 7 — Background daemon & scheduling (1–2 days)

**Goal**: the user can start a long-running process that syncs every N minutes, survives reboot.

**Deliverables**:
- `scheduler/daemon.py` using **APScheduler**'s `BlockingScheduler`. Cron-like: `interval` from config, plus optional `catch_up_on_start`.
- CLI: `ogcs run --daemon` (foreground, logs to stdout — for use inside launchd/systemd); `ogcs run --once` (single sync of all enabled profiles then exit).
- Signal handling: SIGTERM → finish current sync, flush state, exit. SIGHUP → reload config.
- Per-profile jitter to avoid API bursts when multiple profiles share an interval.
- `ogcs service install` — generates and installs a service file:
  - macOS: `~/Library/LaunchAgents/com.sebaespinosa.ogcs.plist` with `RunAtLoad=true`, `KeepAlive.SuccessfulExit=false`, `StandardOutPath` to our log file. Loads via `launchctl load`.
  - Linux: `~/.config/systemd/user/ogcs.service` with `Restart=on-failure`, activated via `systemctl --user enable --now ogcs`.
- `ogcs service uninstall`, `ogcs service status`, `ogcs service logs`.

**Spec — logging**:
- `structlog` with JSON output when daemonized, human-readable when interactive (detect via `sys.stdout.isatty()`).
- Rotate at `$DATA_DIR/logs/ogcs.log` via `logging.handlers.TimedRotatingFileHandler` (7-day retention).

**Spec — health & notifications**:
- Write `$DATA_DIR/last_sync.json` after each run so external tools can check freshness.
- OS notification on sync *failure* (not success) via `pync` on macOS, `notify2`/`dbus` on Linux. Behind a `notifications_on_error = true` config flag.

**Acceptance**: `ogcs service install` on macOS — syncs continue after logout/login and after reboot. Same for systemd-user on a Linux laptop. `launchctl unload` / `systemctl --user stop` cleanly terminates mid-sync without corrupting state.

---

## Phase 8 — Observability, history, recovery (1–2 days)

**Goal**: easy to answer "did my last sync work?" and "what did it do?".

**Deliverables**:
- CLI: `ogcs status` — last sync time per profile, next scheduled run, credential validity, recent error count.
- CLI: `ogcs history [--profile x] [--limit 20]` — reads from SQLite, prints a table.
- CLI: `ogcs logs [--follow]` — tails the daemon log file.
- `ogcs doctor` — runs a series of checks (auth tokens valid? calendar IDs still exist? config schema valid? clock sync reasonable? keychain accessible? service installed and loaded?).

**Acceptance**: each command prints useful output on both a happy state and a broken state (expired Google token, revoked Graph consent, deleted source calendar).

---

## Phase 9 — Packaging & distribution (1 day)

**Goal**: a non-developer can install it cleanly.

**Deliverables**:
- Publish to PyPI (or a private index): `pip install ogcs-sync`.
- `pipx` is the primary recommended install path.
- Homebrew tap (optional): `brew install sebaespinosa/tap/ogcs` that wraps pipx.
- `docs/INSTALL.md`, `docs/SETUP.md` (the OAuth-app registration walkthrough), `docs/CONFIG.md`.
- Makefile / `uv` scripts for release cut (`uv build && uv publish`).

**Acceptance**: clean macOS VM + `pipx install ogcs-sync`, then `ogcs auth google && ogcs auth microsoft && ogcs config init && ogcs run --once` succeeds.

---

## Phase 10 — Optional polish (as desired)

Candidates, in no particular order:
- **TUI** via Textual for an interactive dashboard of profiles/last runs.
- **Tauri or PyQt GUI** if a graphical UI is wanted later — but CLI + daemon is already the primary value.
- **Microsoft Graph webhooks** for push-style near-real-time sync (requires a publicly reachable HTTPS endpoint — probably out of scope for a local tool).
- **Apple Calendar (EventKit) adapter** via `pyobjc` — a Mac-only bonus that would let people sync Google ↔ Apple Calendar too. Non-trivial but well-scoped once the adapter interface exists.

---

## Suggested execution order and milestones

| Milestone | Covers | Outcome |
|---|---|---|
| **M1 — hello world** | Phase 0 | Installable CLI, CI green |
| **M2 — auth works** | Phase 1 | Tokens in keychain |
| **M3 — I can read** | Phases 2, 3 | Listing + normalized events, config in place |
| **M4 — first sync** | Phases 4, 5 | One-way `google_to_microsoft` end-to-end |
| **M5 — real usage** | Phase 6 | Bidirectional, running by hand daily |
| **M6 — set and forget** | Phases 7, 8 | Daemonized with launchd, observable |
| **M7 — shippable** | Phase 9 | Others can install it |

Rough total: **2–3 weeks of focused work** for M1–M6; M7 another few days.

---

## Risks / things that will be harder than they look

1. **Recurrence mapping**. Budget extra time for Phase 4. The reference C# recurrence code is ~1,100 lines for a reason.
2. **Microsoft Graph extended properties** are fiddly — they use GUID-namespaced property sets and require correct `$expand` clauses to read back. Test this end-to-end early.
3. **First-run merge** on existing calendars. Without careful signature matching and a conservative default, the first sync can duplicate every event. Phase 5's `merge_items + disable_delete` defaults are the safety net.
4. **Token refresh under launchd**. macOS Keychain access from background processes has historically been finicky. Test refresh flows while the daemon is actually running under launchd, not just from the shell.
