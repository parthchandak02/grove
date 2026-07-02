# grove — Creative Brief

## Concept
A bioluminescent forest grows on screen. Each tree represents a world city, its form shaped by the day's real sunlight data from the Sunrise-Sunset API. Longer day length grows taller trunks; dawn-to-sunset duration controls branch complexity. Bioluminescent particles drift among the trees, pulsing with the day's light rhythm.

**Narrative Pattern:** Annotation — city name labels always visible near each tree, tap to reveal full sunlight data.

**Tier 2 Patterns (2/2):** Quiet Zone Detail Panel + Hero Number

## Three Dials
- DESIGN_VARIANCE: 6 (semi-organic, trees at varying horizontal positions, branch angles vary by data, but the 5-tree structure is orderly)
- MOTION_INTENSITY: 5 (gentle sway via sine wave on trunks and branches, bioluminescent particle drift, slow pulse on glow nodes)
- VISUAL_DENSITY: 4 (5 trees, 15-25 bioluminescent particles, city labels, chrome elements)

## Data Source
- API: `https://api.sunrise-sunset.org/json?lat={lat}&lng={lng}&date=today&formatted=0`
- Cities (5 global):
  1. San Francisco: lat=37.7749, lng=-122.4194
  2. Tokyo: lat=35.6762, lng=139.6503
  3. London: lat=51.5074, lng=-0.1278
  4. Sydney: lat=-33.8688, lng=151.2093
  5. Dubai: lat=25.2048, lng=55.2708
- Response fields used: `day_length` (seconds), `sunrise` (ISO), `sunset` (ISO), `solar_noon` (ISO)
- Fallback: hardcoded typical day lengths for each city (~14h SF, ~10h Tokyo, ~16h London, ~10h Sydney, ~13h Dubai in July)

## Visual Style
- **Preset:** Dark — Midnight (forest bioluminescence)
- **Palette (locked JS const):**
```js
const PALETTE = {
  bg: '#0a0f0d',         // very dark forest green
  layer: '#0f1a14',      // slightly lighter layer
  accent1: '#2e7d32',    // deep green — trunk/structure
  accent2: '#4caf50',    // green — branches
  accent3: '#76ff03',    // bright neon green — bioluminescence
  biolume: '#b2ff59',    // pale green glow
  text: 'rgba(255,255,255,0.5)',
  label: 'rgba(255,255,255,0.7)',
  glowAlpha: 'rgba(118,255,3,0.08)',
};
```
- **Stroke weight scale:** `{hair:0.5, fine:1, medium:2, heavy:4}`
- **Alpha hierarchy:** `{bg:0.08, field:0.25, active:0.7, label:0.45}`
- **Shape language:** Organic — bezier curves, quadratic curves, rounded shapes. No straight lines except for two structural reference lines.
- **Glow budget:** 1 glow color (`#76ff03`), blur ≤ 30px, ≤ 25 glowing entities per frame
- **Font family:** system-ui, sans-serif for all text elements

## Encoding Contract

| Data Field | Visual Channel | Range | Inverse Function | "Bigger means..." | Legend? |
|---|---|---|---|---|---|
| day_length | trunk height | 8h (28800s) → treeH 80px, 16h (57600s) → treeH 200px | mapToRange(trunkH, 28800, 57600, 80, 200) | more daylight → taller tree | yes |
| sunrise→sunset duration | branch count + complexity | 12h→2 branches, 14h→4 branches, 16h→6 branches | mapToRange(branches, 12*3600, 16*3600, 2, 6) | longer daylight period → more branches | no |
| solar_noon (time of day) | trunk lean angle | 10am→-5deg, 2pm→+5deg | mapToRange(lean, 10, 14, -5, 5) | later solar noon → tree leans right | no |
| city position | tree X position | evenly distributed across canvas width 100-900px | (cityIndex / 4) * canvasWidth | N/A (spatial) | no |
| bioluminescence intensity | particle count + glow radius | sunrise→5 particles/10px, solar_noon→15 particles/25px, sunset→5 particles/10px | mapToRange(particles, 0, 86400, 5, 15) for time-of-day | more light available → more glow | yes |

**Inverse for hit-test:** `getClosestTree(tapX, tapY)` — returns the tree whose bounding box contains the tap point. Each tree stores: `{name, dayLength, sunrise, sunset, solarNoon, x, topY, bottomY}`.

## Interaction
- **Tap a tree:** show Quiet Zone overlay with city name, day length (hours:min), sunrise/sunset times, solar noon time, and a description "How much daylight [city] gets today"
- **Tap empty area:** update Hero Number to show the overall stats
- **After 3s idle:** Quiet Zone overlay fades to 0.1 opacity (leaving labels visible)
- **Auto:** trees sway gently with subtle sine-wave oscillation, particles drift and pulse

## Communication Requirements

**Tier 1 (mandatory):**
1. **Persistent Data Stamp** — "Sunrise-Sunset API · updated [time] · [live|fallback]" at bottom-left (10px system-ui, opacity 0.4)
2. **Data Process Honesty** — if API fails, show "fallback" label in stamp, never "live"
3. **One-to-One Sensory Mapping** — taller tree = more daylight, more branches = longer day. Non-obvious: bioluminescence = time of day. Legend explains this.

**Tier 2 (2 of 2):**
4. **Quiet Zone Detail Panel** — DOM overlay (not canvas text) when a tree is tapped. Shows: city name (16px, bold), day length (bold, hero-sized), sunrise/sunset times, and a 1-line description. Styled with semi-transparent dark green bg, green border, system-ui font. Fades after 3s idle.
5. **Hero Number** — Large (48px) day-length value for the most recent selection, or the global average. Shown at top-right area, with 1-line label "hours of daylight". Style: biolume green, monospace digits.

**Coherence:**
6. Stroke weight scale: `{hair:0.5, fine:1, medium:2, heavy:4}` as JS const
7. Alpha hierarchy: `{bg:0.08, field:0.25, active:0.7, label:0.45}` as JS const
8. Motion tokens: `--motion-fast:160ms, --motion-base:240ms, --motion-slow:360ms, --ease-out-expo` as CSS custom properties
9. AI Slop Score < 30: no Inter/Roboto/Open Sans (use system-ui), layered background (not plain), at least 1 animation (tree sway)

## Canvas Code Requirements (MANDATORY)
1. Single HTML file, no external dependencies, no CDN libraries
2. Config object `C` at top with all tunable values (NUMBER_OF_TREES, SNAP_THRESHOLD, etc.)
3. DPR-aware canvas: `Math.min(window.devicePixelRatio || 1, 2)`, `ctx.setTransform(dpr, 0, 0, dpr, 0, 0)`
4. Canvas CSS: `position:fixed;top:0;left:0;width:100%;height:100%;display:block`
5. Touch guard pattern: `let tapping=false; ontouchstart` gate for tap vs. scroll
6. First frame never blank — pre-render trees with fallback data before API call completes
7. Realistic fallback data if API fails
8. Short description text (`#desc` div, bottom center, muted opacity, system-ui font)
9. Full-viewport, mobile-responsive
10. Works offline after first load (no CDN deps)
11. `.nojekyll` file at project root

## Interaction Affordances
- Trees have a gentle pulsing glow at the canopy (subtle, visible when bioluminescence is high)
- On first visit, a single pulse wave emanates from each tree (staggered, 0.3s apart)
- After 3s inactivity, Quiet Zone panel fades to 0.1 opacity (city labels remain)
- Tappable trees have a soft hover glow when mouse is near

## Implementation Notes for Tree Generation
- Each tree is drawn recursively: trunk (1 segment), then 2 main branches, then sub-branches from each
- Trunk: bezier curve from ground upward, width tapers from `heavy` to `fine`
- Branches: quadratic curves, width from `medium` to `hair`
- Leaf canopy: small circles clustered at branch tips using `accent3` color with `shadowBlur: 15-25`
- All coordinates calculated from city data, no random values
- Trees rendered from left to right so broader trees appear behind narrower ones
- Use `ctx.globalCompositeOperation = 'lighter'` for bioluminescence glow layer, then draw labels on a separate pass with normal composite
