# Flyin Amas GPX Track Animator

A beautiful, interactive web application for visualizing and animating GPS tracks from GPX files. Perfect for showcasing Strava activities, outdoor adventures, or any GPS-recorded journey.

![GPX Track Animator](https://img.shields.io/badge/Status-Active-success)
![License](https://img.shields.io/badge/License-MIT-blue)

## Features

- **📁 Upload Multiple GPX Files** - Drag and drop or browse to add tracks
- **🗺️ Satellite Map View** - High-quality satellite imagery background
- **🎬 Simultaneous & Sequential Animation** - Play tracks together or one after another
- **⏱️ Time-Synchronized Playback** - Tracks play according to actual timestamps
- **🎨 Customizable Track Colors** - Choose colors for each track
- **🏷️ Track Labels** - Add and edit labels for your tracks
- **📊 Legend Display** - Toggle legend showing all loaded tracks
- **🎥 Video Export** - Export animations as WebM video files
- **⚡ Speed Control** - Adjust playback speed from 0.1x to 5x
- **🔍 Zoom to Track** - Follow the action with adjustable zoom level
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

Click "Export Video" to record your animation. The video will:
- Capture the animated tracks and markers on a dark background
- Save as WebM format (widely supported)
- Can be converted to MP4 using VLC or HandBrake if needed

**Note:** Due to browser security (CORS), the background map tiles are not included in exports. For videos with the full map background, use screen recording tools like:
- **Chrome/Edge**: Built-in screen recorder (Extensions → Screen Recorder)
- **OBS Studio**: Free, powerful screen recording
- **macOS**: QuickTime Player (File → New Screen Recording)
- **Windows**: Xbox Game Bar (Win + G)

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

Simply open `index.html` in a web browser, or run a local server:

```bash
# Python
python -m http.server 8000

# Node.js
npx http-server

# Then open http://localhost:8000
```

## Technology Stack

- **React 18** - UI framework (loaded via CDN)
- **Leaflet 1.9.4** - Interactive maps
- **ArcGIS Satellite Tiles** - High-quality imagery
- **MediaRecorder API** - Video export
- **Pure JavaScript** - No build tools required

## File Structure

```
.
├── index.html          # Main application (single-file app)
├── README.md           # This file
└── CLAUDE.md           # Developer documentation
```

## Browser Support

- **Chrome/Edge** (recommended) - Full feature support
- **Firefox** - Full feature support
- **Safari** - Full feature support (WebM may require conversion)

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

## Support

For issues or questions, please open an issue on GitHub.
