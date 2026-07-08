# RationSmart PWA — Session Handoff

**Date written:** 2026-07-08
**Branch:** `testing`
**Live URL:** https://rationsmart-testing.vercel.app
**Backend:** `47.128.1.51:8000` (v1 API, JWT auth)

Read this file end-to-end at the start of a fresh session. It captures
what is currently working, what is still under investigation, and the
exact commands / paths to pick up.

---

## 1. Current state — one line each

| Area | Status |
|---|---|
| FEED 1 wipe on lang change + Continue | **FIXED** (commit `2cd8de4`, confirmed by user). |
| Feed Type UI | **RADIOS ONLY** — dropdown branch removed on 2026-07-08 per user ask. |
| Feed dropdown label (`display_name`) in Hindi | **WORKS** (`/v1/animal/feed-name` returns `display_name` translated). |
| Feed Type radio label in Hindi | **STILL ENGLISH** — pending backend confirmation on response shape when `?lang=hi` sent to `/v1/animal/unique-feed-type/{country_id}`. |
| Feed Category dropdown label in Hindi | **STILL ENGLISH** — same reason as Feed Type. |
| Diagnostic logging | `[store] setFeedSelections(...)` warning stays as safety net for future wipe regressions. |
| Auto-deploy | **OFF** — every change ships via manual `vercel deploy --prod --yes` + `vercel alias set`. |

---

## 2. The four preservation scenarios — behavior audit

The user has repeatedly restated these four rules. Each one is now
implemented; if any one regresses, this section is the reference.

### Scenario 1 — nav round-trip between Cattle Info ↔ Feed Selection
> "Enter data in Cattle Info → go to Feed Selection → enter data →
> navigate between screens. ALL data should be preserved as-is."

**How it works today:**
- Cattle Info's form is initialised from `cattleInfo` in the Zustand store
  (see `useState` initializer at `src/app/(main)/cattle-info/page.tsx:210`).
  `cattleInfo` is written on Continue and persisted via Zustand's persist
  middleware.
- Feed Selection's `items` state is initialised from `feedSelections` in
  the store, padded to 3 empty rows (`src/app/(main)/feed-selection/page.tsx:175`).
- Every `updateItem` / `addFeed` / `deleteItem` / `applySearchResult` on
  Feed Selection mirrors the local `items` back to the store via
  `setFeedSelections(updated)` — so the store is always in sync.
- A **fingerprint-based defensive resync** effect runs on mount +
  whenever `feedSelections` changes. It compares
  `feed_uuid|feed_type_name|category_name` per row; if store has real
  data that items doesn't reflect, items is pulled from store.
  See `src/app/(main)/feed-selection/page.tsx:190-219`.

### Scenario 2 — Generate Report → New Case
> "Enter data in both screens → generate a report → click New Case.
> All data should be preserved in all screens as-is."

**How it works today:**
- The **New Case** button on `/report` does one thing:
  `router.push("/cattle-info")` (see
  `src/app/(main)/report/page.tsx`, look for `handleNewCase`).
- No store wipes on New Case. Cattle Info + Feed Selection both remount
  and re-read the persisted data. Rows come back exactly as they were.
- **Do NOT re-add** a `setFeedSelections([])` / `setCattleInfo(null)`
  on New Case — that was the previous behavior and the user rejected it.

### Scenario 3 — Reset button on Cattle Info
> "When I click Reset in Cattle Info, BOTH screens' data should be cleared."

**How it works today** (`src/app/(main)/cattle-info/page.tsx:666-699`):
- `handleReset` fires:
  - `setForm(EMPTY_FORM)` — Cattle Info UI resets
  - `setFeedSelections([])` — clears store's rows (Feed Selection will
    render 3 fresh empty rows on next mount)
  - `setCattleInfo(null)` — critical, or nav-away-and-back would
    re-hydrate the form from stale cattleInfo
  - `setReportData(null as never)` — hides the forward-arrow toolbar icon
  - 500 ms pulse (spinner + "Form reset" snackbar) so the click is
    visually acknowledged (Nielsen 500 ms sweet spot)

### Scenario 4 — Simulation History restore
> "When I click History, all data including country & language should be
> set as it is in both screens."

**How it works today** (`loadSimulation` at
`src/app/(main)/cattle-info/page.tsx:278-425`):
- `getSimulationDetails(reportId, user.id)` returns cattle_info +
  feed_selection.
- Cattle Info form gets populated via `setForm(...)` including
  `simulation_language` (priority chain: backend value → prev form value
  → primary non-English country lang → null).
- **Also** calls `setCattleInfo(...)` with the restored payload so BOTH
  screens see the restore. Without this call, `langProvider` would
  read the previous simulation's language on nav to Feed Selection.
- `feed_selection[]` from backend is mapped to `FeedItem[]` shape and
  pushed via `setFeedSelections(restoredItems)`.
- `feedSelectionType` is set to `"evaluation"` if any row has
  `quantity_as_fed`, else `"recommendation"`.

### Synthetic dropdown option injection (all four scenarios)
When Cattle Info/Feed Selection remount and a stored feed_type /
category / feed_uuid is NOT in the fresh cascade response (translation
gap, country change, custom feed not yet in list), FeedRow injects a
synthetic entry so the CustomSelect can still render the stored label.
See:
- `src/components/FeedRow.tsx:418-424` (feed_type synthetic)
- `src/components/FeedRow.tsx:499-505` (category synthetic)
- `src/components/FeedRow.tsx:588-594` (sub-category synthetic)

Guards on cascade wipes (both categories and sub-cats): only clear
downstream when `!item.feed_uuid`. A row that has an actively-picked
feed_uuid keeps its stored labels even if the cascade response doesn't
echo them.

---

## 3. i18n V2 — where the dropdown-label translation currently stands

### What's in code today (post commit `308341e`, then radio-revert)
- `src/lib/api.ts:609-618` — `getFeedTypes` and `getFeedCategories` now
  send `?lang=<current>` via `langParam()`. The `lang` query param IS
  supported per swagger (`http://47.128.1.51:8000/openapi.json`).
- `src/components/FeedRow.tsx` — extractors were rewritten to return
  `{ name, display }` pairs from the response:
  - `name` = identity (English `type_name` / `category_name`), stored
    on the row + sent on downstream `/feed-name` call.
  - `display` = localized label (`display_type` / `display_category`
    or the `type_name` itself if backend just translates in-place).
- FEED 1 always shows RADIO buttons for Forage / Concentrate — the
  `feedTypes.length > 2` dropdown branch was removed.

### What is NOT yet confirmed
We have not verified the **actual response shape** the backend sends
when `?lang=hi` is set to `/v1/animal/unique-feed-type/{country_id}`
and `/v1/animal/unique-feed-category`. Two possible shapes:

1. **Bare-string array with translated identity**
   `["चारा", "अन्न"]`
   → both `name` and `display` end up Hindi → row stores Hindi as
     identity → downstream `/feed-name` call sends Hindi identity →
     backend must resolve translated identity for this to work.
   → The `hasForage` gate (`feed_type_name === "Forage"`) in
     `src/app/(main)/feed-selection/page.tsx:578` will **incorrectly
     block Generate** because the check hardcodes English.
2. **Object array with both fields**
   `[{"type_name":"Forage","display_type":"चारा"}, …]`
   → `name` = "Forage" (English identity), `display` = "चारा" (Hindi label).
   → Everything works end-to-end, including hasForage.

Yesterday's ("2026-07-07") flow reportedly showed translated labels
AND worked, so **either** the backend returned shape #2, **or** the
frontend was tolerating shape #1 by pure luck (no Forage gate hit).

### To confirm — inspect one live response

**Option A — read from the browser (no CLI needed)**
1. Open https://rationsmart-testing.vercel.app in Chrome.
2. Log in.
3. Open DevTools (Cmd+Opt+I) → Network tab.
4. Navigate Cattle Info (pick India + Hindi) → Continue → Feed Selection.
5. Find the request `unique-feed-type` (or `unique-feed-category`) in
   the Network list.
6. Click it → Response tab. Screenshot or copy that JSON.

**Option B — curl with a JWT (paste the token in a fresh session)**
1. DevTools → Application → Local Storage →
   `https://rationsmart-testing.vercel.app`.
2. Click the row `rationsmart-storage`.
3. Look at the pretty-printed value on the right. Find
   `state.user.token`. Copy that long string.
4. In terminal:
```bash
TOKEN="<paste>"
curl -s "http://47.128.1.51:8000/v1/animal/unique-feed-type/1?lang=hi" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
curl -s "http://47.128.1.51:8000/v1/animal/unique-feed-category?country_id=1&lang=hi" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
```
5. Paste those two JSONs at the top of the next session. From the shape
   we'll know exactly which extractor path is being taken.

### If backend returns shape #1 (translated identity, no display fields)
- Ask backend team to add `display_type` on unique-feed-type and
  `display_category` on unique-feed-category, mirroring the pattern
  already used by `/v1/animal/feed-name` (which returns `display_name`).
- Frontend already reads those fields — the moment backend ships them,
  labels translate automatically. No further FE work.

### If backend returns shape #2 (dual-field objects)
- The current code should already work. If labels are still English,
  double-check `state.user.preferred_language` in localStorage and
  `state.cattleInfo.simulation_language` — the langProvider reads
  simulation_language first, then preferred_language, then defaults to
  "en".

---

## 4. Files touched in this session (2026-07-08)

| File | Change |
|---|---|
| `src/lib/store.ts` | `setFeedSelections` now warns + logs stack when called with empty or all-blank rows. Diagnostic only. |
| `src/lib/api.ts` | `getFeedTypes` and `getFeedCategories` now spread `langParam()` (send `?lang=`). Comment block above the exports explains the rationale. |
| `src/components/FeedRow.tsx` | (a) `extractNames` → `extractOptions`, returns `{name, display}` pairs. (b) Feed Type always renders radios; dropdown branch removed. |
| `docs/SESSION_HANDOFF.md` | This file. |

Nothing else in the app was modified. If you see other diffs, they
predate this session.

---

## 5. Commands cheat sheet

```bash
# Deploy after any code change (auto-deploy is OFF, per user):
git push origin testing
vercel deploy --prod --yes
# grab the printed rationsmart-testing-<hash>-…vercel.app URL, then:
vercel alias set <that-url> rationsmart-testing.vercel.app

# Verify which build the alias currently serves:
vercel ls rationsmart-testing | head -5

# Backend swagger:
curl -s http://47.128.1.51:8000/openapi.json > /tmp/openapi.json
python3 -c "import json; d=json.load(open('/tmp/openapi.json')); \
  [print(p) for p in sorted(d['paths']) if 'feed' in p]"
```

---

## 6. What to do first in the next session

1. Get the two response bodies (see §3 Option A or B). Paste them.
2. Based on shape, either:
   - Confirm the current code works (shape #2) → close out i18n V2 labels.
   - Write a one-line ask to backend for `display_type` /
     `display_category` (shape #1) → keep frontend as-is.
3. Re-verify the four scenarios in §2 still all pass (no regression).
4. Re-run the language change reproduction from earlier — the console
   should stay quiet (no `[store] setFeedSelections(all-blank)` warning).

If the store's warning fires again unexpectedly, the stack in the
console output will name the culprit — patch that call site directly.
