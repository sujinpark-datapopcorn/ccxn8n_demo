# 나의 우주 대시보드 — Visual Redesign

Status: approved 2026-04-21

## Goal

Replace the minimal NASA dashboard with a visually rich single-page
experience that doubles as a demo of what n8n + a static HTML drop can
produce. The data pipeline stays identical; only the payload Code node
and one added translation node change.

## Non-goals

- No change to NASA API choice, Slack channel, GitHub Pages hosting.
- No server-side rendering; all animation lives in the HTML shipped to
  GitHub Pages.
- No interactive filtering or drilldown beyond the APOD toggle.

## Workflow changes

Add one node between `NASA - APOD` and the existing `Merge`:

```
Schedule ┬─ NASA APOD ─── OpenAI Translate APOD ─┐
         ├─ NASA Asteroid Neo Feed ───────────────┤ Merge(4) → Build → GitHub → Slack
         └─ NASA DONKI Solar Flare ───────────────┘
```

`OpenAI - Translate APOD`
- Type: `n8n-nodes-base.openAi` (or `@n8n/n8n-nodes-langchain.openAi`
  if only the langchain-flavored node is available — pick whichever
  the API `/types/nodes` returns as installed).
- Credential: `openAI account3`.
- Model: `gpt-4o-mini`.
- Operation: chat completion, JSON response format.
- System prompt: "영어 천문 해설을 자연스러운 한국어로 번역한다.
  원문 의미를 유지하되 문장을 짧고 매끄럽게. JSON `{korean: string}`만 반환."
- User prompt: `={{ $json.explanation }}`.
- `onError: "continueRegularOutput"` so a translation failure still
  ships the original APOD.
- Passthrough: merge original APOD fields back into the output so the
  downstream node sees both `title`, `url`, `explanation` (original) and
  `korean` (translated). Done by post-processing in Code expression, or
  easier: Merge configured to keep both inputs.

The `Merge` node jumps from 3→4 inputs: APOD-translated, Asteroid,
Flare, and the original APOD pass-through is included in the APOD
branch's translated item (since OpenAI node receives the full APOD
json item and we instruct it to echo `title`/`url`/`media_type` into
the returned JSON alongside `korean`). This keeps the Merge wiring
identical.

Trade-off accepted: if OpenAI returns malformed JSON, Build Dashboard
HTML falls back to English explanation only. Empty-state, not error.

## HTML layout

Single `index.html`, no external build step. CDN-only runtime deps:
- Three.js r160 (`https://unpkg.com/three@0.160.0/build/three.min.js`)
- ECharts 5.5 (already in use)

Section order top-to-bottom:
1. **Header** — `h1` "나의 우주 대시보드" rendered with orange neon
   text-shadow stack (`#ff8a3c` → `#ffb36a` blur). Background `canvas`
   painting ~200 stars with per-star random opacity sine wave at ~2s
   period, layered behind the title.
2. **Stat strip** — three counters (근접 소행성 / 잠재 위험 / 7일 플레어).
3. **Hero — Three.js Sun** — 600 px tall `canvas`. Sphere radius 2,
   64×64 seg, `MeshBasicMaterial` with `map` = Solar System Scope 2k
   sun texture URL. Corona via second mesh slightly larger using a
   custom `ShaderMaterial` with Fresnel-based alpha and color uniform
   `uCoronaColor`. Rotation `mesh.rotation.y += 0.002` per frame.
4. **APOD** — title + image + Korean explanation by default. Small
   button "원문 보기" toggles between `korean` and `explanation`. If
   `korean` is empty, hide the toggle.
5. **NEO list** — CSS grid of cards. For each asteroid show:
   - 이름 (`name`)
   - 지름 (평균 m) — rounded `(min+max)/2` of
     `estimated_diameter.meters`
   - 거리 — `close_approach_data[0].miss_distance.lunar` → "3.21 LD"
   - 속도 — `close_approach_data[0].relative_velocity.kilometers_per_second` → "14.5 km/s"
   - 잠재 위험 red badge when `is_potentially_hazardous_asteroid`.
   - Sort by ascending lunar distance. Cap visible cards at 12
     (unlikely to exceed with one-day feed but safe).
6. **Flare scatter** — ECharts, dark theme. Only rendered when
   `flareCount > 0`.

## Corona color rule (shared)

Scan the 7-day flare list. Take the **highest** class letter, rank
order `X > M > C > B > A`:
- `X` present → `#ff3030`
- `M` present (no X) → `#ff9040`
- Otherwise (C/B/A or empty) → `#ffd040`

This color is injected into:
- Three.js corona `uCoronaColor` uniform.
- `h1` glow accent (subtle — underline rule or glow tint).

## Flare scatter spec

- X axis: `peakTime` parsed to timestamp, category-to-time.
- Y axis: log scale. `intensity = {A:0.001,B:0.01,C:0.1,M:1,X:10}[letter] * magnitude`.
- Point color: by class letter (A #94a3b8, B #38bdf8, C #84cc16, M #ff9040, X #ff3030).
- Point size: `max(6, magnitude * 10)`.
- Tooltip shows classType, peakTime, sourceLocation if present.

## Build Dashboard HTML node changes

- Input sourcing stays the same: `$('NASA - APOD').first()`,
  `$('NASA - Asteroid Neo Feed').all()`,
  `$('NASA - DONKI Solar Flare').all()`. APOD now includes
  `korean` (may be empty).
- Output `summary` string continues to feed Slack; no change needed.

## Failure modes handled in the renderer

| Case | Render |
|---|---|
| APOD 404 / no title | Empty-state card "APOD 데이터를 불러오지 못했습니다" |
| No asteroids | Grid shows "데이터 없음" |
| No flares | Scatter hidden, replaced with "지난 7일 태양 플레어 관측 없음" |
| Translation missing | Show English only, hide toggle |
| Three.js texture load fails | Corona glow continues, texture defaults to solid orange |

## Out of scope

- Caching historical runs for trend charts.
- Mobile-specific layout beyond responsive grid.
- Analytics or click tracking on GitHub Pages.
