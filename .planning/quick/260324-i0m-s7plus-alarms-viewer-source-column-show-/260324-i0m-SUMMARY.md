# Quick Task 260324-i0m: S7Plus Alarms Viewer Source Column — Connection Name

**Date:** 2026-03-24
**Status:** Complete

## What Was Done

Updated `S7PlusAlarmsViewerPage.vue` so the **Source** column displays the human-readable `protocolConnection` name (e.g. `PLC_MAIN`) instead of the raw `connectionId` number.

## Changes

**File:** `json-scada/src/AdminUI/src/components/S7PlusAlarmsViewerPage.vue`

- Added `connectionNameMap` ref to store `protocolConnectionNumber → name` mapping
- Added `fetchConnectionNames()` — calls `/Invoke/auth/listProtocolConnections` on mount and builds the map
- Added `connectionName(id)` helper — resolves an id to its name, falls back to the raw number string
- Updated `onMounted` to call `fetchAlarms()` and `fetchConnectionNames()` in parallel via `Promise.all`
- Added `<template #[`item.connectionId`]>` slot to display resolved name in the Source column

## Behavior

- Source column shows connection name (e.g. `PLC_MAIN`, `S7-1500_Line2`)
- Falls back to raw number string if no matching protocolConnection found
- Sorting on the Source column still works (header key remains `connectionId`)
- No regressions to filtering, ack, delete, or other column behavior
