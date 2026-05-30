# Snooze + Text-Click-Toggle — Design

Date: 2026-05-30
Target file: `src/todo-card.js` (+ `translations/{de,en,no}.json`)

## Summary

Two enhancements to the custom Home Assistant todo card:

1. **Snooze items** (Gmail-style): defer an item to a future moment, hidden from the list until the time arrives.
2. **Text-click toggle**: clicking the task text completes/uncompletes the item (current behavior expands it). Expand is moved to a chevron icon.

## Architecture

Single-file Lit element (`src/todo-card.js`, ~1441 lines). All changes are in-file:

- Task-row layout refactor
- New snooze popup render fragment
- Filter state + filter-dropdown extension
- New helpers: `_isSnoozed`, `_computeSnoozePresets`, `_handleSnooze`, `_handleOpenSnoozeMenu`
- Translation keys added to all three locale files

No backend changes. Snooze uses HA `todo.update_item` service with `due_datetime`.

## Feature 2 — Text-Click Toggle

### Current behavior
- `task-item` row: `@click=_toggleExpand`
- `checkbox`: `@click=_handleStatusUpdate` (stopPropagation)
- Expand area: subtasks + Edit button

### New behavior
- Row click handler removed
- Checkbox + task-text together: `@click=_handleStatusUpdate`
- New chevron icon (right side): `@click=_toggleExpand`
  - `mdi:chevron-down` collapsed / `mdi:chevron-up` expanded
- Layout: `[☐] [title + due badge] [🕐 snooze] [⌄ chevron]`
- Applies to both regular `task-item` and `shopping-item` variant
- Completed items: text click un-completes (symmetric)

## Feature 1 — Snooze

### Snooze trigger icon

- `mdi:clock-outline` in every active task row, right of text, left of chevron
- Becomes `mdi:clock` when item is snoozed (future `due_datetime`)
- `@click` stopPropagation; opens popup via `this._snoozeMenuTaskId = task.uid`
- Hidden on completed items

### Snooze popup

Rendered below the task row when `_snoozeMenuTaskId === task.uid`:

```
Zurückstellen bis…
Morgen              So., 07:00
Nächste Woche       Mo., 07:00
Nächstes Wochenende Sa., 07:00
─────────────────
📅 Datum und Zeit auswählen
```

### Preset computation (`_computeSnoozePresets()`)

Returns 3 presets relative to "now":

- **Tomorrow**: today + 1 day at 07:00 local
- **Next week**: next Monday at 07:00 local (if today is Monday → +7 days)
- **Next weekend**: next Saturday at 07:00 local (if today is Saturday → +7 days)

Labels show localized short weekday + time using `Intl.DateTimeFormat` with the user's `hass.locale.language`.

### Custom picker

- "Datum und Zeit auswählen" reveals inline date + time inputs (same style as add/edit area)
- Confirm button calls `_handleSnooze(task, datetime)`
- Cancel closes the picker

### `_handleSnooze(task, isoDatetime)`

```js
await this._hass.callService("todo", "update_item",
  { item: task.uid, due_datetime: isoDatetime },
  { entity_id: this._config.entity });
this._snoozeMenuTaskId = null;
this.fetchTodoItems();
```

### `_isSnoozed(task)`

Returns true iff:
- `task.due` contains `T` (i.e. is a datetime, not a date-only value), AND
- `new Date(task.due) > new Date()`, AND
- `task.status !== 'completed'`

### Filter logic

- Filter state extends: `{ active, overdue, completed, snoozed }`, `snoozed` default `false`
- Snoozed items are excluded from active and overdue categories (snoozed is its own bucket)
- Filter dropdown gains a "Snoozed" toggle with count
- Header counter line gains snoozed count when > 0
- Sorting unchanged

### Translations (new keys)

- `snoozeUntil` — "Zurückstellen bis…" / "Snooze until…" / "Utsett til…"
- `snoozeTomorrow` — "Morgen" / "Tomorrow" / "I morgen"
- `snoozeNextWeek` — "Nächste Woche" / "Next week" / "Neste uke"
- `snoozeNextWeekend` — "Nächstes Wochenende" / "Next weekend" / "Neste helg"
- `snoozePickDateTime` — "Datum und Zeit auswählen" / "Pick date and time" / "Velg dato og tid"
- `snoozed` — "Zurückgestellt" / "Snoozed" / "Utsatt"

## Edge Cases

- **Manually set to done while snoozed**: item leaves snoozed bucket automatically (filter checks status)
- **Snooze time reached**: item becomes overdue at next fetch (existing polling refresh handles this; no dedicated timer)
- **User sets date-only `due_date` manually**: not treated as snoozed (date-only items stay visible)
- **Snooze popup open while another task is tapped**: previously-open popup closes (`_snoozeMenuTaskId` is single-valued)
- **Custom picker datetime in the past**: still sent to HA; will appear as overdue immediately (intentional, mirrors normal due-datetime behavior)
- **Shopping-item variant**: gets the same chevron/text-toggle/snooze treatment

## Out of Scope

- No per-card configurable presets (chose Gmail-style fixed 3)
- No swipe gestures
- No backend storage of snooze metadata; relies on `due_datetime` semantics
- No notifications when snooze time elapses (HA-side concern)

## Files Touched

- `src/todo-card.js` — all logic + render changes
- `translations/de.json`, `translations/en.json`, `translations/no.json` — 6 new keys each
