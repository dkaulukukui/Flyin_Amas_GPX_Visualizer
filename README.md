# Flyin Amas GPX Track Animator

A beautiful, interactive web application for visualizing and animating GPS tracks from GPX files. Perfect for showcasing Strava activities, outdoor adventures, or any GPS-recorded journey.

![GPX Track Animator](https://img.shields.io/badge/Status-Active-success)
![Version](https://img.shields.io/badge/Version-3.0.0-blue)
![License](https://img.shields.io/badge/License-MIT-blue)

> **⚠️ Disclaimer:** This code was generated with AI assistance. While thoroughly tested, users should review and verify the code before use in production environments.

## What's New in v3.0.0

✨ **Major UI/UX Update**

- ⏱️ **Duration Input**: Set the video length directly (5-300s, default 60s) — replaces the speed slider; preview and export always match
- 🌺 **Example Tracks**: Two sample paddling tracks load automatically so you can try everything instantly; they clear as soon as you upload your own files
- ✂️ **Track Trimming**: Per-track start/end sliders to cut out dock time, drives to the start line, or GPS noise — fully reversible
- 🔤 **Label Styling**: Choose the font and size of track labels (in the preview and in exported videos)
- ⏳ **Export ETA**: The export overlay shows estimated time remaining and warns you not to close the tab (with a browser confirmation if you try)
- 📂 **Collapsible Settings**: Sidebar sections fold away so you can focus on what you're adjusting
- 🪶 **Lighter App**: Removed the unused 30 MB FFmpeg.wasm dependency

## Previous Update: v2.6.0

🏁 **Track Comparison / Race Mode** - Compare tracks recorded on different days!

- 📍 **Common Point Alignment**: Click anywhere on the map to set a shared start point, with an adjustable radius circle
- 🎯 **Flexible Matching**: Align each track by its closest point, first pass, or last pass through the circle
- ⏱️ **True Race Replay**: Every track replays at its actual recorded pace from its crossing — see who was really faster
- ⚠️ **Smart Feedback**: Live per-track match distances and warnings when a track has no point inside the circle
- 🎥 **Works in Exports**: Aligned races export to video exactly as previewed

## Previous Update: v2.5.0

📱 **Vertical Video Export** - New 9:16 aspect ratio for Instagram Reels, TikTok, and YouTube Shorts (720x1280, 1080x1920, 2160x3840)

## Previous Update: v2.4.0

🎬 **MP4 Export Everywhere + Major Stability Overhaul**

- 📹 **True MP4 (H.264) Export**: Videos now export as MP4 on Chrome, Edge, Firefox 130+, and Safari 16.4+ — including **iPhone, iPad, and Android**
- 🧠 **No More Out-of-Memory Crashes**: New WebCodecs single-pass export encodes each frame straight into the video — no frames are stored in RAM
- ⏱️ **Exact FPS & Duration**: Frames carry explicit timestamps, so the video is always exactly the configured length
- 🏃 **Preview Matches Export**: The in-browser preview now plays at exactly the same speed as the exported video
- 🔒 **Security Hardening**: Track labels are sanitized (XSS fix), GPX files are validated, and uploads are capped at 50 MB
- 🛟 **Smart Fallback**: Older browsers fall back to the previous MediaRecorder export, now using ~30x less memory

## Previous Updates (v2.1.x)

🎉 **Major Video Export Overhaul** - Videos now export at **exactly** the specified FPS and duration!

- ✨ **Two-Phase Export System**: Pre-renders frames, then plays back at precise intervals
- ✅ **Accurate FPS**: Videos now play at exactly 30 FPS (or your chosen frame rate)
- ✅ **Accurate Duration**: 60-second setting = exactly 60 seconds of video
- 🎨 **Full Track Preview**: Toggle dimmed preview of complete track path
- 📍 **Position Markers**: White-rimmed circles show current position
- 🗺️ **Map Tiles Included**: Satellite imagery now appears in exported videos
- ⚙️ **Customizable Settings**: Control duration (5-300s), FPS (15-60), quality, and resolution
- 📱 **iOS Detection**: Graceful handling with screen recording instructions

## Features

- **📁 Upload Multiple GPX Files** - Drag and drop or browse to add tracks
- **🗺️ Satellite Map View** - High-quality satellite imagery background with map tiles included in exports
- **🎬 Simultaneous & Sequential Animation** - Play tracks together or one after another
- **🏁 Track Comparison (Race Mode)** - Align tracks from different days at a common point and race them at their real recorded paces
- **⏱️ Time-Synchronized Playback** - Tracks play according to actual timestamps
- **🎨 Customizable Track Colors** - Choose colors for each track
- **🏷️ Track Labels** - Add and edit labels for your tracks
- **👁️ Full Track Preview** - Toggle dimmed preview showing complete track path
- **📍 Position Markers** - White-rimmed circles showing current position on each track
- **📊 Legend Display** - Toggle legend showing all loaded tracks
- **🎥 Advanced Video Export** - Export high-quality videos with precise control:
  - **Duration Control**: 5-300 seconds (default: 60s)
  - **Frame Rate Control**: 15-60 FPS (default: 30 FPS)
  - **Quality Presets**: Low, Medium, High, Ultra
  - **Multiple Resolutions**: 720p to 4K in 16:9, 9:16 (vertical), or 4:3
  - **Custom Video Title**: Add text overlay with positioning and styling options
  - **Multiple Formats**: MP4, WebM (VP9), or WebM (VP8) depending on browser
- **⏱️ Duration Control** - Set the animation/video length directly (5-300 seconds)
- **✂️ Track Trimming** - Cut unwanted points from the start or end of any track (reversible)
- **🔤 Label Styling** - Pick the font and size of on-map track labels
- **🌺 Example Tracks** - Bundled sample tracks load on startup and clear when you upload your own
- **🔍 Zoom to Track** - Follow the action with adjustable zoom level (12-18)
- **📈 Statistics** - View total distance, points, and time ranges

## Live Demo

Visit: [https://dkaulukukui.github.io/Flyin_Amas_GPX_Visualizer/](https://dkaulukukui.github.io/Flyin_Amas_GPX_Visualizer/)

## Usage

### Basic Usage

1. **Open the application** in your web browser
2. **Upload GPX files** by dragging and dropping or clicking the upload zone
3. **Customize tracks** - edit labels, change colors
4. **Configure playback** - choose animation style, speed, zoom options
5. **Play the animation** - watch your tracks come to life
6. **Export video** (optional) - save your animation as a video file

### Animation Modes

- **Simultaneous**: All tracks animate together based on their actual timestamps
- **Sequential**: Tracks play one after another in the order they were uploaded

### Video Export

The application uses a sophisticated two-phase export system to ensure perfect video quality:

**How It Works:**
- **Modern browsers (preferred path)**: Each frame is rendered with fully loaded map tiles, then encoded straight into an in-memory MP4 using the WebCodecs API. Frames carry explicit timestamps, so the FPS and duration are exact, and no frames need to be stored in RAM.
- **Older browsers (fallback)**: A two-phase MediaRecorder export — frames are pre-rendered and stored compressed, then played back at precise intervals while recording (WebM output).

**Export Settings:**
- **Duration**: Set video length from 5 to 300 seconds (default: 60s)
- **Frame Rate**: Choose FPS from 15 to 60 (default: 30 FPS for smooth motion)
- **Quality**: Select from Low, Medium, High, or Ultra presets
- **Resolution**: Multiple options from 720p to 4K in 16:9, 9:16 (vertical), or 4:3 aspect ratios
- **Video Title**: Add custom text overlay with positioning and styling
- **Format**: MP4 (H.264) on modern browsers; WebM (VP9/VP8) fallback on older browsers

**What's Included in Exported Videos:**
- ✅ Satellite map background (full tiles)
- ✅ Animated track paths with full preview (if enabled)
- ✅ Position markers (white-rimmed circles)
- ✅ Track labels
- ✅ Legend (if enabled)
- ✅ Custom video title (if configured)

**Performance Tips:**
- Longer durations and higher resolutions take more time
- Disable "Zoom to Track" to speed up rendering (no per-frame tile loading)
- For quick tests, use shorter durations (10-20s) and lower resolutions (720p)

**iOS/Mobile:**
- ✅ **iOS 16.4+ (iPhone, iPad)**: Video export works and produces MP4 (via WebCodecs)
- ✅ **Android**: Video export works in modern browsers (Chrome recommended), producing MP4
- ⚠️ **Older iOS versions**: Use the built-in Screen Recording feature instead:
  1. Open Control Center (swipe down from top-right on newer devices, or up from bottom on older ones)
  2. Tap the Screen Recording button (circle icon)
  3. Start your animation playback in the browser
  4. Recording automatically saves to your Photos app

## Deployment

### GitHub Pages (Recommended - FREE)

1. **Fork or clone this repository**
2. **Enable GitHub Pages**:
   - Go to repository Settings
   - Navigate to Pages
   - Select "main" branch as source
   - Click Save
3. **Access your site** at `https://yourusername.github.io/Flyin_Amas_GPX_Visializer/`

### Other Static Hosts (Also FREE)

- **Netlify**: Drag and drop the repository folder
- **Cloudflare Pages**: Connect your GitHub repo
- **Vercel**: Import from GitHub

### Local Development

**Important:** You must run a local server (not open `index.html` directly) for map tiles to work correctly due to CORS requirements.

```bash
# Python
python -m http.server 8000

# Node.js
npx http-server

# Then open http://localhost:8000
```

**Note:** Opening `index.html` directly as `file://` will block map tiles due to CORS security restrictions.

## Technology Stack

- **React 18** - UI framework (loaded via CDN)
- **Leaflet 1.9.4** - Interactive maps with canvas rendering
- **ArcGIS Satellite Tiles** - High-quality imagery with CORS support
- **WebCodecs API + mp4-muxer** - Hardware-accelerated H.264 encoding into MP4 (preferred export path)
- **MediaRecorder API** - Browser-native video encoding (fallback export path, WebM)
- **Canvas API** - Frame-by-frame video rendering with precise timing
- **Pure JavaScript** - No build tools or compilation required

## File Structure

```
.
├── index.html          # Main application (single-file app)
├── README.md           # This file
└── CLAUDE.md           # Developer documentation
```

## Browser Support

### Desktop
- **Chrome/Edge** (recommended) - Full feature support, MP4 video export
- **Firefox** - Full feature support; MP4 export on Firefox 130+, WebM on older versions
- **Safari 16.4+ (macOS)** - Full feature support, MP4 video export

### Mobile
- **iOS (iPhone/iPad)** - Animation and viewing work perfectly; **MP4 video export works on iOS 16.4+**. Older iOS versions can use iOS Screen Recording instead.
- **Android** - Animation and MP4 video export work in modern browsers (Chrome recommended)

## Privacy

All processing happens **client-side** in your browser:
- GPX files never leave your device
- No data is sent to any server
- No tracking or analytics

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

MIT License - feel free to use and modify for your own projects.

## Credits

Built with ❤️ for the GPS tracking community.

**Development:** This project was developed with AI assistance using Claude Code. The codebase is AI-generated and has been tested for functionality, but users are encouraged to review the code before deployment.

## Support

For issues or questions, please open an issue on GitHub.
