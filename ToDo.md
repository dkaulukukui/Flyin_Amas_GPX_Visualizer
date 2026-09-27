To Do List:

## Done in v3.13.0

- Leaderboard overlay: place a finish point; a separate panel ranks tracks by distance to it and re-orders
  live; tracks reaching the finish radius lock in by finish time; own position/size/font; optional finish flag.

## Done in v3.12.0

- iPhone Safari rendering confirmed clean on the phone: tile pane on its own compositing layer (no shaded
  box) + track layers on Leaflet's canvas renderer (SVG lines tore ahead of the boats). Found by on-device
  switches in v3.10.1/v3.11.1; switches removed.

## Done in v3.11.0

- **Fixed** the iPhone Safari shaded box (darker box over the travelled tracks' bounding box, rest washed
  out, edge following the boat). Cause: with Leaflet 3D disabled (v3.2), tiles and track layers were
  painted as one layer and Safari re-rendered the tiles under the changing tracks differently. Fix: tile,
  track, and label panes get their own compositing layers (`will-change: transform`). Found by bisecting
  on the phone with the v3.10.1 `?debug=` switches (`layers` removed the box); switches now removed.
  Earlier attempts that didn't address this symptom: imagery (v3.8.0), tile prefetch (v3.9.0, still
  useful), full canvas repaint (v3.9.1), SVG track layers (v3.10.0, kept - faster).

## Done in v3.10.0

- The "lighter band" on mobile: a phone screenshot showed a darker box exactly covering the bounding box
  of the travelled tracks, with the rest of the map washed out (edges screen-aligned; moves with the boat,
  then gets left behind). It is iPhone Safari's canvas compositing of the semi-transparent track strokes —
  not tiles or imagery (v3.8.0/v3.9.0) and not Leaflet's partial repaint (v3.9.1 changed nothing). Fixed by
  drawing track layers with Leaflet's SVG renderer (also faster). `?renderer=canvas` switches back for
  comparison. **Re-test on the phone.**

## Done in v3.9.1

- Attempted fix (did not help): full-canvas repaints at 1x for the track canvas.

## Done in v3.9.0

- Moving band of lighter tiles ahead of the tracks on mobile — this was the leading edge of the view still
  downloading (Leaflet only requests tiles once they're on screen; the v3.8.0 "imagery" explanation was
  wrong about the band). Fixed with tile prefetch along the follow camera's upcoming path (next 4 s of
  video), also used by exports. Slow-connection emulation: view without tiles 45% -> 2% (Clarity),
  4% -> 0% (standard); exports ~20% faster.

## Done in v3.8.0

- Esri Clarity imagery as the default (cleaner, more consistent water at high zoom) with a Map Imagery
  choice in the Camera tab. Note: Clarity tiles load through two redirects, so first loads are slower
  (hidden by the v3.9.0 prefetch).

## Done in v3.7.0

- ~~make the zoom controls on the map adjust the camera zoom setting as well~~ — Done. +/-, mouse wheel,
  and pinch zoom set the camera Zoom Level while the follow camera is running (range now 10-18).
- ~~investigate map going white / washed out when zoomed in during playback (mobile)~~ — Investigated:
  not missing imagery (Esri has real tiles at zoom 18 along the course) and not reproducible in desktop
  Chrome's phone emulation. Found and fixed heavy per-frame work that starves slow phones (all track
  layers rebuilt every frame, totals recomputed every frame, full-size scoreboard canvas) and made
  not-yet-loaded tiles dark instead of light grey. **Needs a re-test on a real phone** — if it still
  happens, note the phone/browser, zoom level, and whether pausing lets the map fill in.

## Done in v3.6.0

- Load tracks when the page opens and save/load export settings, combined: project files (tracks + all
  settings, reproduce a video exactly), settings-only presets, and `?project=` / `?gpx=` startup links.

## Done in v3.5.0

- ~~consider a different settings layout for better usability~~ — Done. Sidebar split into Tracks / Camera /
  Overlays / Export tabs with the Export Video button pinned at the bottom; playback controls (play, reset,
  scrubber, time, duration) moved to a transport bar under the map preview; settings are remembered in the
  browser with a "Reset all settings to defaults" link.

## Done in v3.4.0

- ~~show statistics on screen (Timer, Speed, total distance, distance between two tracks) with font/size/
  location customization, uncluttered placement, a framework for future stats, and an adjustable start
  point~~ — Done. The legend is now a scoreboard (Scoreboard & Stats section): race timer header, one row
  per track with Speed / Distance / Heart Rate / Stroke Rate columns, and a Gap row between two chosen
  tracks. Imperial or Metric. "Set start here" on the playback position sets where timer and distance
  start. New stats are one entry in `STAT_DEFINITIONS` (HR and cadence already read from GPX extensions).
  Future ideas: wave counter, flyin ama counter.

## Done in v3.2.0

- ~~allow user in increase font size of legend~~ — Done. Legend Size slider (10-40px) under Show legend.
- ~~browser playback zoom level doesnt match exported video~~ — Fixed. Every export resolution now
  shows exactly the area and proportions of the preview (higher resolutions are just sharper).
- ~~should we adjust browser playback ascpect ratio to match output?~~ — Yes, done. The preview is
  letterboxed to the export aspect ratio ("Show video frame in preview", on by default) and renders
  exactly like the exported video.

## Done in v3.1.0

- ~~add smoothing or interpolation to tracks so that tracks arent jumpy~~ — Done. Track heads are
  interpolated between GPS fixes (no more per-frame stepping), plus a Track Smoothing option
  (Off/Light/Medium/Strong, default Light) under Display Options to remove GPS jitter.

## Done in v3.0.0

- ~~adjust the speed to be a video duration input instead of a slider, default to 60s~~ — Done.
  Duration number input (5-300s, default 60s) drives both the preview and the exported video.
- ~~preload the example files in example/, automatically clear the files when user uploads thier own~~ — Done.
  `examples/Kai.gpx` and `examples/Kimo.gpx` load on startup (marked "example" in the
  track list) and are removed automatically on the first user upload.
- ~~allow user to change the font and text size of the track labels~~ — Done. Label Font and
  Label Size controls under "Show track labels"; applies to the preview and exported videos.
- ~~povide user with estimated time to complete when exporting. warn them not to close tab or exit
  until finished~~ — Done. Export overlay shows estimated time remaining and a keep-tab-open warning,
  and the browser asks for confirmation if the user tries to leave mid-export.
- ~~make all the settings collapsable~~ — Done. Sidebar sections (Playback, Compare Tracks,
  Display Options, Export Settings, Video Title) are collapsible.
- ~~provide the ability to trim tracks~~ — Done. ✂ Trim button per track with start/end point
  sliders; non-destructive and reversible (Reset). Animation, alignment, stats, and exports all
  use the trimmed track.
- ~~Consider removing the FFmpeg.wasm dependency (~30 MB download)~~ — Removed. It could never work
  on static hosting (requires SharedArrayBuffer / cross-origin isolation headers) and the WebCodecs
  export path doesn't need it.

## Done previously

- (v2.4.0) MP4 export via WebCodecs (incl. iOS 16.4+/Android), out-of-memory fix, preview/export
  speed match, input sanitation/XSS hardening.
- (v2.5.0) 9:16 (vertical) aspect ratio for output.
- (v2.6.0) Compare tracks from different days: common point + radius circle, closest/first/last
  pass matching, race-from-point playback.
- Test v2.4.0 export on real devices: iPhone (Safari 16.4+), Android Chrome, desktop Safari, Firefox.

## Remaining

(nothing queued — add new ideas here)
