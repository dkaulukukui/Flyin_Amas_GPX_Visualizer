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

2. **React Application** (lines 528-1680)
   - Single root component: `GPXTrackAnimator`
   - Uses React hooks exclusively (no class components)
   - JSX transpiled in-browser using Babel Standalone

3. **Version Management** (lines 531-536)
   - `APP_VERSION` constant with changelog comments
   - **CRITICAL**: Increment version number when making changes
   - Format: `major.minor.patch` (semantic versioning)

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
- Implementation: `index.html:789-859`

**Sequential Mode**:
- Plays tracks one after another in upload order
- Each track uses full 0-100% progress range
- Implementation: `index.html:861-876`

**Animation Loop**:
- Uses `requestAnimationFrame` for smooth rendering
- Delta-time based progress: `progressIncrement = (deltaTime / 100) * speed`
- Controlled by `isPlayingRef` to avoid stale closure issues
- Implementation: `index.html:695-764`

#### Video Export Architecture

**CRITICAL DESIGN**: Frame-by-frame rendering system (not real-time capture)

**Why this approach**:
- Map tiles load asynchronously and may not be ready during live playback
- Real-time capture creates choppy videos with missing tiles
- Frame-by-frame ensures every frame has fully loaded tiles before capture

**Rendering Process** (`index.html:1147-1285`):
1. Create offscreen canvas with user-selected resolution (16:9 or 4:3)
2. Setup MediaRecorder with 30 FPS stream from canvas
3. Loop through 300 frames (10-second video):
   - Set progress value (triggers React re-render)
   - Wait for tiles to load using `waitForTilesToLoad()`
   - Draw composite frame: tiles → canvases → labels → legend
   - Capture to MediaRecorder stream
   - Small delay for stream capture
4. Stop recorder, download WebM blob

**Tile Loading Optimization** (`index.html:999-1025`):
- First checks if all tiles already loaded (fast path: 16ms)
- Only waits (max 500ms) if tiles still loading
- Critical for smooth 30 FPS frame rate

**Composite Frame Drawing** (`index.html:1027-1076`):
- Canvas scaled to match export resolution
- Draws in order: map tiles → track canvases → labels → legend
- Uses canvas transforms for scaling and centering
- Try-catch around tile drawing for CORS failures

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

1. **Update version number** in `APP_VERSION` (line 536)
2. **Add changelog entry** in comments above version
3. Test locally with `python -m http.server`
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
Edit tile layer URLs at `index.html:555-576`:
- Satellite layer: ArcGIS World Imagery
- Backup layer: OpenStreetMap
- **Must include** `crossOrigin: 'anonymous'` for video export

### Adjusting Animation
- Default speed range: `0.1` to `5.0` (lines 1411-1413)
- Speed slider step: `0.1` (line 1412)
- Zoom levels: `12` to `18` (lines 1473-1475)

### Video Export Settings
- Target FPS: `30` (line 1189)
- Total frames: `300` (10 seconds, line 1256)
- Tile wait timeout: `500ms` (line 1000)
- Resolution presets: lines 1514-1524

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
- Video export is CPU-intensive (renders 300+ frames)
- Rendering overlay hides map to improve performance during export

## Debugging

### Common Issues

**Video export produces 1KB corrupted file**:
- Canvas is CORS-tainted from non-CORS images
- Check tile server supports `crossOrigin: 'anonymous'`
- Verify browser console for CORS errors

**Animation not smooth**:
- Check `isPlayingRef` synced with `isPlaying` state
- Verify `requestAnimationFrame` loop not cancelled
- Test with single track to isolate performance

**Tiles not loading in video**:
- Increase `maxWaitTime` in `waitForTilesToLoad()` (line 1000)
- Check network tab for tile 404s or timeouts
- Verify tile URLs are correct for current map view

**Export works once then fails**:
- Ensure all refs/state properly reset in `stopRecording()` (lines 1295-1310)
- Check MediaRecorder stream tracks are stopped
- Verify canvas cleanup in `recorder.onstop` handler

## Deployment

**Primary**: GitHub Pages (https://dkaulukukui.github.io/Flyin_Amas_GPX_Visializer/)
- Auto-deploys on push to `master` branch
- Serves from repository root
- Takes 1-2 minutes to update

**Alternatives**: Netlify, Cloudflare Pages, Vercel (any static host)

No build process required - single HTML file is the entire app.
