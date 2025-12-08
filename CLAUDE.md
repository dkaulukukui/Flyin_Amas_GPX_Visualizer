# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a single-page GPX Track Animator application for visualizing and animating GPS tracks from Strava or other GPS devices. The entire application is self-contained in `gpx-track-animator.html`.

## Architecture

### Single-File Application Structure

The project uses a monolithic HTML file containing:
- **Styles**: Embedded CSS with a dark theme and orange accent color (#fc4c02)
- **React Application**: Written in JSX via Babel standalone (loaded from CDN)
- **External Dependencies**: All loaded via CDN (no build process)
  - React 18 (production build)
  - Leaflet 1.9.4 (mapping library)
  - Babel Standalone (JSX transpilation)

### Core Components

**Main React Component**: `GPXTrackAnimator` (lines 330-767)

Key state management:
- `tracks`: Array of parsed GPX files with {id, name, points, color}
- `progress`: Animation progress (0-100%)
- `isPlaying`: Animation playback state
- `speed`: Animation speed multiplier (0.5-5x)
- `animationStyle`: 'sequential' (one after another) vs 'simultaneous' (all at once)
- `isRecording`: Video recording state

### GPX Parsing

`parseGPX()` function (lines 368-387):
- Parses GPX XML using DOMParser
- Extracts `<trkpt>` elements with lat/lon coordinates and timestamps
- Assigns random HSL colors to each track
- Returns track object with unique ID

### Map Rendering

- Uses Leaflet with satellite imagery from ArcGIS World_Imagery
- Initialized on component mount (lines 348-359)
- Map instance stored in `mapInstanceRef`
- Polylines re-rendered on every progress/track change (lines 456-514)
- Shows gray preview lines + colored animated lines + circular markers

### Animation System

Animation loop (lines 431-454):
- Uses `requestAnimationFrame` for smooth animation
- Progress increments by `0.5 * speed` per frame
- Stops automatically at 100%
- **Sequential mode**: Calculates cumulative points across all tracks
- **Simultaneous mode**: All tracks animate at same percentage

### Video Recording

Recording implementation (lines 530-585):
- Captures Leaflet canvas element using `captureStream(30)` API
- Uses MediaRecorder with VP9 codec at 5Mbps
- Auto-starts animation and auto-stops at completion
- Downloads as .webm file

## Development

### Running the Application

Simply open `gpx-track-animator.html` in a modern web browser. No build step required.

### Testing Changes

Refresh the browser after making changes. For live development:
```bash
python -m http.server 8000
# or
npx http-server
```

### Browser Requirements

- Must support ES6+, React 18, MediaRecorder API, and canvas.captureStream()
- Chrome/Edge recommended for best recording compatibility
- Safari may have limited MediaRecorder support

## Key Constraints

- **No package.json or build system**: All dependencies via CDN
- **React written in JSX**: Transpiled in browser by Babel Standalone (performance cost)
- **Single file architecture**: All code in one HTML file for portability
- **Recording canvas-based**: Only captures the map canvas, not the full UI
