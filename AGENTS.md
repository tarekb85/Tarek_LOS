# AGENTS.md — Tarek Life OS

Read this first in any chat that touches Life OS. It records *how the system is meant to work* so nobody has to re-explain it.

**Source-of-truth order (when things disagree):**
1. Live database schema + data — always check `information_schema` before assuming a column/table exists
2. This file
3. Dashboard code (`index.html`) — what is actually implemented
4. Chat history — context only

Never put tokens, passwords, backups or document numbers in this file or the repo.

---

## 1. The Calendar-vs-Reminders rule  (most important convention)

> **If Tarek says "add it to the calendar", it goes on the calendar ONLY. Do not also create a Reminders row.**

| Goes on the **Google Calendar** (never in Reminders) | Goes in **Reminders** (dashboard) |
|---|---|
| Appointments with an exact date/time (car inspection, doctor, etc.) | Things with **no calendar entry** that Tarek wants visible until actioned |
| Long-term dated items (e.g. bike maintenance in 6 months) | e.g. "clean the water filter and use it again" |

Why: the 6am daily digest reads the calendar ~7 days ahead, so a calendar item is already on the radar. Reminders is the "sitting there until I do it" list.

**Long-lead calendar events get explicit pings.** Default calendar pings are only 1 day / 1 hour / 30 min, which is too late for a months-away item. For all-day events use `overrideReminders` (minutes before 00:00 of the day): **9540** (= 9am, one week before) and **900** (= 9am, day before). Timed appointments: 60 and 1440 min is fine.

When moving something to the calendar: create the event, verify it, then **delete** the Reminders row (don't leave both).

Examples: a car inspection with an exact date and time → calendar only. Bike maintenance (all-day, months ahead) → calendar only.

---

## 2. Which table holds what

Always tell Tarek which table you picked so he can correct you. Don't ask "should we make a new table?" — default to the generic tables below.

| Table | Use for | Notes |
|---|---|---|
| `recurring_items` (+ `recurring_log`) | Cadence-based care: pets, plants, other | `category` pet/plant/other; `cadence_months` NOT NULL; next_due computed from last_done |
| `reminders` | Personal tasks with **no calendar entry** (health, family, work, birthdays, school, travel, finance) | `status` pending/done; optional `cadence_months`; `priority`; `sensitive_document_id` links doc-expiry reminders |
| `notes` | Freeform notes. `category`: `place`, `hazard`, `storage_location`, `general`, or new | **`status` must be `active`/`resolved`** (not "pending" — DB constraint rejects it). Place notes set `place_id` |
| `places` | Named places (shops, areas, trips…) | Referenced by `notes.place_id`, `shopping_items.place_id` |
| `shopping_items` | Things to buy (`category` groceries / general), optional `place_id` | `status` pending/done. Deliberately **not** reminders and not in the digest — it's a list Tarek (or Claude on request) checks when going to a shop |
| `items` | Physical possessions | `attributes` jsonb for category-specific fields; `status` active/needs_repair/retired |
| `debts` | Money in both directions. `direction` = `owed_to_me` (default) or `i_owe`; `status` owed/settled (= outstanding/closed, for both directions) | Dashboard "Money" section shows the two groups separately with their own totals. Always set `direction` when inserting |
| `sensitive_documents` | Passport/ID/insurance **metadata only** (the files themselves are stored elsewhere, never here) | `status` active/expired/replaced; expiry may spawn a linked reminder |
| `medications` | Medication list | |
| `personal_profile` | Key/value personal facts | |
| `daily_digest` | Snapshot written by the 6am digest (headline, verdict, calendar jsonb) | Don't hand-edit |
| `project_context` | Single row (id=1): short mirror of the conventions so any chat with the database connector can read it | Keep in sync with the private master; update after any structural change |
| `spain_trip_list` | Legacy trip list | Superseded by notes/shopping_items; leave unless asked |

Rule of thumb: one table per purpose; only add a table when real arithmetic/date logic or CHECK-constrained enums need real columns.

---

## 3. Dashboard (`index.html`)

- Static single-page app talking to the database directly from the browser. Login required; every table is locked by row-level security to the owner's account. Public sign-ups are disabled.
- Sections (all collapsible, state remembered; collapse all / expand all): Today, Pets, Plants, Other, Places, Reminders, Shopping, Debts. Hero overview at top (Overdue / Today / Next 7d / Owed tiles + jump chips).
- The dashboard **does not read the calendar or `daily_digest`**. It only shows database tables — which is exactly why calendar items must not be duplicated into Reminders.
- Place notes and shopping items are written from the Places section; reminders from the Reminders section.
- Hosting, access and push details are kept privately, not in this repo. Claude pushes dashboard changes itself (Claude GitHub App); never hand Tarek manual upload steps. Never commit tokens, secrets or backups.

---

## 4. Daily digest

A 6am scheduled task ("Life OS Morning Digest") writes `daily_digest` and a styled artifact. It always reads then updates the existing artifact in place, never publishes a new one. Window ≈ 7 days ahead; includes the "family" calendar. Identifiers for the task and artifact are kept privately.

---

## 5. Working rules

- Read live state before writing; never work from cached copies of table contents.
- DDL via `apply_migration`; single statements via `execute_sql`. Verify every change by querying afterwards — a success response is not proof.
- New tables need **both** RLS (policy scoped to the owner's `auth.uid()`) **and** explicit GRANTs/REVOKEs (default ACL auto-grants anon/authenticated).
- Check CHECK constraints before inserting a new status/category value.
- Dashboard edits: edit → syntax-check the JS → push → verify the commit. Test with a real browser when behaviour matters.
- Never write tokens or secrets into docs, repo or database tables.
- Supabase free tier auto-pauses after ~7 days without writes; the weekly review/digest keeps it awake.
- Ask before: schema changes, destructive deletes, anything touching `sensitive_documents`.

---

## 6. Changelog of conventions
- 2026-11-06: Dashboard "Money" section now splits owed-to-me vs I-owe using `debts.direction` (column added 30 Oct; dashboard previously ignored it).
- 2026-11-06: Public copy of this file trimmed to conventions only; infrastructure details moved to the private master.
- 2026-10-16: Calendar-vs-Reminders rule + long-lead ping practice recorded.
- 2026-10 : Collapsible sections + hero overview; shopping list section; notes status fix (`active`/`resolved`).
- 2026-09 : Schema genericised (household_tools→items, household_notes+place_notes→notes).
