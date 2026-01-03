# Cloudflare Deployment Guide for MP3 Splitter

## Problem
FFmpeg.wasm requires `SharedArrayBuffer` to function properly, which is only available when specific security headers are set. Cloudflare Pages doesn't add these headers by default, causing the error:
```
Failed to load FFmpeg: ReferenceError: SharedArrayBuffer is not defined
```

## Solution Implemented

### 1. Security Headers Configuration
Created [`public/_headers`](public/_headers) file to add required security headers:

```
/*
  Cross-Origin-Opener-Policy: same-origin
  Cross-Origin-Embedder-Policy: require-corp
```

These headers enable `SharedArrayBuffer` in the browser by creating a secure context.

### 2. FFmpeg Configuration Update
Updated [`src/components/FFmpegDemo.vue`](src/components/FFmpegDemo.vue:242) to use the single-threaded version of FFmpeg:

- Changed from `@ffmpeg/core-mt` (multi-threaded) to `@ffmpeg/core` (single-threaded)
- Removed the `workerURL` parameter since single-threaded version doesn't use workers
- Removed the `s0` configuration option

**Why single-threaded?**
- Multi-threaded version requires `SharedArrayBuffer` for worker communication
- Single-threaded version works without `SharedArrayBuffer` (though it's slower)
- With the security headers in place, both versions will work, but single-threaded is more reliable

## Deployment Steps

### Option 1: Cloudflare Pages (Git Integration)

1. Push your code to GitHub/GitLab/Bitbucket
2. Connect your repository to Cloudflare Pages
3. Configure build settings:
   - **Build command**: `npm run build`
   - **Build output directory**: `dist`
4. Deploy

### Option 2: Cloudflare Pages (Direct Upload)

1. Build your project locally:
   ```bash
   npm run build
   ```

2. Upload the `dist` folder to Cloudflare Pages using:
   - Cloudflare Dashboard → Pages → Create a project → Direct Upload
   - Or using Wrangler CLI:
     ```bash
     npx wrangler pages deploy dist
     ```

### Option 3: Cloudflare Workers + Assets

If you need more control, you can use Cloudflare Workers with Assets:

1. Build your project:
   ```bash
   npm run build
   ```

2. Create a `wrangler.toml` file:
   ```toml
   name = "mp3-splitter"
   main = "worker.js"
   assets = { directory = "./dist" }
   compatibility_date = "2024-01-01"
   ```

3. Deploy:
   ```bash
   npx wrangler pages deploy dist
   ```

## Important Notes

### Security Headers
The `_headers` file must be in the `public` folder (or `dist` after build) for Cloudflare to recognize it. These headers are applied to all routes matching the pattern (`/*`).

### Browser Compatibility
- Modern browsers (Chrome 92+, Firefox 89+, Safari 15.2+, Edge 92+) support `SharedArrayBuffer` with proper headers
- Older browsers will not work with FFmpeg.wasm

### Performance Considerations
- **Single-threaded version**: Slower processing but more compatible
- **Multi-threaded version**: Faster but requires the security headers
- With the headers configured, you can switch back to multi-threaded if needed by changing the baseURL back to `@ffmpeg/core-mt`

### Testing Locally
To test the headers locally before deployment:

1. Using Vite dev server (headers already configured in [`vite.config.ts`](vite.config.ts:18-22)):
   ```bash
   npm run dev
   ```

2. Verify headers in browser DevTools:
   - Open DevTools → Network tab
   - Reload the page
   - Click on the main document
   - Check Response Headers for:
     - `Cross-Origin-Opener-Policy: same-origin`
     - `Cross-Origin-Embedder-Policy: require-corp`

## Troubleshooting

### Still getting SharedArrayBuffer error?

1. **Clear browser cache** - Old headers might be cached
2. **Check headers in DevTools** - Verify the headers are actually being sent
3. **Hard refresh** - Ctrl+Shift+R (Windows) or Cmd+Shift+R (Mac)
4. **Check Cloudflare settings** - Ensure no custom rules override the headers

### FFmpeg not loading?

1. Check browser console for specific error messages
2. Verify CDN URLs are accessible
3. Check network tab for failed requests
4. Ensure you're using a compatible browser version

### Slow performance?

The single-threaded version is slower than multi-threaded. If performance is critical:
1. Ensure security headers are properly configured
2. Switch back to multi-threaded version by changing the baseURL to `@ffmpeg/core-mt`
3. Add back the `workerURL` parameter in the load configuration

## Additional Resources

- [FFmpeg.wasm Documentation](https://ffmpegwasm.netlify.app/)
- [Cloudflare Pages Documentation](https://developers.cloudflare.com/pages/)
- [SharedArrayBuffer Browser Support](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer)
- [COOP and COEP Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy)
