# CLAUDE.md

This file provides guidance to Claude Code when working with code in this repository.

## Project Overview

FitKit is a privacy-first, offline-capable fitness companion app — a single-page HTML application that provides AI-powered exercise form analysis via webcam or video. All AI inference runs locally in the browser using TensorFlow.js + MoveNet.

## Tech Stack

- **Pure static HTML/CSS/JS** — no build tools, no package manager, no framework
- **TensorFlow.js 4.17** (CDN: `jsdelivr`) + **MoveNet SinglePose Lightning** for pose estimation
- **Canvas API** for skeleton overlay rendering
- **localStorage** for training log persistence
- CDN mirror for China users: `jsd.onmicrosoft.cn` (fallback documented in code)

## Development

- No build step, no npm, no test framework
- Open `index.html` directly in a browser or serve locally: `python -m http.server 8080`
- `test.html` contains standalone unit tests for core logic — open it in a browser to run
- MoveNet model is downloaded from tfhub.dev on first use (~10MB), then cached by IndexedDB
- For users in China, the model may fail to download; the app shows detailed troubleshooting instructions

## Architecture

### File Structure

```
fitkit-demo/
├── index.html      # Entire application (HTML + CSS + JS, ~1200 lines)
├── test.html       # Unit tests for core biomechanics logic
├── docs/
│   └── PRD.md      # Product requirements document
└── README.md
```

### Application Structure (index.html)

The app uses a single-page architecture with tab-based navigation. Page visibility is toggled via CSS `.page.active`.

**Global state** is managed in the `S` object (~line 576):
- `currentPage` — active page id
- `detector` — MoveNet detector instance
- `cameraStream` / `isRunning` / `repCount` — video analysis state
- `videoSource` — one of `camera | demo | upload | url`
- `timerRunning` / `timerRemaining` / `timerPhase` — interval timer state

**Pages (sections):**
| Page | ID | Description |
|------|-----|-------------|
| Home | `page-home` | Toolbox dashboard with tool cards |
| Video Analysis | `page-analysis` | Camera/demo/upload/URL input + MoveNet analysis |
| Timer | `page-timer` | Customizable work/rest interval timer |
| Training Log | `page-log` | Session history from localStorage |
| Exercise Library | `page-library` | Reference with expandable exercise details |

**Navigation:** `navigateTo(page)` hides all pages and shows the target page. Sidebar `.nav-item` click handlers call it.

### Pose Estimation Engine

Located in the Video Analysis section (~line 752):

1. **Model loading** (`loadModel`): Sets WebGL backend, creates MoveNet detector with `MODEL_CONFIG`. First load downloads ~10MB from tfhub.dev. Shows detailed error messages for common failure modes (network blocked, WebGL unavailable).

2. **Video sources** — four tabs managed by `S.videoSource`:
   - `camera` — `startCamera()` uses `getUserMedia` (640x480, user-facing)
   - `demo` — Pre-set Mixkit video URLs (may not load in China)
   - `upload` — `loadVideoFile()` creates Object URL from File API
   - `url` — `loadVideoFromUrl()` sets video.src directly

3. **Detection loop** (`detectPose`): Reads video frame → runs detector → draws skeleton → runs exercise analyzer → updates rep counter. Runs via `requestAnimationFrame`.

4. **Keypoint constants** (`KP`): COCO 17 keypoints (NOSE through RIGHT_ANKLE). Used by all analyzers.

5. **Skeleton drawing** (`SKELETON`): 12 bone pairs connecting keypoints. Color-coded: green for good form, red for errors, blue default.

### Exercise State Machines (`ExerciseAnalyzers`)

Each exercise has an `analyze(keypoints, prevState)` method returning a result object. All analyzers share a common debounce mechanism (`DEBOUNCE_FRAMES = 4`) to prevent flickering between phases.

**squat analyzer** — Uses bilateral keypoints:
- **Measurements:** Knee angle, hip angle, torso lean ratio, knee valgus ratio, hip-below-knee depth check, hip-knee coordination ratio
- **Phases:** UP → DESCENDING → BOTTOM → ASCENDING → UP
- **Errors (priority order):** knee_valgus (bad) → torso_lean (warn) → shallow (warn) → knee_first (warn) → hip_high (warn)
- **Targets:** kneeBottom=85°, kneeTop=165°, hipAngleMin=60°, torsoMax=0.35, valgusMax=0.12

**curl analyzer** — Uses right-side keypoints only:
- **Measurements:** Elbow angle, elbow drift ratio, shoulder flexion
- **Phases:** DOWN → CURLING → UP → RELEASING → DOWN
- **Errors:** elbow_drift (warn) → shoulder_swing (bad) → partial_extension (warn) → partial_flexion (warn)
- **Targets:** elbowTop=45°, elbowBottom=155°, driftMax=0.25, shoulderFlexMax=0.12

**press analyzer** — Uses right-side keypoints:
- **Measurements:** Elbow angle, elbow flare, wrist overhead
- **Phases:** UP → DESCENDING → BOTTOM → ASCENDING → UP
- **Errors:** elbow_flare (bad) → partial_lockout (warn) → shallow_bottom (warn)
- **Targets:** elbowBottom=80°, elbowTop=160°, elbowFlareMax=0.7

Each analyzer also has a `getSuggestions()` method that maps error codes to Chinese-language improvement suggestions with emoji severity indicators (🔴 critical, 🟡 warning, ✅ good).

**Form error tracking** (`formErrors`): Cumulative error counts per exercise per error code, used by `generateSuggestions()` for post-session reports.

### Biomechanical Utilities

- `angleBetween(a, b, c)` — Three-point joint angle using arctan2 of cross/dot product. Returns degrees.
- `verticalAngle(a, b)` — Angle of a segment relative to horizontal. Used for body line checks.

### Interval Timer

- Customizable work/rest intervals with round tracking
- Display format: `MM:SS` with phase label
- Timer runs via `setInterval` at 100ms resolution

### Training Log

- Records stored in `localStorage` under key `fitkit_log`
- Each entry: date, exercise name, rep count
- Rendered by `renderLogs()` with formatted date strings

### Test Suite (test.html)

- Minimal test framework: `test(name, fn)`, `assert(cond, msg)`, `assertEqual`, `assertClose`
- `makeKP(overrides)` generates mock MoveNet keypoints for biomechanics testing
- Covers: `angleBetween` correctness, squat phase detection, knee valgus, depth, curl elbow drift, press elbow flare, debounce logic
- Pure JS, no dependencies — just open in browser

## Important Conventions

- All code in one file — don't split into modules unless the user asks
- CSS variables defined in `:root` — use them, don't hardcode colors
- Chinese-language UI — all user-facing text is Chinese
- No external API calls — everything is client-side
- Error messages for model loading include China-specific troubleshooting (tfhub.dev blocked → self-host model, CDN mirror switch)

## Common Gotchas

- `videoEl.srcObject` vs `videoEl.src` — camera uses `srcObject`, file/URL uses `src`. When switching sources, always call `stopCamera()` first to release the stream.
- `videoEl` autoplay: camera mode needs `autoplay` attribute; file/URL mode must NOT have it (or subsequent src changes may fail)
- Autoplay policy: `videoEl.muted = true` is set for file playback to bypass browser autoplay restrictions
- MoveNet `minPoseScore: 0.2` (lowered from default) — important for detecting partially visible poses
- Keypoint `score < 0.3` causes the analyzer to return `null` for that frame — the detection loop skips analysis, keeps previous state
- Debounce requires `DEBOUNCE_FRAMES` consecutive frames of the same raw phase before a phase transition occurs
- On `navigateTo('log')`, `renderLogs()` is called explicitly because log data may have changed
- The demo video URLs point to Mixkit (hosted overseas) — they will not load for most China users

## CSS Patterns

- Dark theme with CSS custom properties: `--bg`, `--surface`, `--surface2`, `--border`, `--text`, `--text2`, `--accent`, `--green`, `--yellow`, `--red`
- Responsive: sidebar collapses to icon-only at 768px, `tool-grid` switches to single column
- `.page` elements hidden by default; `.page.active` shown
- Interactive elements use `.btn` base + modifier classes: `.btn-primary`, `.btn-danger`, `.btn-outline`
- Form feedback types: `.good` (green), `.warn` (yellow), `.bad` (red)
