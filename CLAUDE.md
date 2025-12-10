# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Flyin Amas GPX Track Animator is a **single-file web application** (`index.html`) that visualizes and animates GPS tracks from GPX files with video export capabilities. All processing happens client-side in the browser with no backend server.

**Live deployment**: GitHub Pages (https://dkaulukukui.github.io/Flyin_Amas_GPX_Visializer/)

## Architecture

### Single-File Application Structure

The entire application is contained in `index.html` with this organization:

1. **HTML Head** (lines 1-524)
   - External dependencies via CDN (React 18, Leaflet 1.9.4, Babel Standalone)
   - Embedded CSS styles for entire application
   - All styles use consistent color scheme: `#fc4c02` (primary orange), `#2d2d2d` (dark backgrounds)

2. **React Application** (lines ~528-2500+)
   - Single root component: `GPXTrackAnimator`
   - Uses React hooks exclusively (no class components)
   - JSX transpiled in-browser using Babel Standalone

3. **Version Management** (lines ~786-839)
   - `APP_VERSION` constant with comprehensive changelog comments
   - **CRITICAL**: Increment version number when making changes
   - Format: `major.minor.patch` (semantic versioning)
   - Current version: 2.1.3 (as of last update)

### Key Technical Patterns

#### GPX File Processing
- Files uploaded via drag-and-drop or file input
- XML parsing extracts `<trkpt>` elements with `lat`, `lon`, optional `time` attributes
- Each track assigned unique ID, random color, default label from filename
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

**Animation Loop**:
- Uses `requestAnimationFrame` for smooth rendering
- Delta-time based progress: `progressIncrement = (deltaTime / 100) * speed`
- Controlled by `isPlayingRef` to avoid stale closure issues

**Visual Elements**:
- Full track preview (dimmed, togglable): Shows complete path at 30% opacity
- Animated track: Current progress shown at full opacity
- Position markers: White-rimmed circles at current position
- Track labels: Customizable labels that follow track position

#### Video Export Architecture

**CRITICAL DESIGN**: Two-phase export system (v2.1.x)

**Why this approach**:
- Map tiles load asynchronously and may not be ready during real-time capture
- Tile loading delays (100-1500ms) prevent consistent frame capture
- MediaRecorder requires frames at precise intervals for correct FPS/duration
- Two-phase approach separates rendering from timing

**Phase 1: Pre-Rendering** (takes several minutes):
1. Create offscreen canvas with user-selected resolution (16:9 or 4:3)
2. Calculate total frames: `duration × FPS` (e.g., 60s × 30fps = 1800 frames)
3. Loop through all frames sequentially:
   - Set progress value (triggers React re-render)
   - Apply zoom-to-track if enabled
   - Wait for tiles to load using `waitForTilesToLoad()` (100-1500ms per zoom change)
   - Draw composite frame: tiles → tracks → markers → labels → legend
   - Store frame as `ImageData` in memory array
4. Progress bar shows pre-render progress (0-100%)
5. **MediaRecorder is NOT started yet**

**Phase 2: Playback** (real-time, exactly equals video duration):
1. Start MediaRecorder with `captureStream(targetFPS)`
2. Loop through pre-rendered frames at precise intervals:
   - Calculate target time: `startTime + (frameIndex × frameDuration)`
   - Draw pre-rendered `ImageData` to canvas
   - Wait until exact target time
   - Stream auto-captures at specified FPS
3. After all frames played back, stop MediaRecorder
4. Download video blob (WebM, MP4, or WebM VP8 depending on browser)

**Frame Capture Function** (`captureFrame`):
- Draws composite frame to offscreen canvas
- Layer order: map tiles → direct track rendering → labels → legend → video title
- Direct track rendering bypasses Leaflet for reliability
- Includes full track preview (dimmed) if `showAllTracks` enabled
- Draws position markers (white-rimmed circles) at current position

**Direct Track Rendering** (`drawTracksDirect`):
- Bypasses Leaflet canvas renderer for reliability (iOS/Safari fix)
- Converts lat/lon to pixel coordinates using map bounds
- Draws tracks, markers, and previews directly to canvas
- Ensures all visual elements appear in exported video

**Tile Loading Optimization** (`waitForTilesToLoad`):
- Smart detection: only waits when zoom changes
- Fast path if tiles already loaded (16ms check)
- Max wait time: 1500ms for new tiles after zoom
- No wait needed if zoom unchanged (just `requestAnimationFrame`)

#### State Management

All state managed through React useState/useRef hooks:

**Component State** (useState):
- `tracks`: Array of loaded GPX tracks
- `isPlaying`, `progress`, `speed`: Animation controls
- `isRecording`, `isRendering`, `renderProgress`: Export state
- `showAllTracks`, `showLabels`, `showLegend`: Display toggles
- `animationStyle`: 'simultaneous' or 'sequential'
- `zoomToTrack`, `zoomLevel`: Zoom controls
- `exportAspectRatio`, `exportResolution`: Export settings
- `exportDuration`: Video duration in seconds (5-300s, default 60s)
- `exportFPS`: Target frame rate (15-60 FPS, default 30 FPS)
- `exportQuality`: Video quality preset ('low', 'medium', 'high', 'ultra')
- `exportFormat`: Current export format ('MP4', 'WebM (VP9)', 'WebM (VP8)')
- `videoTitle`, `titlePosition`, `titleSize`, `titleFont`, `titleColor`: Video title customization

**Refs** (useRef):
- `mapInstanceRef`: Leaflet map instance
- `polylineRefs`: Array of Leaflet polyline objects
- `isPlayingRef`: Synced with isPlaying for animation loop
- `mediaRecorderRef`, `compositeCanvasRef`, `compositeStreamRef`: Export rendering
- `renderCancelledRef`: Cancel signal for export

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

### Making Changes

1. **Update version number** in `APP_VERSION` (around line 839)
2. **Add changelog entry** in comments above version (around line 787-794)
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
- Default speed range: `0.1` to `5.0`
- Speed slider step: `0.1`
- Zoom levels: `12` to `18`

### Video Export Settings
User-configurable via UI:
- **Duration**: 5-300 seconds (default: 60s)
- **FPS**: 15-60 FPS (default: 30 FPS)
- **Quality**: Low, Medium, High, Ultra (affects bitrate)
- **Resolution**: Multiple 16:9 and 4:3 presets (720p to 4K)
- **Aspect Ratio**: 16:9 or 4:3

Hardcoded timing values:
- Tile wait after zoom: 100ms + 1500ms max for tile loading
- Frame interval: `1000 / targetFPS` milliseconds

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
- Requires `MediaRecorder` API (Chrome, Firefox, Edge)
- Safari may have WebM codec issues
- Canvas `captureStream()` required for video export

### Performance Considerations
- Large GPX files (>10,000 points) may slow animation
- Video export is CPU and memory intensive:
  - Phase 1 pre-renders all frames (e.g., 1800 frames for 60s @ 30fps)
  - Each frame stored as ImageData in memory (can be several GB for 4K videos)
  - Phase 2 playback runs in real-time (60s video = 60s playback)
- Longer durations and higher resolutions increase export time significantly
- Rendering overlay hides map during Phase 1 to improve performance

## Debugging

### Common Issues

**Video has wrong FPS or duration**:
- Check console logs for "Phase 1 complete" and "Phase 2: Playing back"
- Verify MediaRecorder only starts during Phase 2 (look for "🔴 Recording started")
- Ensure Phase 2 playback completes (should take exactly the video duration)
- Check browser console for "Video Metadata Inspection" - should show correct FPS/duration

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

**Export Phase 1 too slow**:
- Each frame waits for tiles when zoom changes (100ms + 1500ms max)
- Disable "Zoom to Track" to speed up export (no zoom changes = no tile waits)
- Consider lower FPS or shorter duration for faster exports

**Out of memory during export**:
- Reduce resolution (4K uses ~4x memory vs 1080p)
- Reduce duration (60s = 1800 frames @ 30fps, 300s = 9000 frames)
- Each frame stored as ImageData in memory during Phase 1
- Close other tabs/applications to free up RAM

## Deployment

**Primary**: GitHub Pages (https://dkaulukukui.github.io/Flyin_Amas_GPX_Visializer/)
- Auto-deploys on push to `master` branch
- Serves from repository root
- Takes 1-2 minutes to update

**Alternatives**: Netlify, Cloudflare Pages, Vercel (any static host)

No build process required - single HTML file is the entire app.

## Version History & Key Milestones

**Current Version: 2.1.3**

### Major Achievements (v2.1.x)
- ✅ **Solved FPS Problem**: Videos now export at exactly the specified FPS and duration
- ✅ **Two-Phase Export**: Pre-rendering phase + playback phase ensures quality and timing
- ✅ **Visual Completeness**: Position markers, full track preview, map tiles all included
- ✅ **Browser Playback Match**: Export output matches browser preview exactly

### Technical Journey
- **v1.x**: Initial implementation, struggled with FPS accuracy
- **v1.9.x**: Multiple failed attempts using manual `requestFrame()` timing
- **v2.0.0**: Attempted real-time capture (failed due to tile loading speed)
- **v2.1.0**: Breakthrough with two-phase export system
- **v2.1.1**: Critical fix - start recording only during Phase 2
- **v2.1.2**: Added position markers and full track preview to exports
- **v2.1.3**: Fixed `showAllTracks` toggle to work in browser playback

### Key Learning
The MediaRecorder API requires frames at **consistent time intervals** to produce correct FPS. The two-phase approach decouples slow tile loading (Phase 1) from precise timing requirements (Phase 2), solving the fundamental problem that plagued earlier versions.
