To Do List:

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
