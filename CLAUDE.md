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
   - Current version: 3.0.0 (as of last update)

### Key Technical Patterns

#### GPX File Processing
- Files uploaded via drag-and-drop or file input (50 MB size cap per file)
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
- **Satellite tiles**: ArcGIS World Imagery with `crossOrigin: 'anonymous'` for CORS
- **Backup tiles**: OpenStreetMap as fallback layer
- Initial map center configurable via `INITIAL_MAP_CENTER` (default: Oahu, Hawaii)
- Polylines drawn using Leaflet canvas renderer
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

**Single source of truth**: `getVisiblePoints(track, progressPercent)` (component scope) computes
visible points for all three modes and is used by the polyline render effect, the zoom-follow
effect, `renderExportFrame`, and `drawTracksDirect` — change animation behavior ONLY there.

**Animation Loop**:
- Uses `requestAnimationFrame` for smooth rendering
- **Preview pacing matches export exactly**: `progressIncrement = (deltaTime / (exportDuration * 1000)) * 100`
- `exportDuration` is a direct user input (v3.0.0): Duration number field, 5-300s, default 60s
- Controlled by `isPlayingRef` to avoid stale closure issues

**Track trimming (v3.0.0)**:
- `tracks` keeps the original uploaded points; per-track `trimStart`/`trimEnd` indices are set by
  the ✂ Trim sliders in the track list
- `effectiveTracks` (useMemo) applies the trim and recomputes startTime/endTime/duration —
  **all** animation, alignment, zoom, stats, and export logic consumes `effectiveTracks`, never
  `tracks` directly (UI lists and labels still use `tracks`)

**Example tracks (v3.0.0)**:
- `example/Koa_example.gpx` and `example/Nalu_example.gpx` are fetched on startup and loaded with
  `isExample: true`; `handleFileUpload` filters them out as soon as the user uploads their own files
- `.gitignore` excludes `*.gpx` but exempts `example/*.gpx`

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
- `labelFont`, `labelSize`: Track label styling (preview + export)
- `trimOpenId`: Track whose ✂ Trim sliders are expanded (trim values live on the track objects)
- `exportFPS`: Target frame rate (15-60 FPS, default 30 FPS)
- `exportQuality`: Video quality preset ('low', 'medium', 'high', 'ultra')
- `exportFormat`: Current export format ('MP4', 'WebM (VP9)', 'WebM (VP8)')
- `videoTitle`, `titlePosition`, `titleSize`, `titleFont`, `titleColor`: Video title customization

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
Edit tile layer URLs (search for "ArcGIS" or "openstreetmap"):
- Satellite layer: ArcGIS World Imagery
- Backup layer: OpenStreetMap
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

**Current Version: 3.0.0**

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

### Key Learning
The MediaRecorder API requires frames at **consistent time intervals** to produce correct FPS, which
forced the complex two-phase design. The WebCodecs API removes that constraint entirely: frames carry
explicit timestamps, so rendering can be as slow as needed while output timing stays exact — and
nothing has to be buffered in RAM. MediaRecorder remains only as a fallback for older browsers.
