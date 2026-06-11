To Do List:

## Done in v2.4.0

- ~~ios crashes during export~~ — Fixed. Export no longer stores raw frames in RAM (the crash cause).
  iOS 16.4+ now exports MP4 natively via WebCodecs; older iOS gets screen-recording instructions.
- ~~Input sanitation and general security concern cleanup~~ — Fixed. Track labels are HTML-escaped
  (XSS fix), GPX files are validated (malformed XML rejected, invalid coordinates/timestamps skipped),
  and uploads are capped at 50 MB.
- ~~need a way to prevent site from over running the ram of the host machine and crashing~~ — Fixed.
  Preferred export path (WebCodecs) encodes frames straight into the MP4 with no frame storage at all.
  The MediaRecorder fallback now stores compressed WebP frames (~30x less RAM than before).
- ~~speed playback between the preview and exported videos isnt the same~~ — Fixed. Preview animation
  now runs for exactly the computed export duration, so preview and video pacing match.
- ~~figure out how to have IOS and andriod users export to video (mp4)~~ — Done. WebCodecs single-pass
  export produces real MP4 (H.264) on Chrome, Edge, Firefox 130+, Safari 16.4+ (including iPhone/iPad),
  and modern Android browsers.

## Remaining

- ~~Add feature for comparing two GPX tracks from different days~~ — Done in v2.6.0. "🏁 Compare Tracks"
  sidebar section: pick a common point on the map, adjust the radius circle, match by closest/first/last
  pass through the circle, then all tracks race from that point at their real recorded paces.
- (complete) Test v2.4.0 export on real devices: iPhone (Safari 16.4+), Android Chrome, desktop Safari, Firefox.
- Consider removing the FFmpeg.wasm dependency (~30 MB download) — it is only used by the legacy
  MediaRecorder fallback path for FPS correction, which most browsers no longer need.

- (complete v2.5.0) Add 9:16 (vertical) aspect ratio for output.

- adjust the speed to be a video duration input instead of a slider, default to 60s

- preload the example files in example/, automatically clear the files when user uploads thier own

- allow user to change the font and text size of the track labels

- povide user with estimated time to complete when exporting. warn them not to close tab or exit until finished

- make all the settings collapsable

- provide the ability to trim tracks