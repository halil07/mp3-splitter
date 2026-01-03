# MP3 Splitter

A web-based MP3 splitter application built with Vue 3, TypeScript, and FFmpeg WASM. Upload an MP3 file, add multiple split points, and split it into as many segments as you need.

## Features

- **Drag & Drop Upload**: Easily upload MP3 files by dragging and dropping or clicking to browse
- **Audio Preview**: Play your uploaded MP3 directly in the browser
- **Multiple Split Points**: Add as many split points as you need using the "+" button
- **Draggable Markers**: Drag split markers along the timeline to adjust positions
- **Full-Width Slider**: Navigate through the entire audio file with a full-width slider
- **Segment Preview**: Preview each segment before splitting with play buttons
- **Visual Feedback**: See time ranges and durations for all segments
- **Real-time Progress**: Track the splitting process with status messages and progress bar
- **Play Resulting MP3s**: Play each generated segment directly from the results
- **Download Options**: Download individual segments or all at once
- **Client-side Processing**: All processing happens in your browser using FFmpeg WASM - no server required

## Technology Stack

- **Vue 3** - Progressive JavaScript framework with Composition API
- **TypeScript** - Type-safe JavaScript
- **Vite** - Fast build tool and dev server
- **FFmpeg WASM** - WebAssembly version of FFmpeg for audio processing
- **@ffmpeg/ffmpeg** - FFmpeg WASM core library
- **@ffmpeg/util** - Utility functions for FFmpeg WASM

## How It Works

1. **Upload**: Upload an MP3 file (drag & drop or file picker)
2. **Preview**: Listen to your audio using the built-in audio player
3. **Add Split Points**: 
   - Use the slider to navigate to where you want to split
   - Click "Add Split Point" to add a marker at that position
   - Repeat to add as many split points as needed
4. **Adjust Markers**: Drag markers along the timeline to fine-tune positions
5. **Remove Markers**: Click the "X" on any marker to remove it
6. **Preview Segments**: Click play buttons to preview each segment before splitting
7. **Split**: Click "Split MP3 into X Parts" to create all segments
8. **Play & Download**: Play each resulting MP3 or download them individually or all at once

The application uses FFmpeg commands for each segment:
```bash
ffmpeg -i input.mp3 -ss {startTime} -t {duration} -c copy part{N}.mp3
```

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur) + [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin).

## Type Support for `.vue` Imports in TS

TypeScript cannot handle type information for `.vue` imports by default, so we replace the `tsc` CLI with `vue-tsc` for type checking. In editors, we need [TypeScript Vue Plugin (Volar)](https://marketplace.visualstudio.com/items?itemName=Vue.vscode-typescript-vue-plugin) to make the TypeScript language service aware of `.vue` types.

If the standalone TypeScript plugin doesn't feel fast enough to you, Volar has also implemented a [Take Over Mode](https://github.com/johnsoncodehk/volar/discussions/471#discussioncomment-1361669) that is more performant. You can enable it by the following steps:

1. Disable the built-in TypeScript Extension
    1) Run `Extensions: Show Built-in Extensions` from VSCode's command palette
    2) Find `TypeScript and JavaScript Language Features`, right click and select `Disable (Workspace)`
2. Reload the VSCode window by running `Developer: Reload Window` from the command palette.

## Customize configuration

See [Vite Configuration Reference](https://vitejs.dev/config/).

## Project Setup

```sh
npm install
```

### Compile and Hot-Reload for Development

```sh
npm run dev
```

### Type-Check, Compile and Minify for Production

```sh
npm run build
```

### Lint with [ESLint](https://eslint.org/)

```sh
npm run lint
```

## Browser Compatibility

- Chrome/Edge 90+
- Firefox 88+
- Safari 14.1+

**Note**: This application requires SharedArrayBuffer support, which may require specific HTTP headers in production environments:

```
Cross-Origin-Embedder-Policy: require-corp
Cross-Origin-Opener-Policy: same-origin
```

## Performance Notes

- FFmpeg WASM loads on demand (when you first split a file)
- Large files (>50MB) may cause browser memory issues
- Processing time depends on file size and number of segments
- All processing happens locally in your browser

## Usage Tips

- **Adding Split Points**: Move the slider to your desired position, then click "Add Split Point"
- **Adjusting Positions**: Drag the red markers along the timeline to fine-tune split points
- **Removing Points**: Click the "X" button on any marker to remove it
- **Previewing Segments**: Click the play button next to each segment preview to listen
- **Playing Results**: After splitting, use the play button on each result to listen before downloading

## License

MIT
