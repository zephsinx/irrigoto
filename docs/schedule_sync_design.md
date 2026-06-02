# HA ↔ device schedule sync

How the weekly watering schedule stays consistent between Home Assistant and
an irrigoto device. This documents the system as it is built.

## Behavior

- The device stores its schedule in NVS and **runs it autonomously** — no
  network or HA required. Whatever schedule it holds keeps firing.
- You can edit the schedule on **either surface**: the device's built-in web
  UI (`/schedule`) or the HA dashboard's Compose form.
- Edits made in HA are held in HA and **pushed to the device when it next
  connects** (the devices deep-sleep, so this can be on the next wake).
- Edits made on the device **appear in HA after it connects** and HA polls it.
- Conflicts resolve **last-writer-wins** by per-entry `last_modified`; the
  **device wins** an exact-second tie. (Ties are effectively impossible in
  normal use; the rule exists only for determinism.)
- **Deleting** an entry is done on the device web UI. HA's sync is
  **upsert-only** — saving/editing HA slots never deletes device entries (see
  *Upsert-only* below).

## Entry schema

Each schedule entry carries:

| Field | Type | Notes |
| --- | --- | --- |
| `id` | uint32 | Stable across edits, assigned by the device at creation. `id=0` on the wire means "allocate one." |
| `last_modified` | uint32 | Unix epoch of the last edit. Drives LWW. |
| `source` | uint8 | Who last wrote it: `0` device/unknown, `2` HA. Diagnostic only. |
| `zone` | uint8 | **1-based** in the schedule (see *Zone encoding*). |
| `mode` | uint8 | 0 Pulse, 1 Gentle, 2 Smooth. |
| `depth` | uint8 | eighths of an inch (1 = 1/8″ … 8 = 1″); `0` = device default. |
| `hour`,`minute` | uint8 | local time of day. |
| `days_mask` | uint8 | bit per weekday, Sun=bit0 … Sat=bit6 (e.g. 42 = MWF). |
| `enabled` | uint8 | 0/1. |

A device-wide **`schedule_version`** (uint32, in NVS) increments on every commit
to the schedule, whatever the source. It is the cheap way both sides detect
"the other end changed something" without replaying events.

### Zone encoding (important)

The firmware uses **two zone conventions**:

- The immediate-water HTTP endpoint `/zone/water` takes a **0-based** zone id.
- **Schedule entries are 1-based**: the firmware waters `zone − 1`, and
  `sync_schedule` **rejects** any entry with `zone < 1`.

HA stores the **0-based device zone id** everywhere internally (it's what the
`"Name (#id)"` zone dropdown carries). So the HA↔device boundary converts:
**push adds 1, pull subtracts 1** (clamped ≥ 0). Get this wrong and either the
whole sync batch is rejected (`z=0`) or entries land on the wrong zone.

## Device (firmware) side

### Storage
NVS holds the entry blob (schema above), the `schedule_version` counter, and a
next-id counter so new entries get unique ids across reboots.

### HTTP API — `GET /api/schedule`
Returns JSON: `schedule_version`, `max_entries`, `last_status`, `ok`, the
`entries[]` (full schema), the device's `zones[]` as `{id, name}`, plus
`next_run` and `last_run`. This is HA's source of truth for the device's
current schedule (HA polls it; see *Pull* below).

`ok` is `true` when the last sync `last_status` indicates success
(`"ok…"` / `"sync ok…"`).

### ESPHome service — `sync_schedule(entries)`
Merges HA-proposed entries into the device schedule with LWW + device-wins-tie.
Payload is `;`-separated entries, each 11 comma fields:

```
id,last_modified,zone,mode,depth,hour,minute,days_mask,enabled,source,tombstone
```

- `id=0` → device allocates the next id (and reports it back via the next
  poll / event so HA can learn it).
- `tombstone=1` → delete that id if present, else no-op. Lets a delete be
  expressed explicitly so the device never deletes entries HA simply didn't
  mention.
- **Atomic & range-checked**: a non-tombstone entry with an out-of-range field
  (notably `zone < 1`) makes the device **reject the entire batch**.

### Event — `schedule_changed`
Fires on any schedule mutation and carries the new `schedule_version`. It does
**not** carry the entries (the full-entry payload was removed because it
overflowed the ESPHome API send buffer); HA fetches entries via the HTTP poll
above. `schedule_version` is also exposed to HA as an attribute on the device's
`schedule_full` sensor.

## Home Assistant side

### Per-device slot storage
Each device owns its own slots: `input_text.<dev>_sched_1 … _8`. A slot's value
is a flat CSV (empty = unused):

```
id,last_modified,zone,mode,depth,hour,minute,days_mask,enabled
```

`zone` here is the **0-based device id**; the device is implied by which
input_text holds the slot, so there is no device prefix. Each device's entries
are numbered 1..N within its own namespace — devices cannot collide, and slot
numbers are always sequential and present.

`input_text.<dev>_last_seen_schedule_version` holds HA's last-acknowledged
device version, for the inbound-pending indicator.

### Aggregator
`sensor.irrigoto_desired_schedule` walks every device's slots and exposes:
- `entries` — decoded, for the dashboard table (carries `slot`, `device`, etc.).
- `per_device_entries` — `{device: [raw entries]}`, the input to the push.

### Pull (device → HA): the mirror
`<dev>_mirror_device_schedule_to_slots` writes the device's entries into that
device's slots (positional, skipping a slot whose HA `last_modified` is newer —
a mid-edit guard). It triggers on the device's REST-sensor change, the
`schedule_changed` event, the **online edge**, and **HA start**.

On the event/online/start paths it forces a REST re-poll and **waits for the
poll to actually catch up to the device's `schedule_version`** before reading
entries — otherwise a stale/partial poll would mirror an incomplete list. While
a device is asleep the mirror does nothing (it never invents an empty schedule),
so cached slots persist; the schedule re-appears on the next wake.

### Push (HA → device): upsert-only
The reconciler builds a `sync_schedule` payload from `per_device_entries`
(converting zone +1) and calls `esphome.<dev>_sync_schedule`. The device's LWW
makes unchanged entries no-ops, so HA can re-push freely.

**Upsert-only by default**: the push emits **no tombstones**, so a sync can only
add/update device entries — never delete them. Deletes are done on the device
web UI, or deliberately by enabling the overlap-override boolean (which also
gates the existing "don't push an all-tombstone wipe against a non-empty
device" guard). This is intentional: HA's mirrored view can briefly be stale or
partial, and "absent from HA" must never silently delete live device entries.

### Compose form
Targets the **active device's** slots. The Slot selector (1–8) picks a slot for
that device; Save writes `<dev>_sched_<slot>` preserving the existing `id` (so
an edit updates the same device entry rather than creating a duplicate; an empty
slot saves `id=0` and the device allocates one). The Zone dropdown lists the
device's zones as `"Name (#id)"`; that list is cached in
`input_text.<dev>_zone_cache` so it survives device sleep and HA restarts, and
the form falls back to the cache when the device is asleep.

### Pending indicators
- `sensor.<dev>_schedule_pending_outbound` — HA has edits not yet on the device.
- `sensor.<dev>_schedule_pending_inbound` — device `schedule_version` is ahead
  of HA's `last_seen_schedule_version` (HA is pulling).
- `sensor.irrigoto_schedule_overlap_status` — cross-device time-overlap check.

## Design notes (why it's built this way)

- **Per-device slots, not one shared pool.** A single shared slot space with
  per-device positional mapping let two devices collide, leaving entries with no
  slot number. Per-device namespaces remove the collision entirely.
- **Upsert-only push.** Deriving deletes from "ids absent in HA's current view"
  wiped live entries whenever that view was stale or partial. HA no longer
  auto-deletes.
- **Pull over an HTTP poll, with a freshness wait.** Entries are read from the
  device's HTTP API (the event can't carry them). Because the devices deep-sleep
  and the poll lags, the mirror waits for the poll to reach the device's current
  `schedule_version` before committing — a fixed delay pulled stale/partial
  lists.
- **Zone ±1 at the boundary.** Schedule zones are 1-based on the device but
  0-based in HA; the conversion lives only at the push/pull boundary.
