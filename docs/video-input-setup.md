# Video Input Setup for Live Stream Processing

This guide covers setting up video input sources for live stream processing with the `@vladmandic/human` library. Human supports multiple video input types across browser and Node.js environments.

---

## Table of Contents

1. [Browser-Based Video Input](#browser-based-video-input)
   - [WebCam API (`human.webcam.start()`)](#webcam-api-humanwebcamstart)
   - [HTMLVideoElement with Live Stream](#htmlvideoelement-with-live-stream)
   - [WebRTC Media Tracks](#webrtc-media-tracks)
   - [HLS/DASH Streaming](#hlsdash-streaming)
2. [Node.js Video Input](#nodejs-video-input)
   - [ffmpeg Pipeline with pipe2jpeg](#ffmpeg-pipeline-with-pipe2jpeg)
   - [Video File Processing](#video-file-processing)
   - [Real-Time vs Batch Processing](#real-time-vs-batch-processing)
3. [Configuration Options](#configuration-options)
4. [Performance Considerations](#performance-considerations)

---

## Browser-Based Video Input

### WebCam API (`human.webcam.start()`)

The simplest way to get webcam input in the browser is using Human's built-in `WebCam` utility (`human.webcam`). It wraps the browser's `getUserMedia` API and handles device selection, stream management, and DOM element association.

**Source:** [`src/util/webcam.ts`](https://github.com/vladmandic/human/blob/main/src/util/webcam.ts)

#### WebCamConfig Options

| Option   | Type                         | Default | Description                                                                 |
|----------|------------------------------|---------|-----------------------------------------------------------------------------|
| `element`| `string \| HTMLVideoElement` | `undefined` | DOM element ID, an actual `HTMLVideoElement`, or undefined (creates one)  |
| `debug`  | `boolean`                    | `true`  | Print debug messages to console                                             |
| `mode`   | `'front' \| 'back'`         | `'front'` | Camera facing mode (`'user'` or `'environment'`)                          |
| `crop`   | `boolean`                    | `false`  | Use `crop-and-scale` resize mode for better framing                        |
| `width`  | `number`                     | `0`     | Desired webcam width (ideal constraint)                                     |
| `height` | `number`                     | `0`     | Desired webcam height (ideal constraint)                                    |
| `id`     | `string`                     | `undefined` | `deviceId` of the specific video device to use                             |

#### Basic Webcam Setup

```javascript
import * as H from '@vladmandic/human';

const human = new H.Human({
  modelBasePath: 'models/',
  face: { enabled: true, mesh: { enabled: true }, emotion: { enabled: true } },
  body: { enabled: true },
  hand: { enabled: true },
  gesture: { enabled: true },
});

async function main() {
  await human.webcam.start({ crop: true, width: 640, height: 480, mode: 'front' });
  human.video(human.webcam.element);
}

main();
```

#### Selecting a Specific Camera

```javascript
async function main() {
  const devices = await human.webcam.enumerate();
  console.log('Available cameras:', devices);
  
  const selectedId = devices[0].deviceId;
  await human.webcam.start({ id: selectedId, width: 1280, height: 720 });
  human.video(human.webcam.element);
}

main();
```

#### Accessing Webcam Properties

```javascript
console.log('Width:', human.webcam.width);
console.log('Height:', human.webcam.height);
console.log('Label:', human.webcam.label);
console.log('Settings:', human.webcam.settings);
console.log('Capabilities:', human.webcam.capabilities);
```

#### Controlling Playback

```javascript
human.webcam.pause();
await human.webcam.play();
human.webcam.stop();
```

---

### HTMLVideoElement with Live Stream

If you already have a live video stream (from a file, network source, or other capture mechanism) assigned to an `HTMLVideoElement`, you can pass it directly to `human.detect()` or `human.video()`.

```javascript
const video = document.getElementById('myVideo');
const canvas = document.getElementById('myCanvas');

async function processFrame() {
  const result = await human.detect(video);
  human.draw.all(canvas, result);
  requestAnimationFrame(processFrame);
}

// Wait for video to be ready
video.addEventListener('loadeddata', () => {
  processFrame();
});
```

---

### WebRTC Media Tracks

For WebRTC-based video (e.g., from a peer connection or screen share), extract the video track and assign it to an `HTMLVideoElement`:

```javascript
const stream = await navigator.mediaDevices.getDisplayMedia({ video: true });
const videoTrack = stream.getVideoTracks()[0];

const video = document.createElement('video');
video.srcObject = new MediaStream([videoTrack]);
await video.play();

human.video(video);
```

---

### HLS/DASH Streaming

For HTTP-based streaming (HLS or DASH), use a library like [`hls.js`](https://github.com/video-dev/hls.js) or [`dash.js`](https://github.com/Dash-Industry-Forum/dash.js) to attach the stream to a video element, then pass that element to Human:

```html
<video id="video" controls autoplay muted></video>
```

```javascript
import Hls from 'hls.js';

const video = document.getElementById('video');
const hls = new Hls();
hls.loadSource('https://example.com/stream.m3u8');
hls.attachMedia(video);

hls.on(Hls.Events.MANIFEST_PARSED, () => {
  video.play();
  human.video(video);
});
```

---

## Node.js Video Input

### ffmpeg Pipeline with pipe2jpeg

For Node.js, the recommended approach for live video input is using `ffmpeg` to decode video frames and `pipe2jpeg` to parse individual JPEG frames from the output stream.

**Source:** [`demo/nodejs/node-video.js`](https://github.com/vladmandic/human/blob/main/demo/nodejs/node-video.js)

#### Dependencies

```bash
npm install pipe2jpeg @vladmandic/pilogger
# ffmpeg must be installed on the system
```

#### Complete Example

```javascript
const { spawn } = require('child_process');
const Pipe2Jpeg = require('pipe2jpeg');
const Human = require('@vladmandic/human');

const humanConfig = {
  modelBasePath: 'file://models/',
  filter: { enabled: false },
  face: { enabled: true, mesh: { enabled: true }, emotion: { enabled: true } },
  body: { enabled: false },
  hand: { enabled: false },
};

const human = new Human.Human(humanConfig);
const pipe2jpeg = new Pipe2Jpeg();
let busy = false;

const ffmpegParams = [
  '-loglevel', 'quiet',
  '-i', './input.mp4',
  '-an',
  '-c:v', 'mjpeg',
  '-pix_fmt', 'yuvj422p',
  '-f', 'image2pipe',
  'pipe:1',
];

function detect(jpegBuffer) {
  if (busy) return;
  busy = true;
  const tensor = human.tf.node.decodeJpeg(jpegBuffer, 3);
  const res = await human.detect(tensor);
  human.tf.dispose(tensor);
  console.log('Frame result:', res.face?.length, 'faces detected');
  busy = false;
}

async function main() {
  await human.tf.ready();
  await human.load();

  pipe2jpeg.on('data', (jpegBuffer) => detect(jpegBuffer));

  const ffmpeg = spawn('ffmpeg', ffmpegParams, { stdio: ['ignore', 'pipe', 'ignore'] });
  ffmpeg.stdout.pipe(pipe2jpeg);
}

main();
```

#### Controlling Frame Rate and Resolution

Add ffmpeg video filters to control the output:

```javascript
const ffmpegParams = [
  '-loglevel', 'quiet',
  '-i', './input.mp4',
  '-an',
  '-c:v', 'mjpeg',
  '-pix_fmt', 'yuvj422p',
  '-f', 'image2pipe',
  '-vf', 'fps=5,scale=640:480',
  'pipe:1',
];
```

---

### Video File Processing

For processing video files without real-time constraints, you can decode frames with ffmpeg as above but omit the `-re` flag. This processes as fast as possible rather than at the source frame rate.

```javascript
const ffmpegParams = [
  '-i', './video.mp4',
  '-an',
  '-c:v', 'mjpeg',
  '-pix_fmt', 'yuvj422p',
  '-f', 'image2pipe',
  '-vf', 'fps=10',
  'pipe:1',
];
```

---

### Real-Time vs Batch Processing

| Mode         | ffmpeg Flag  | Use Case                                    |
|--------------|--------------|---------------------------------------------|
| Real-Time    | `-re`        | Live streams, webcam feeds, real-time input |
| Batch        | (none)       | Pre-recorded files, offline processing      |

To enable real-time mode, add `-re` before the input:

```javascript
const ffmpegParams = [
  '-re', // process in real-time
  '-i', 'rtsp://camera-ip/stream',
  '-an',
  '-c:v', 'mjpeg',
  '-pix_fmt', 'yuvj422p',
  '-f', 'image2pipe',
  'pipe:1',
];
```

---

### Node.js Webcam (node-webcam)

For periodic screenshots from a system webcam using `fswebcam`:

**Source:** [`demo/nodejs/node-webcam.js`](https://github.com/vladmandic/human/blob/main/demo/nodejs/node-webcam.js)

#### Dependencies

```bash
npm install node-webcam @vladmandic/pilogger
# fswebcam must be installed on the system
```

```javascript
const nodeWebCam = require('node-webcam');
const tf = require('@tensorflow/tfjs-node');
const Human = require('@vladmandic/human');

const optionsCamera = {
  callbackReturn: 'buffer',
  saveShots: false,
};
const camera = nodeWebCam.create(optionsCamera);

const human = new Human.Human({ modelBasePath: 'file://models/' });

function buffer2tensor(buffer) {
  return human.tf.tidy(() => {
    const decode = human.tf.node.decodeImage(buffer, 3);
    let expand;
    if (decode.shape[2] === 4) {
      const channels = human.tf.split(decode, 4, 2);
      const rgb = human.tf.stack([channels[0], channels[1], channels[2]], 2);
      expand = human.tf.reshape(rgb, [1, decode.shape[0], decode.shape[1], 3]);
    } else {
      expand = human.tf.expandDims(decode, 0);
    }
    return human.tf.cast(expand, 'float32');
  });
}

async function detect() {
  setTimeout(() => detect(), 5000); // capture every 5 seconds
  camera.capture('webcam-snap', async (err, data) => {
    if (err) { console.error('Capture error:', err); return; }
    const tensor = buffer2tensor(data);
    const result = await human.detect(tensor);
    console.log('Face count:', result.face?.length);
  });
}

async function main() {
  await human.load();
  detect();
}

main();
```

---

## Configuration Options

### Browser Configuration

```javascript
const config = {
  modelBasePath: 'models/',
  backend: 'webgl',       // 'webgl', 'wasm', 'cpu'
  async: true,
  debug: false,
  filter: { enabled: true, equalization: true, flip: false },
  face: {
    enabled: true,
    detector: { rotation: false },
    mesh: { enabled: true },
    attention: { enabled: false },
    iris: { enabled: true },
    description: { enabled: true },
    emotion: { enabled: true },
  },
  body: { enabled: true },
  hand: { enabled: true },
  gesture: { enabled: true },
  object: { enabled: false },
  segmentation: { enabled: false },
};
```

### Node.js Configuration

```javascript
const config = {
  modelBasePath: 'file://models/',
  backend: 'tensorflow',  // 'tensorflow', 'wasm', 'cpu'
  async: true,
  filter: { enabled: false },
  face: {
    enabled: true,
    detector: { enabled: true, rotation: false },
    mesh: { enabled: true },
    iris: { enabled: true },
    description: { enabled: true },
    emotion: { enabled: true },
  },
};
```

---

## Performance Considerations

### Backend Selection

| Environment | Recommended Backend | Notes                                                                 |
|-------------|--------------------|-----------------------------------------------------------------------|
| Browser     | `webgl`            | Hardware-accelerated; use `wasm` if WebGL is unavailable             |
| Node.js     | `tensorflow`       | GPU-accelerated if CUDA is available; use `wasm` for portability     |
| Any         | `wasm`             | Portable fallback; slower than GPU backends                          |
| Any         | `cpu`              | Slowest option; only for testing or environments without GPU/WASM    |

### Model Selection

Enable only the models you need. Each additional model increases per-frame processing time:

```javascript
// Minimal - face detection only
face: { enabled: true, detector: { enabled: true } }

// Moderate - face + body + hands
face: { enabled: true, mesh: { enabled: true }, emotion: { enabled: false } },
body: { enabled: true },
hand: { enabled: true },

// Full - all models
face: { enabled: true, mesh: { enabled: true }, emotion: { enabled: true }, iris: { enabled: true }, description: { enabled: true } },
body: { enabled: true },
hand: { enabled: true },
gesture: { enabled: true },
```

### Input Resolution

Processing time scales with input resolution. Downscale inputs when high resolution is not needed:

- Use ffmpeg's `-vf scale=WIDTH:HEIGHT` filter in Node.js pipelines
- Set webcam `width` and `height` constraints in browser
- Smaller inputs significantly reduce inference time without major accuracy loss for most use cases

### Frame Processing Strategy

```javascript
// Skip frames if previous detection is still running
if (busy) return;
busy = true;
const result = await human.detect(tensor);
busy = false;
```

### Batching in Node.js

For processing multiple video sources or batch video files, use parallel worker threads:

```javascript
const { Worker } = require('worker_threads');

function createWorker(videoPath) {
  const worker = new Worker('./worker.js', { workerData: { videoPath } });
  return worker;
}
```

---

## Quick Reference

| Input Type          | Browser | Node.js | Use Case                           |
|---------------------|---------|---------|------------------------------------|
| WebCam              | Yes     | Limited | Live person capture                |
| Video File          | Yes     | Yes     | Offline/recorded content           |
| ffmpeg Pipeline     | No      | Yes     | Video decoding + processing        |
| WebRTC              | Yes     | No      | Peer-to-peer video                 |
| HLS/DASH            | Yes     | No      | HTTP-based streaming               |
| node-webcam         | No      | Yes     | Periodic webcam screenshots        |
