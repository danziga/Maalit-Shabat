# מעלית שבת – נתניה

A single self-contained `index.html` (no server, no build, no external libraries)
that computes, on-demand for **any** Hebrew year, the Shabbat/Yom-Tov elevator
schedule for Netanya: candle-lighting / havdala / arvit times, and the windows
when the "Shabbat elevator" runs automatically.

Live site: https://danziga.github.io/Maalit-Shabat/ — GitHub Pages serves
`index.html` from `main` directly (no Actions workflow). A push to `main` is the
deploy; the live page may need a hard refresh (Ctrl+F5) to bypass cache.

## Layout

- `index.html` — everything: `<style>`, `<body>` skeleton, one `<script>`.
  Sections inside the script are marked with `/* ===== ... ===== */` banners.
- `README.md` — full method write-up (Hebrew). Keep it in sync with code changes.
- `זמני מעלית שבת.xlsx` — the community's original תשפ"ו schedule; source of the
  `TEMPLATES` library and the candle-lighting offsets. Not used at runtime.

## Computation chain (per Hebrew year)

`buildYearColumns(y)` merges two sources → sorts by Julian Day:
1. Fixed festival **anchors** (ר"ה א', ר"ה ב', יו"כ, סוכות, שמח"ת, פסח א',
   שביעי של פסח, שבועות) — always present regardless of weekday.
2. Every Shabbat from the parasha cycle (`genParshaTable`, `israel=true`);
   parshaless Shabbatot that coincide with an anchor day are skipped, the rest
   become "שבת חול המועד".

`computeFullYear(y)` = for each column: `computeTimes` (NOAA sunset →
candle = sunset−20, havdala = sunset+30) + `pickBlocks` (elevator template).

`render` / `findAutoIndex` / `goToToday` = UI.

## Conventions / gotchas

- **Hebrew calendar + parasha engine** are hand ports of Python `pyluach`
  (validated to 0 errors over 1950–2150 daily and Hebrew years 5700–5920).
  Don't "fix" them casually — reproduce against pyluach if in doubt.
- **Rosh Hashana is split** into two records: `parts:'entrance'` (א', entrance
  time only) and `parts:'exit'` (ב', exit time only) — the chag is continuous,
  no havdala between the days. This is also why day 2 shows when it lands on a
  weekday (e.g. Sunday in תשפ"ז).
- **`edgeNoun(entry, weDate)`** decides the label noun for *every* user-facing
  string: `שבת` / `חג` / `שבת וחג`, from `week_end`'s weekday and whether
  `dayType ∈ YOMTOV_TYPES` (chol-hamoed Shabbatot are NOT yom tov here).
- **Chag ending on Friday** flows straight into Shabbat → its exit time is
  suppressed (`computeFullYear`); the following Shabbat record carries the real
  motzaei. ר"ה ב' on Friday then has no times → `render` shows a note.
- **`goToToday`** rolls forward to next Hebrew year's Rosh Hashana during the
  gap between a year's last Shabbat and 1 Tishrei (when `todayHebrewYear()`
  still returns the outgoing year). `state.autoYear` = the year actually landed
  on; the status badge shows only when `state.baseYear === state.autoYear`.
- **`TEMPLATES`** (elevator windows) is a fixed תשפ"ו library — there is no
  formula. Chag → exact template by `day_type`; regular Shabbat → nearest by
  candle-lighting minutes. To change elevator policy, edit `TEMPLATES` only.
- Dates flow as `YYYY-MM-DD` strings; `parseISO` builds **local** `Date`s.
  Weekday: JS `getDay()` (0=Sun…6=Sat) in the UI; `hebWeekday` (1=Sun…7=Sat)
  in the calendar engine — don't mix them.

## Testing (no framework)

Run the script headless with a minimal `document` stub and eval the `<script>`
body. Pattern used repeatedly here:

```js
const els = {};
const mkEl = id => ({ id, value:'', textContent:'', innerHTML:'',
  addEventListener:(ev,fn)=>{ listeners[id+':'+ev]=fn; }, style:{} });
global.document = { getElementById: id => els[id] ||= mkEl(id) };
eval(fs.readFileSync('script-body.js','utf8'));  // lines between <script> and </script>
```

Then drive `rebuildYear(y)`, `render(i, status)`, `goToToday()`, or the stored
`listeners['yearNextBtn:click']()`. Override `Date` to simulate "today".
Sanity sweep: render every entry for Hebrew years ~5760–5850 and assert
0 throws + chronological `week_end` order.

## Working agreements

- Commit/push only when asked. Branch is `main`; pushing deploys.
- Keep `README.md` updated with any behavior change.
- Preserve the "single file, zero dependencies, works offline" property.
