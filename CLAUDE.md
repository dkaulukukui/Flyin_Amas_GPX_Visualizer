# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Flyin Amas GPX Track Animator is a **single-file web application** (`index.html`) that visualizes and animates GPS tracks from GPX files with video export capabilities. All processing happens client-side in the browser with no backend server.

**Live deployment**: GitHub Pages (https://dkaulukukui.github.io/Flyin_Amas_GPX_Visializer/)

## Architecture

### Single-File Application Structure

The entire application is contained in `index.html` with this organization:

1. **HTML Head**
   - External dependencies via CDN (React 18, Leaflet 1.9.4, Babel Standalone, mp4-muxer)
   - Embedded CSS styles for entire application
   - All styles use consistent color scheme: `#fc4c02` (primary orange), `#2d2d2d` (dark backgrounds)

2. **React Application**
   - Single root component: `GPXTrackAnimator`
   - Uses React hooks exclusively (no class components)
   - JSX transpiled in-browser using Babel Standalone

3. **Version Management**
   - `APP_VERSION` constant with comprehensive changelog comments
   - **CRITICAL**: Increment version number when making changes
   - Format: `major.minor.patch` (semantic versioning)
   - Current version: 3.12.0 (as of last update)

### Key Technical Patterns

#### GPX File Processing
- Files uploaded via drag-and-drop or file input (50 MB size cap per file); `.json` files are opened as
  projects/presets; tracks can also load from startup links (see Projects, Presets & Startup Links)
- XML parsing via DOMParser; malformed XML is rejected (`parsererror` check)
- Extracts `<trkpt>` elements with `lat`, `lon`, optional `time` attributes
- Points with missing/non-numeric/out-of-range coordinates are skipped; invalid timestamps are ignored
- Each track assigned unique ID, random color, default label from filename (part before first underscore)
- **Security**: track labels are HTML-escaped via `escapeHtml()` before being injected into Leaflet divIcon HTML (XSS prevention)
- Tracks stored in React state as array of objects with structure:
  ```javascript
  {
    id: string,
    label: string,
    color: string,
    points: [{lat, lon, time?}],
    startTime?: Date,
    endTime?: Date,
    duration?: number
  }
  ```

#### Map Rendering (Leaflet Integration)
- Map initialized with `preferCanvas: true` for better video export
- **Compositing layer (v3.11.1) — do not remove**: `.leaflet-tile-pane { will-change: transform }`. With
  Leaflet 3D disabled, tiles and tracks were painted as one layer and iPhone Safari re-rendered the tiles
  under the changing tracks' bounding box differently (a darker box following the boat, rest washed out).
  Don't also promote the overlay (track) pane: on iPhone that left torn, stale copies of the lines as the
  camera moved (v3.11.0). Doesn't bring back the desktop tile seams (those came from per-tile 3D layers).
  Confirmed clean on a real iPhone together with the canvas track renderer (v3.12.0)
- **Track layer renderer (v3.12.0)**: `L.canvas()` (the map is created with `renderer: L.canvas()`). SVG was
  faster in emulation but its lines left torn, stale copies ahead of the boats on iPhone. Exports never depend
  on these layers: `captureFrame` draws tracks with `drawTracksDirect` (it still copies any Leaflet canvases
  if present). Don't reintroduce a "<canvas> required" check in `startRecording`
- `window.L_DISABLE_3D = true` is set before Leaflet loads (tiles positioned with left/top, not
  per-tile 3D layers) — required for seam-free CSS scaling of the preview; it also forces
  whole-number zoom (Leaflet ignores `zoomSnap` without 3D), which the design relies on

#### Frame Scaling / WYSIWYG Preview (v3.2.0)
- **Reference frame**: every user-facing size (zoom level, line widths 2/4px, marker radius, label/
  legend/title font sizes, paddings, fitBounds padding) is defined for a frame whose long side is
  `REFERENCE_LONG_SIDE` = 1920px (i.e. the 1080p export)
- **Preview**: `.map-stage` (fills the map area) → `.map-frame` (letterboxed to the export aspect
  ratio when `matchVideoFrame`) → `.map-surface` (reference size, e.g. 1920x1080, CSS
  `transform: scale(displayScale)`) → legend, title, `#map`. Leaflet handles scaled containers
  for mouse/touch. `--ui-scale` counter-scales Leaflet's zoom buttons/attribution
- **Export**: `getExportRenderPlan(w, h)` → the map is resized to `mapScale` × reference
  (`mapScale` = power of two ≥ frame scale: 1 for 720p/1080p, 2 for 4K and 2560x1920), zoom is
  `zoomLevel + log2(mapScale)` (whole number), and `captureFrame` draws at `drawScale` ≤ 1 into the
  video canvas. `exportScale` state (= mapScale while exporting, else null) makes Leaflet layers
  render at that scale and removes the surface's CSS transform
- `getFitView(bounds, scale)`: fit-all view snapped to a whole reference zoom — used by the preview
  zoom effect and renderExportFrame so both frame the same area
- `getLegendLayout(fontSize, scale)`: legend geometry shared by the DOM legend and the canvas legend
- **Satellite tiles** (`MAP_IMAGERY`, `mapImagery` setting, Camera tab): Esri **Clarity** (default, native to
  z19) or standard Esri World Imagery (native to z18), both with `crossOrigin: 'anonymous'` for CORS. Layers
  live in `map._imageryLayers`; an effect shows the selected one. Clarity tiles load through two HTTP
  redirects (slower first load); both services send `Cache-Control: max-age=86400`
- **Tile prefetch (v3.9.0)**: Leaflet only requests on-screen tiles, so a moving follow camera shows its
  leading edge still loading (a visible band on mobile). `prefetchFollowTiles(fromProgress, zoom, w, h)`
  samples `getFollowCenter` over the next `PREFETCH_LOOKAHEAD_SECONDS` of video, computes the tile URLs
  (`getTileUrlsForView`, same template/zoom as the layer) and downloads them with `new Image()`
  (`crossOrigin='anonymous'` so the HTTP cache entry is reused), max `PREFETCH_MAX_IN_FLIGHT` at once.
  Called by a throttled preview effect and by `renderExportFrame`
- Initial map center configurable via `INITIAL_MAP_CENTER` (default: Oahu, Hawaii)
- Polylines drawn using Leaflet canvas renderer. **Track layers are persistent (v3.7.0)**: full-track
  previews are rebuilt only when tracks/visibility/scale change (`previewLayersRef`); each track's
  animated line, marker, and label live in `animatedLayersRef` and are moved per frame with
  `setLatLngs`/`setLatLng`, recreated only when their style key changes. Don't go back to
  remove-and-re-add per frame — it starved phones of time to load tiles
- **Camera zoom sync (v3.7.0)**: a `zoomend` handler turns user zooms (+/-, wheel, pinch) into
  `zoomLevel` while `followCameraActiveRef` is true (zoom-to-track on, 0 < progress < 100, not
  exporting), clamped to `ZOOM_LEVEL_MIN`..`ZOOM_LEVEL_MAX`. **Wrap any app-initiated view change in
  `moveMapProgrammatically(...)`** so it isn't mistaken for a user zoom
- The preview scoreboard canvas is sized/positioned to the panel (drawn on an off-screen scratch
  canvas via `drawScoreboard`, which returns the panel bounds); `.leaflet-container` background is dark
  so tiles still downloading don't flash white
- Labels positioned at current track positions using custom DOM markers (`.map-label`)
- Legend rendered as absolute-positioned overlay (`.map-legend`)

#### Animation System
Two animation modes with different timestamp handling:

**Simultaneous Mode** (default):
- Synchronizes tracks by absolute GPS timestamps
- Calculates earliest start and latest end across all tracks
- Progress maps to absolute time span, showing tracks at their actual time

**Sequential Mode**:
- Plays tracks one after another in upload order
- Each track uses full 0-100% progress range

**Aligned Mode (Track Comparison, v2.6.0)**:
- User picks a common point on the map (`alignPoint`) with a radius circle (`alignRadius`, meters)
- `alignmentMatches` (useMemo) finds each track's match inside the circle by `alignMethod`:
  `'closest'` (min distance), `'earliest'` (first pass), `'latest'` (last pass); no match → `null`
- When `alignmentActive`, all matched tracks animate from their match point using **real elapsed
  time from their own crossing** (`alignedTiming.spanMs` = longest remaining time); unmatched
  tracks show only as dimmed preview; point-index pacing fallback for timestamp-less tracks
- Overrides the animationStyle select (disabled while active)

**Single source of truth**: `getHeadState(track, progressPercent)` (component scope) returns where a
track's head is — `{from, index, frac}` (visible points `points[from..index]` plus a head `frac` of
the way to the next point) or null — for all three modes. `getVisiblePoints` is a thin wrapper used by
the polyline render effect, zoom-follow, `renderExportFrame`, and `drawTracksDirect`; the scoreboard
stats read `getHeadState` directly. Change animation behavior ONLY in `getHeadState`.

**Head interpolation (v3.1.0)**: `getVisiblePoints` returns the reached GPS fixes plus one
interpolated "head" point (`lerpPoint`) partway to the next fix — by time fraction in time-based
modes, by fractional point count in index-based modes. Consumers treat the last returned point as
the current position, so marker, label, and follow camera glide instead of stepping per fix.
Interpolation only happens between adjacent timed fixes; uses precomputed `track.timesMs`.

**Animation Loop**:
- Uses `requestAnimationFrame` for smooth rendering
- **Preview pacing matches export exactly**: `progressIncrement = (deltaTime / (exportDuration * 1000)) * 100`
- `exportDuration` is a direct user input (v3.0.0): Duration number field, 5-300s, default 60s
- Controlled by `isPlayingRef` to avoid stale closure issues

**Track trimming (v3.0.0)**:
- `tracks` keeps the original uploaded points; per-track `trimStart`/`trimEnd` indices are set by
  the ✂ Trim sliders in the track list
- `effectiveTracks` (useMemo) applies the trim and recomputes startTime/endTime/duration, then
  applies GPS smoothing (`smoothPoints`, v3.1.0) and precomputes `timesMs` (ms per point, NaN if missing) —
  **all** animation, alignment, zoom, stats, and export logic consumes `effectiveTracks`, never
  `tracks` directly (UI lists and labels still use `tracks`)

**Example tracks (v3.0.0)**:
- `examples/Kai.gpx` and `examples/Kimo.gpx` are fetched on startup and loaded with
  `isExample: true`; `handleFileUpload` filters them out as soon as the user uploads their own files
- `.gitignore` excludes `*.gpx` but exempts `examples/*.gpx`

**Scoreboard & Stats (v3.4.0)**:
- The legend is a scoreboard: header = race-scope stats (or "TRACKS"), one row per track with a column
  per track-scope stat, footer = pair-scope stats. `drawScoreboard(ctx, w, h, scale, progress, reserve)`
  draws it on canvas — used by BOTH the preview overlay (`<canvas class="map-scoreboard">` on the
  surface, redrawn after every render) and `captureFrame`, so preview and export match exactly
- `STAT_DEFINITIONS` registry (top level). **To add a stat**, add an entry with `id`, `label`, `scope`
  ('race' | 'track' | 'pair'), `compute(ctx)` → number|null, `format(value, units)`, `template(units)`
  (widest text; fixes column width so the panel doesn't jitter), optional `requires` (point field) and
  `short` caption. Add it to `DEFAULT_STATS_ENABLED`. The UI checkbox is generated from the registry.
  Sensor data: add the field to `GPX_EXTENSION_FIELDS` (parsed by local name from `<extensions>`);
  per-track ctx exposes `field(key)`, `distanceM`, `speedMps()`; race ctx `elapsedMs`; pair ctx `distanceM`
- `computeScoreboardStats(progress)` builds the contexts: race clock from `getRaceClockMs` (aligned: tau;
  simultaneous: absolute span; sequential/untimed: null → "—"), distances from `track.cumDistM`
  (precomputed in `effectiveTracks`), speed averaged over `SPEED_WINDOW_MS` (10 s), gap = haversine
  between the two `gapTracks` heads
- `statsStartProgress`: timer and distance are measured from this progress ("Set start here" button,
  orange marker on the playback bar); timer is negative before it, distance 0

**Visual Elements**:
- Full track preview (dimmed, togglable): Shows complete path at 30% opacity
- Animated track: Current progress shown at full opacity
- Position markers: White-rimmed circles at current position
- Track labels: Customizable labels that follow track position
- Centered track: optional single track to follow during zoom animation (`centeredTrackId`)

#### Video Export Architecture

**CRITICAL DESIGN (v2.4.0)**: Two export paths, chosen automatically at export time.

**Preferred path — WebCodecs single-pass MP4 export** (`exportWithWebCodecs`):
1. `pickAvcEncoderConfig()` probes `VideoEncoder.isConfigSupported` with H.264 codec candidates
   (High 5.2 down to Baseline 3.0) — returns null if WebCodecs or the `Mp4Muxer` global is unavailable
2. Creates an `mp4-muxer` Muxer with an in-memory `ArrayBufferTarget` (`fastStart: 'in-memory'`)
3. For each frame: `renderExportFrame()` renders to the composite canvas, then a `VideoFrame` with an
   **explicit timestamp** (`frame * 1e6/fps` microseconds) is encoded and closed immediately
4. Backpressure: waits while `encoder.encodeQueueSize > 4` so frames never pile up in RAM
5. `encoder.flush()` → `muxer.finalize()` → download MP4 blob

Why this is the preferred path:
- **No RAM accumulation**: frames go straight into the encoder (fixes out-of-memory crashes)
- **Exact FPS/duration**: explicit timestamps mean render speed doesn't affect video timing
- **MP4 output everywhere supported**: Chrome/Edge 94+, Firefox 130+, Safari 16.4+ (including iOS), Android Chrome
- **Faster**: single pass; no real-time playback phase

**Fallback path — MediaRecorder two-phase export** (older browsers only):
1. **Phase 1 (pre-render)**: renders every frame and stores it as a **compressed WebP/PNG blob**
   (`canvas.toBlob`, ~hundreds of KB each) — NOT raw ImageData (~8 MB each), to limit RAM use
2. **Phase 2 (playback)**: starts MediaRecorder on `captureStream(0)`, then for each stored frame:
   decode via `createImageBitmap`, draw, call `videoTrack.requestFrame()`, and wait until the exact
   target wall-clock time
3. MediaRecorder is started **only** for Phase 2 (critical for correct duration)
4. Duration diagnostics are logged to the console (FFmpeg.wasm correction was removed in v3.0.0)

If neither path is available (very old iOS), an alert with screen-recording instructions is shown.

**Shared helpers**:
- `renderExportFrame(frame, totalFrames, ...)`: sets progress, applies zoom-to-track, waits for tiles
  (`waitForTilesToLoad`), composites via `captureFrame` — used by both export paths
- `captureFrame(...)`: draws composite frame: map tiles → tracks → labels → legend → video title
- `drawTracksDirect(...)`: bypasses Leaflet canvas renderer for reliability (iOS/Safari fix);
  draws tracks, markers, and previews directly to the export canvas
- `restoreMapSize` / `downloadVideoBlob` / `finishExport`: cleanup helpers

**Platform support (v2.4.0)**:
- **Desktop Chrome/Edge/Firefox 130+/Safari 16.4+**: WebCodecs MP4 export
- **iOS 16.4+ / modern Android**: WebCodecs MP4 export (the old hard iOS block was removed)
- **Older browsers**: MediaRecorder WebM fallback
- **Very old iOS**: alert with screen-recording instructions

#### Projects, Presets & Startup Links (v3.6.0)
- **Project file** (`buildProject` / `applyProject`): `{format: 'flyinamas-gpx-project', version: 1,
  appVersion, savedAt, settings, video?, tracks?}`. `settings` = every `usePersistentState` value except
  `activeTab` (**new preferences are included automatically**). `video` = title, `statsStartProgress`,
  `gapTracks`/`centeredTrack` (indices into `tracks`), `alignment {point, active}`. `tracks[]` = name, label,
  color, trimStart/trimEnd, and raw points stored column-wise (`lat`, `lon`, `time` ms|null, plus
  `GPX_EXTENSION_FIELDS` keys when present). No `tracks` → a **preset**: `applyProject` applies settings only
- `applyProject` validates everything first (format/version, points, hex colors, clamped trims; settings via
  `coerceSetting`), confirms before replacing the user's own tracks, then replaces tracks + settings + video
  state. Bump `PROJECT_VERSION` only for incompatible format changes
- **Startup links** (`parseStartupLinks`, in the mount effect): `?project=URL`, `?gpx=URL[&label=][&color=]`
  (repeatable; label/color apply to the preceding gpx), `&title=`. Project first, then gpx (parallel,
  URL order), then title; one alert lists failures; examples are skipped when any link is given
- Shared helpers: `createTrack(points, name)` (used by `parseGPX` and projects), `addLoadedTracks(tracks,
  {title})` (uploads + links: replace examples, auto-title), `fetchText`/`fetchGpxTrack` (http(s) only,
  readable errors incl. CORS), `downloadBlob`

#### UI Layout (v3.5.0)
- **Sidebar** (`.sidebar`, flex column): `.sidebar-tabs` (from `SIDEBAR_TABS`; `activeTab` state) →
  `.sidebar-content` (scrolls; one tab rendered at a time) → `.sidebar-footer` (Export Video button,
  always visible). Tabs: **Tracks** (upload, track list + trim, animation style, smoothing, full-track
  preview, Compare Tracks, totals), **Camera** (zoom-follow, centered track, zoom level), **Overlays**
  (collapsible Video Title, Scoreboard & Stats, Track Labels, Watermark), **Export** (aspect ratio,
  resolution, preview framing, quality, FPS). Non-Tracks tabs show a hint until tracks are loaded
- **Transport bar** (`.transport-bar`, under `.map-stage` inside `.map-container`, a flex column):
  play/pause, reset, scrubber (`handleProgressBarPointerDown`, pointer events → mouse + touch; shows the
  stats-start marker), time readout, Duration input
- **Remembered settings**: `usePersistentState(settingsRegistry, key, default)` (module-level hook) is used
  instead of `useState` for preferences; values are saved to `localStorage`
  (`SETTINGS_STORAGE_KEY`) after any change and loaded at startup (`savedSettings`; wrong-typed values
  ignored, objects merged over defaults). `resetAllSettings()` restores every registered default. Not
  persisted on purpose: tracks, `videoTitle`, `statsStartProgress`, `statsGapIds`, `centeredTrackId`,
  `alignPoint`. **When adding a new preference, use `usePersistentState`**
- The aspect-ratio effect only resets `exportResolution` when it doesn't match the ratio (so a
  remembered resolution survives startup)

#### State Management

All state managed through React useState/useRef hooks:

**Component State** (useState):
- `tracks`: Array of loaded GPX tracks
- `isPlaying`, `progress`: Animation controls
- `isRecording`, `isRendering`, `renderProgress`: Export state
- `showAllTracks`, `showLabels`, `showLegend`: Display toggles
- `animationStyle`: 'simultaneous' or 'sequential'
- `zoomToTrack`, `zoomLevel`, `centeredTrackId`: Zoom controls
- `alignPoint`, `alignRadius`, `alignMethod`, `isPickingPoint`, `alignmentActive`: Track comparison/alignment
  (Leaflet circle + match markers are removed during export — preferCanvas would bake them into frames)
- `exportAspectRatio`, `exportResolution`: Export settings
- `exportDuration`: Video/preview duration in seconds — direct user input (5-300s, default 60)
- `labelFont`, `labelSize`: Track label styling (preview + export), labelSize default 16
- `legendSize`: legend/scoreboard text size in reference px (10-40, default 15); box sizes to content
- `showWatermark`, `watermarkText`, `watermarkSize`, `watermarkOpacity`: lower-right watermark (default on,
  `DEFAULT_WATERMARK_TEXT` "Made at Flyinamas.com", 12px, 50%). Geometry from `getWatermarkLayout()`
  (shared by the DOM `.map-watermark` and `captureFrame`); its `reserve` lifts a bottom-right legend/title
  above it. Leaflet's attribution sits top-right to keep the corner clear
- `matchVideoFrame`: letterbox the preview to the export aspect ratio (default true)
- `stageSize`: measured map-area size (ResizeObserver) used to fit the preview frame
- `exportScale`: export map scale while exporting, else null (see Frame Scaling)
- `statsEnabled` (id → bool, default timer/speed/distance), `statsUnits` ('imperial' default | 'metric'),
  `statsStartProgress`, `statsGapIds` (null → first two tracks), `scoreboardFont`: see Scoreboard & Stats
- `trackSmoothing`: 'off' | 'light' | 'medium' | 'strong' (default 'light') — Gaussian window radius
  from `SMOOTHING_RADIUS` (0/2/4/8 points); endpoints and timestamps preserved
- `trimOpenId`: Track whose ✂ Trim sliders are expanded (trim values live on the track objects)
- `exportFPS`: Target frame rate (15-60 FPS, default 30 FPS)
- `exportQuality`: Video quality preset ('low', 'medium', 'high', 'ultra')
- `exportFormat`: Current export format ('MP4', 'WebM (VP9)', 'WebM (VP8)')
- `videoTitle`, `titlePosition`, `titleSize`, `titleFont`, `titleColor`: Video title customization.
  `videoTitle` starts as `DEFAULT_VIDEO_TITLE` ("FlyinAmas") for the example tracks; the first upload
  replaces it with the track date unless the user changed it. titleSize default 28

**Refs** (useRef):
- `mapInstanceRef`: Leaflet map instance
- `polylineRefs`: Array of Leaflet polyline objects
- `isPlayingRef`: Synced with isPlaying for animation loop
- `mediaRecorderRef`, `compositeCanvasRef`, `compositeStreamRef`: Export rendering
- `renderCancelledRef`: Cancel signal for export (checked by both export paths)
- `exportStartTimeRef`: Export start timestamp for the overlay ETA

## Development Workflow

### Local Testing
```bash
# Serve locally (required for CORS on map tiles)
python -m http.server 8000
# OR
npx http-server

# Open http://localhost:8000
```

**IMPORTANT**: Do not open `index.html` directly as `file://` - CORS will block map tiles.

### Syntax Validation
The app is a single HTML file with in-browser Babel. To check the JS/JSX parses:
```bash
awk '/<script type="text\/babel">/{flag=1;next}/<\/script>/{if(flag){flag=0}}flag' index.html > /tmp/app.jsx
npx -y esbuild /tmp/app.jsx --outfile=/dev/null
```

### Making Changes

1. **Update version number** in `APP_VERSION`
2. **Add changelog entry** in comments above version
3. Test locally with `python -m http.server 8000`
4. Commit and push to GitHub (auto-deploys to GitHub Pages)

### Git Workflow
```bash
# Commit changes (increment version first!)
git add index.html
git commit -m "vX.Y.Z - Brief description of changes"
git push origin master

# GitHub Pages auto-deploys in 1-2 minutes
```

## Common Modifications

### Changing Map Tiles
Edit `MAP_IMAGERY` (top of the script): each entry has `label`, `url`, `maxNativeZoom`, `attribution`.
Any source must send CORS headers (`Access-Control-Allow-Origin`) or exports will fail.
- **Must include** `crossOrigin: 'anonymous'` for video export

### Adjusting Animation
- Zoom levels: `12` to `18`
- Video/preview duration: Duration input, 5-300s (single source of truth for both)

### Video Export Settings
User-configurable via UI:
- **Duration**: direct input, 5-300s (default 60s)
- **FPS**: 15-60 FPS (default: 30 FPS)
- **Quality**: Low, Medium, High, Ultra (affects bitrate)
- **Resolution**: Multiple 16:9, 9:16 (vertical), and 4:3 presets (720p to 4K)
- **Aspect Ratio**: 16:9, 9:16 (vertical - Reels/TikTok/Shorts), or 4:3

Zoom level semantics (v3.2.0): the Zoom slider value is the zoom of the 1920px reference frame;
the preview and all export resolutions show that same area.

Hardcoded timing values:
- Tile wait after zoom change: 150ms + up to 2000ms for tile loading
- WebCodecs frame timestamps: `frame * round(1e6 / targetFPS)` microseconds
- Keyframe interval (WebCodecs): every 2 seconds (`targetFPS * 2` frames)

### Color Scheme
Primary brand color `#fc4c02` used for:
- Header title, buttons, progress bars, version badge
- Changing requires find-replace across all CSS

## Critical Constraints

### CORS Requirements
- All external resources must support CORS for video export
- Tile servers must send proper CORS headers
- Canvas becomes "tainted" if non-CORS images drawn

### Browser Compatibility
**Desktop:**
- WebCodecs MP4 export: Chrome/Edge 94+, Firefox 130+, Safari 16.4+
- Older browsers fall back to MediaRecorder (WebM)

**Mobile:**
- **iOS 16.4+**: MP4 export works via WebCodecs
- **Older iOS**: export unavailable; alert offers screen-recording instructions
- **Android**: MP4 export works in modern Chrome

### Performance Considerations
- Large GPX files (>10,000 points) may slow animation
- WebCodecs path: no frame storage; memory use is flat regardless of duration/resolution
- Fallback path: frames stored as compressed blobs (~hundreds of KB each); much lighter than the
  old raw ImageData approach but still grows with duration × FPS
- Longer durations and higher resolutions increase export time
- Rendering overlay hides map during export

## Debugging

### Common Issues

**Video export fails / wrong format**:
- Check console for "Starting WebCodecs single-pass MP4 export" — if absent, the browser fell back
  to MediaRecorder (look for "Codec Support Detection" logs)
- `pickAvcEncoderConfig` returning null means WebCodecs/H.264 unavailable on that browser

**Video has wrong FPS or duration (fallback path only)**:
- Verify MediaRecorder only starts during Phase 2 (look for "🔴 Recording started")
- Check browser console for "Video Metadata Inspection" - should show correct FPS/duration
- A duration-mismatch warning is logged if the encoded duration is off by >10%

**Video export produces 1KB corrupted file**:
- Canvas is CORS-tainted from non-CORS images
- Check tile server supports `crossOrigin: 'anonymous'`
- Verify browser console for CORS errors

**Tracks or markers not showing in video**:
- Check console for "⚠️ Warning: Found X canvases but none had content"
- This triggers fallback to direct track rendering (`drawTracksDirect`)
- Verify tracks have valid lat/lon coordinates
- Check if `showAllTracks` toggle affects visibility

**Animation not smooth in browser**:
- Check `isPlayingRef` synced with `isPlaying` state
- Verify `requestAnimationFrame` loop not cancelled
- Test with single track to isolate performance

**Export too slow**:
- Each frame waits for tiles when zoom changes (150ms + up to 2000ms)
- Disable "Zoom to Track" to speed up export (no zoom changes = no tile waits)
- Consider lower FPS for faster exports

**Out of memory during export**:
- Should no longer occur on the WebCodecs path (no frame storage)
- On the fallback path, reduce resolution or duration (frames stored as compressed blobs)

## Deployment

**Primary**: GitHub Pages (https://dkaulukukui.github.io/Flyin_Amas_GPX_Visializer/)
- Auto-deploys on push to `master` branch
- Serves from repository root
- Takes 1-2 minutes to update

**Alternatives**: Netlify, Cloudflare Pages, Vercel (any static host)

No build process required - single HTML file is the entire app.

## Version History & Key Milestones

**Current Version: 3.12.0**

### Major Achievements
- ✅ **MP4 Export Everywhere (v2.4.0)**: WebCodecs single-pass export produces real MP4 on desktop and mobile (incl. iOS 16.4+/Android)
- ✅ **Out-of-Memory Fix (v2.4.0)**: No frame storage on the preferred path; compressed frames on fallback
- ✅ **Preview/Export Speed Match (v2.4.0)**: Single duration source drives both preview and export
- ✅ **Security Hardening (v2.4.0)**: Label XSS fix, GPX validation, file size cap
- ✅ **Solved FPS Problem (v2.1.x)**: Videos export at exactly the specified FPS and duration
- ✅ **Visual Completeness**: Position markers, full track preview, map tiles all included

### Technical Journey
- **v1.x**: Initial implementation, struggled with FPS accuracy
- **v1.9.x**: Multiple failed attempts using manual `requestFrame()` timing
- **v2.0.0**: Attempted real-time capture (failed due to tile loading speed)
- **v2.1.0**: Breakthrough with two-phase export system
- **v2.1.1**: Critical fix - start recording only during Phase 2
- **v2.1.4**: Added iOS detection to block export (later superseded)
- **v2.2.x**: Centered-track follow, configurable map center, label auto-naming
- **v2.3.0**: captureStream(0) + manual requestFrame timing; duration computed from playback speed
- **v2.4.0**: WebCodecs single-pass MP4 export; RAM fix; preview/export speed match; security hardening
- **v2.5.0**: 9:16 vertical export format (Reels/TikTok/Shorts)
- **v2.6.0**: Track comparison - common-point alignment with radius circle, race-from-point playback
- **v3.0.0**: Duration input replaces speed slider; example tracks; per-track trimming; label font/size; collapsible sidebar; export ETA + leave warning; FFmpeg.wasm removed
- **v3.1.0**: Interpolated track heads (smooth motion between GPS fixes); Track Smoothing option
- **v3.2.0**: WYSIWYG preview (reference-frame scaling, letterboxed preview, exports match preview at any resolution); Legend Size
- **v3.3.0**: "FlyinAmas" default title; lower-right watermark (text/size/opacity); larger default text sizes
- **v3.4.0**: Scoreboard & stats (timer, speed, distance, HR, stroke rate, gap) with a stats registry
- **v3.5.0**: Tabbed sidebar (Tracks/Camera/Overlays/Export), transport bar under the preview, remembered settings
- **v3.6.0**: Project files (tracks + settings), settings presets, ?project= / ?gpx= startup links
- **v3.7.0**: Map zoom controls set the camera zoom; playback performance work (persistent track layers)
- **v3.8.0**: Esri Clarity imagery by default (clearer water), Map Imagery choice
- **v3.9.0**: Tile prefetch along the follow camera's path (preview + export)
- **v3.9.1**: Track canvas always repaints in full at 1x (did not fix the iPhone artifact)
- **v3.10.0**: Track layers drawn with the SVG renderer (did not fix the iPhone artifact)
- **v3.10.1**: `?debug=` diagnostic switches used to bisect the iPhone artifact on the device (removed in v3.11.0)
- **v3.11.0**: Tile/track/label panes on their own compositing layers — fixed the iPhone shaded box but tore track lines
- **v3.11.1**: Only the tile pane on its own compositing layer; temporary `?debug=` switches (overlay, markers, canvas)
- **v3.12.0**: Canvas track renderer + tile-pane layer, confirmed clean on iPhone; switches removed

### Key Learning
The MediaRecorder API requires frames at **consistent time intervals** to produce correct FPS, which
forced the complex two-phase design. The WebCodecs API removes that constraint entirely: frames carry
explicit timestamps, so rendering can be as slow as needed while output timing stays exact — and
nothing has to be buffered in RAM. MediaRecorder remains only as a fallback for older browsers.
