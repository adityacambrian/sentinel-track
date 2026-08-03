# Live Stream Worker Activity Logging with @vladmandic/human

This guide explains how to use the `@vladmandic/human` library to process a live video stream and log worker activity based on detection results. It covers browser and Node.js approaches, from setup to a complete logging pipeline.

---

## 1. Overview of the Human Library

The `@vladmandic/human` library is a TensorFlow.js-based computer vision library that provides real-time detection and analysis on images and video. It supports the following capabilities relevant to worker activity logging:

| Capability | Description |
|---|---|
| **Face Detection** | Detects faces in each frame with bounding boxes, landmarks, and confidence scores |
| **Body Pose Estimation** | Tracks body keypoints (PoseNet, BlazePose, EfficientPose, MoveNet) for posture and position |
| **Hand Tracking** | Detects hands with keypoints, finger curl, and finger direction |
| **Gesture Recognition** | Classifies gestures from face, body, hand, and iris results (e.g., `point`, `fist`, `pinch`) |
| **Emotion Detection** | Classifies emotions (`angry`, `disgust`, `fear`, `happy`, `sad`, `surprise`, `neutral`) |
| **Person Combining** | Merges face, body, and hand results into unified `PersonResult` objects with tracking IDs |

All results are returned as a `Result` object (see [src/result.ts](https://github.com/vladmandic/human/blob/main/src/result.ts)) with the following structure:

```typescript
interface Result {
  face: FaceResult[];
  body: BodyResult[];
  hand: HandResult[];
  gesture: GestureResult[];
  object: ObjectResult[];
  persons: PersonResult[];  // getter combining face + body + hand + gesture
  performance: Record<string, number>;
  timestamp: number;  // milliseconds since UNIX epoch
  width: number;
  height: number;
  error: string | null;
}
```

Each `PersonResult` contains:

```typescript
interface PersonResult {
  id: number;
  face: FaceResult;
  body: BodyResult | null;
  hands: { left: HandResult | null; right: HandResult | null };
  gestures: GestureResult[];
  box: Box;  // [x, y, width, height]
  boxRaw?: Box;  // normalized to 0..1
}
```

---

## 2. Setting Up a Live Video Stream Input

### Browser: Webcam via `human.webcam`

The `Human` class exposes a `webcam` property (`WebCam` instance, see [src/util/webcam.ts](https://github.com/vladmandic/human/blob/main/src/util/webcam.ts)) that handles webcam access and stream management.

```typescript
import { Human } from '@vladmandic/human';

const human = new Human({
  face: { enabled: true, emotion: { enabled: true } },
  body: { enabled: true },
  hand: { enabled: true },
  gesture: { enabled: true },
});

// Start the webcam and attach to a video element
await human.webcam.start({ crop: true, width: 640, height: 480 });
```

The `WebCam.start()` method accepts a `WebCamConfig` with these options:

| Option | Type | Description |
|---|---|---|
| `element` | `string \| HTMLVideoElement \| undefined` | DOM element ID or existing `<video>` element; creates a new one if omitted |
| `debug` | `boolean` | Print diagnostic messages |
| `mode` | `'front' \| 'back'` | Use front (`user`) or back (`environment`) camera |
| `crop` | `boolean` | Use `crop-and-scale` resize mode |
| `width` | `number` | Desired webcam width |
| `height` | `number` | Desired webcam height |
| `id` | `string` | Specific `deviceId` to use |

After starting, the webcam stream is available at `human.webcam.element` (an `HTMLVideoElement`).

### Browser: HTMLVideoElement with Live Stream

You can also pass any `<video>` element playing a live stream (WebRTC, HLS, DASH, or a local video file) directly to `human.detect()`:

```typescript
const video = document.getElementById('video') as HTMLVideoElement;
video.srcObject = remoteStream;  // e.g., from WebRTC peer connection
await video.play();

// Then in the detection loop:
const result = await human.detect(video);
```

Supported input types are defined in [src/exports.ts](https://github.com/vladmandic/human/blob/main/src/exports.ts) as the `Input` type: `Tensor | AnyCanvas | AnyImage | AnyVideo | ImageObjects | ExternalCanvas`.

### Browser: WebRTC Media Tracks

For WebRTC-based streams, obtain a `MediaStreamTrack` and attach it to a `<video>` element, then pass that element to Human:

```typescript
const stream = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
const video = document.createElement('video');
video.srcObject = stream;
await video.play();
const result = await human.detect(video);
```

### Node.js: Video File with ffmpeg Pipeline

In Node.js, use `ffmpeg` to decode a video into a stream of JPEG frames, then process each frame with `pipe2jpeg`:

```javascript
const { spawn } = require('child_process');
const Pipe2Jpeg = require('pipe2jpeg');
const Human = require('@vladmandic/human');

const human = new Human.Human({ modelBasePath: 'file://models/' });
const pipe2jpeg = new Pipe2Jpeg();

const ffmpeg = spawn('ffmpeg', [
  '-loglevel', 'quiet',
  '-i', './worker-stream.mp4',
  '-an', '-c:v', 'mjpeg', '-pix_fmt', 'yuvj422p',
  '-f', 'image2pipe', 'pipe:1',
]);

pipe2jpeg.on('data', (jpegBuffer) => {
  const tensor = human.tf.node.decodeJpeg(jpegBuffer, 3);
  human.detect(tensor).then((result) => {
    // process result
    human.tf.dispose(tensor);
  });
});

ffmpeg.stdout.pipe(pipe2jpeg);
```

See [demo/nodejs/node-video.js](https://github.com/vladmandic/human/blob/main/demo/nodejs/node-video.js) for the full example.

### Node.js: Webcam via node-webcam

For Node.js environments with a physical webcam, use the `node-webcam` package to capture snapshots at intervals:

```javascript
const nodeWebCam = require('node-webcam');
const Human = require('@vladmandic/human');

const camera = nodeWebCam.create({ callbackReturn: 'buffer', saveShots: false });
const human = new Human.Human({ modelBasePath: 'file://models/' });

camera.capture(tempFile, (err, data) => {
  const tensor = human.tf.node.decodeImage(data, 3);
  human.detect(tensor).then((result) => {
    // process result
    human.tf.dispose(tensor);
  });
});
```

See [demo/nodejs/node-webcam.js](https://github.com/vladmandic/human/blob/main/demo/nodejs/node-webcam.js) for the full example.

---

## 3. Running Detection on Each Frame

### Browser: Continuous Video Loop

The `Human.video()` method runs detection continuously on an `HTMLVideoElement`:

```typescript
// Start continuous detection on the webcam element
human.video(human.webcam.element);
```

This uses `requestAnimationFrame` to process each frame. You can also run a manual loop with `requestAnimationFrame` for more control:

```typescript
async function detectionLoop() {
  if (!human.webcam.element.paused && human.webcam.element.readyState >= 2) {
    const result = await human.detect(human.webcam.element);
    // process result here
  }
  requestAnimationFrame(detectionLoop);
}
detectionLoop();
```

For smoother rendering, separate the detection loop from the draw loop:

```typescript
// Detection loop - runs as fast as possible
async function detectLoop() {
  if (!video.paused && video.readyState >= 2) {
    await human.detect(video);
  }
  requestAnimationFrame(detectLoop);
}

// Draw loop - runs at ~30fps
function drawLoop() {
  const interpolated = human.next();  // smoothened result
  human.draw.canvas(video, canvas);
  human.draw.all(canvas, interpolated);
  setTimeout(drawLoop, 33);  // ~30fps
}
```

See [demo/tracker/index.ts](https://github.com/vladmandic/human/blob/main/demo/tracker/index.ts) for a full example of separated detection and draw loops.

### Node.js: Frame-by-Frame Processing

In Node.js, each frame is processed individually as a tensor. The `human.detect()` method returns a `Promise<Result>`:

```javascript
const tensor = human.tf.node.decodeJpeg(jpegBuffer, 3);
const result = await human.detect(tensor);
human.tf.dispose(tensor);  // always dispose after use
```

### Using Interpolated Results

For smoother tracking across frames, use `human.next()` which applies temporal interpolation to the last known result:

```typescript
const interpolated = human.next();  // uses human.result internally
// or with a specific result:
const smoothed = human.next(someResult);
```

See [src/util/interpolate.ts](https://github.com/vladmandic/human/blob/main/src/util/interpolate.ts) for the interpolation logic.

---

## 4. Extracting Worker-Relevant Data from Results

### Person Tracking with IDs

The `Result.persons` getter combines face, body, and hand results into unified `PersonResult` objects. Each person has a stable `id` that persists across frames when tracking is enabled:

```typescript
const result = await human.detect(video);
for (const person of result.persons) {
  console.log(`Person #${person.id}: box=${person.box}, score=${person.face?.score}`);
}
```

The person combining logic is implemented in [src/util/persons.ts](https://github.com/vladmandic/human/blob/main/src/util/persons.ts).

### Pose and Body Position

From `PersonResult.body`, extract keypoints for worker posture and position tracking:

```typescript
for (const person of result.persons) {
  if (person.body) {
    const keypoints = person.body.keypoints;
    // keypoints[0] = nose, keypoints[5] = left shoulder, etc.
    const nose = keypoints.find((k) => k.part === 'nose');
    const leftShoulder = keypoints.find((k) => k.part === 'leftShoulder');
    const rightShoulder = keypoints.find((k) => k.part === 'rightShoulder');
    // Use positions for worker pose analysis
  }
}
```

### Gesture Recognition

Worker gestures are available in `PersonResult.gestures` and per-hand `HandResult.landmarks`:

```typescript
for (const person of result.persons) {
  for (const gesture of person.gestures) {
    console.log(`Person #${person.id} gesture: part=${gesture.part}, gesture=${gesture.gesture}`);
  }
  // Per-hand gestures from landmarks
  if (person.hands.left) {
    const leftFingerCurls = person.hands.left.landmarks;
    // e.g., { index: { curl: 'none', direction: 'verticalUp' }, ... }
  }
}
```

Gesture types are defined in [src/result.ts](https://github.com/vladmandic/human/blob/main/src/result.ts): `HandType = 'hand' | 'fist' | 'pinch' | 'point' | 'face' | 'tip' | 'pinchtip'`.

### Emotion Detection

Worker emotional state is available from `FaceResult.emotion`:

```typescript
for (const person of result.persons) {
  if (person.face?.emotion) {
    const dominantEmotion = person.face.emotion.reduce((prev, curr) =>
      prev.score > curr.score ? prev : curr
    );
    console.log(`Person #${person.id} emotion: ${dominantEmotion.emotion} (${dominantEmotion.score})`);
  }
}
```

Emotion types: `'angry' | 'disgust' | 'fear' | 'happy' | 'sad' | 'surprise' | 'neutral'`.

### Timestamps

Every `Result` includes a `timestamp` field with `Date.now()` milliseconds since the UNIX epoch:

```typescript
console.log(`Detection at ${new Date(result.timestamp).toISOString()}`);
```

---

## 5. Logging the Results

### JSON Lines Format

For structured logging, write each detection as a JSON line to a file or stream:

```typescript
function formatLogEntry(result: Result): string {
  const entries = result.persons.map((person) => ({
    timestamp: result.timestamp,
    personId: person.id,
    score: person.face?.score ?? person.body?.score ?? 0,
    box: person.box,
    pose: person.body ? person.body.keypoints.map((k) => ({
      part: k.part,
      position: k.position,
      score: k.score,
    })) : null,
    gestures: person.gestures.map((g) => ({
      part: g.part,
      gesture: g.gesture,
    })),
    emotion: person.face?.emotion?.reduce((prev, curr) =>
      prev.score > curr.score ? prev : curr
    ) ?? null,
    hands: {
      left: person.hands.left ? {
        label: person.hands.left.label,
        landmarks: person.hands.left.landmarks,
      } : null,
      right: person.hands.right ? {
        label: person.hands.right.label,
        landmarks: person.hands.right.landmarks,
      } : null,
    },
    fps: result.performance.total ? 1000 / result.performance.total : 0,
  }));
  return entries.map((e) => JSON.stringify(e)).join('\n');
}
```

### CSV Format

For tabular analysis, log selected fields as CSV:

```typescript
function formatCsvEntry(result: Result): string {
  return result.persons.map((person) => {
    const emotion = person.face?.emotion?.reduce((p, c) => p.score > c.score ? p : c);
    return [
      result.timestamp,
      person.id,
      person.face?.score?.toFixed(3) ?? '',
      person.body?.score?.toFixed(3) ?? '',
      emotion?.emotion ?? '',
      emotion?.score?.toFixed(3) ?? '',
      person.box.join(','),
    ].join(',');
  }).join('\n');
}
```

### Handling Intermittent Detections

Workers may leave and re-enter the frame. Use the person `id` to track continuity and handle gaps:

```typescript
const activeWorkers = new Map<number, { lastSeen: number; activity: string[] }>();

function processResult(result: Result) {
  const now = result.timestamp;
  const currentIds = new Set<number>();

  for (const person of result.persons) {
    currentIds.add(person.id);
    const entry = activeWorkers.get(person.id);
    if (entry) {
      entry.lastSeen = now;
      // Append activity based on gestures/emotion
      const gesture = person.gestures[0];
      if (gesture) entry.activity.push(`${now}:${gesture.gesture}`);
    } else {
      activeWorkers.set(person.id, { lastSeen: now, activity: [] });
    }
  }

  // Mark workers that haven't been seen for 30 seconds as gone
  for (const [id, entry] of activeWorkers) {
    if (!currentIds.has(id) && now - entry.lastSeen > 30000) {
      console.log(`Worker #${id} left frame at ${new Date(now).toISOString()}`);
      activeWorkers.delete(id);
    }
  }
}
```

### Performance Monitoring

The `Result.performance` object contains timing values for each detection operation. Use it to monitor fps and latency:

```typescript
const fps = 1000 / (result.performance.total || 1);
const faceLatency = result.performance.face || 0;
const bodyLatency = result.performance.body || 0;
const handLatency = result.performance.hand || 0;
const gestureLatency = result.performance.gesture || 0;

console.log(`FPS: ${fps.toFixed(1)} | Face: ${faceLatency}ms | Body: ${bodyLatency}ms | Hand: ${handLatency}ms | Gesture: ${gestureLatency}ms`);
```

---

## 6. Complete Code Example: Live Stream Processing Loop with Logging

### Browser Example

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Worker Activity Logger</title>
  <style>
    body { font-family: monospace; background: #1a1a1a; color: #e0e0e0; margin: 0; padding: 16px; }
    #log { height: 300px; overflow-y: auto; background: #0d0d0d; padding: 8px; border-radius: 4px; font-size: 13px; }
    #status { padding: 8px; font-size: 14px; }
  </style>
</head>
<body>
  <div id="status">Initializing...</div>
  <video id="video" autoplay playsinline style="display:none"></video>
  <div id="log"></div>

  <script type="module">
    import { Human } from 'https://cdn.jsdelivr.net/npm/@vladmandic/human/dist/human.esm.js';

    const human = new Human({
      backend: 'webgl',
      modelBasePath: 'https://vladmandic.github.io/human-models/models',
      filter: { enabled: true, equalization: true },
      face: { enabled: true, emotion: { enabled: true }, iris: { enabled: true } },
      body: { enabled: true },
      hand: { enabled: true },
      gesture: { enabled: true },
      object: { enabled: false },
      segmentation: { enabled: false },
    });

    const video = document.getElementById('video');
    const logEl = document.getElementById('log');
    const statusEl = document.getElementById('status');
    const activeWorkers = new Map();
    const LOG_INTERVAL = 5000;  // log summary every 5 seconds
    let lastLogTime = 0;

    function appendLog(msg) {
      const line = document.createElement('div');
      line.textContent = `[${new Date().toISOString()}] ${msg}`;
      logEl.appendChild(line);
      logEl.scrollTop = logEl.scrollHeight;
    }

    function processResult(result) {
      const ts = result.timestamp;
      const fps = result.performance.total ? (1000 / result.performance.total).toFixed(1) : 'N/A';

      for (const person of result.persons) {
        const emotion = person.face?.emotion?.reduce((p, c) => p.score > c.score ? p : c);
        const gesture = person.gestures[0];
        const pose = person.body ? person.body.keypoints.length + ' keypoints' : 'no body';

        const entry = {
          timestamp: ts,
          personId: person.id,
          score: person.face?.score ?? person.body?.score ?? 0,
          pose,
          gesture: gesture ? `${gesture.part}:${gesture.gesture}` : 'none',
          emotion: emotion ? emotion.emotion : 'unknown',
          emotionScore: emotion ? emotion.score.toFixed(2) : 'N/A',
          fps,
        };

        const existing = activeWorkers.get(person.id);
        if (existing) {
          existing.lastSeen = ts;
          existing.activityCount++;
        } else {
          activeWorkers.set(person.id, { firstSeen: ts, lastSeen: ts, activityCount: 1 });
          appendLog(`Worker #${person.id} ENTERED frame | pose: ${entry.pose} | emotion: ${entry.emotion}`);
        }

        // Log individual gesture/emotion events
        if (gesture && gesture.gesture !== 'none') {
          appendLog(`  Worker #${person.id} gesture: ${gesture.gesture} (score: ${person.face?.score?.toFixed(2)})`);
        }
      }

      // Check for workers that left
      const now = Date.now();
      for (const [id, worker] of activeWorkers) {
        if (!result.persons.some((p) => p.id === id) && now - worker.lastSeen > 3000) {
          appendLog(`Worker #${id} LEFT frame (seen for ${worker.activityCount} frames)`);
          activeWorkers.delete(id);
        }
      }

      // Periodic summary log
      if (now - lastLogTime > LOG_INTERVAL) {
        lastLogTime = now;
        const summary = Array.from(activeWorkers.entries())
          .map(([id, w]) => `#${id}:${w.activityCount}f`)
          .join(' | ');
        appendLog(`--- SUMMARY (${activeWorkers.size} workers): ${summary || 'none'} ---`);
      }
    }

    async function detectionLoop() {
      if (!video.paused && video.readyState >= 2) {
        try {
          const result = await human.detect(video);
          processResult(result);
        } catch (err) {
          appendLog(`Detection error: ${err}`);
        }
      }
      requestAnimationFrame(detectionLoop);
    }

    async function main() {
      statusEl.textContent = 'Starting webcam...';
      await human.load();
      await human.webcam.start({ crop: true, width: 640, height: 480 });
      video.width = human.webcam.width;
      video.height = human.webcam.height;
      statusEl.textContent = `Detecting | ${human.webcam.width}x${human.webcam.height} | ${human.models.loaded()} models loaded`;
      appendLog(`Human v${human.version} | TFJS v${human.tf.version['tfjs-core']} | Backend: ${human.tf.getBackend()}`);
      detectionLoop();
    }

    window.onload = main;
  </script>
</body>
</html>
```

### Node.js Example

```javascript
const fs = require('fs');
const { spawn } = require('child_process');
const Pipe2Jpeg = require('pipe2jpeg');
const Human = require('@vladmandic/human');

const LOG_STREAM = fs.createWriteStream('worker-activity.log', { flags: 'a' });

const humanConfig = {
  modelBasePath: 'file://models/',
  backend: 'cpu',
  async: true,
  filter: { enabled: true },
  face: { enabled: true, emotion: { enabled: true }, iris: { enabled: true } },
  body: { enabled: true },
  hand: { enabled: true },
  gesture: { enabled: true },
  object: { enabled: false },
};

const human = new Human.Human(humanConfig);
const pipe2jpeg = new Pipe2Jpeg();
const activeWorkers = new Map();
let frameCount = 0;

function logEntry(result) {
  const ts = result.timestamp;
  const iso = new Date(ts).toISOString();

  for (const person of result.persons) {
    const emotion = person.face?.emotion?.reduce((p, c) => p.score > c.score ? p : c);
    const gesture = person.gestures[0];

    const entry = {
      timestamp: ts,
      iso,
      personId: person.id,
      score: person.face?.score ?? person.body?.score ?? 0,
      pose: person.body ? person.body.keypoints.length + ' keypoints' : null,
      gesture: gesture ? { part: gesture.part, gesture: gesture.gesture } : null,
      emotion: emotion ? { emotion: emotion.emotion, score: emotion.score } : null,
      box: person.box,
      fps: result.performance.total ? (1000 / result.performance.total).toFixed(1) : null,
    };

    LOG_STREAM.write(JSON.stringify(entry) + '\n');

    // Track worker presence
    const existing = activeWorkers.get(person.id);
    if (existing) {
      existing.lastSeen = ts;
      existing.frames++;
    } else {
      activeWorkers.set(person.id, { firstSeen: ts, lastSeen: ts, frames: 1 });
      LOG_STREAM.write(JSON.stringify({ timestamp: ts, event: 'worker_entered', personId: person.id }) + '\n');
    }
  }

  // Detect workers leaving (not seen for 3+ seconds)
  const now = Date.now();
  for (const [id, worker] of activeWorkers) {
    if (!result.persons.some((p) => p.id === id) && now - worker.lastSeen > 3000) {
      LOG_STREAM.write(JSON.stringify({ timestamp: now, event: 'worker_left', personId: id, framesSeen: worker.frames }) + '\n');
      activeWorkers.delete(id);
    }
  }

  frameCount++;
  if (frameCount % 30 === 0) {
    const active = activeWorkers.size;
    LOG_STREAM.write(JSON.stringify({ timestamp: now, event: 'summary', activeWorkers: active, totalFrames: frameCount }) + '\n');
  }
}

async function detect(jpegBuffer) {
  if (!jpegBuffer) return;
  const tensor = human.tf.node.decodeJpeg(jpegBuffer, 3);
  try {
    const result = await human.detect(tensor);
    logEntry(result);
  } catch (err) {
    LOG_STREAM.write(JSON.stringify({ timestamp: Date.now(), event: 'error', error: err.message }) + '\n');
  } finally {
    human.tf.dispose(tensor);
  }
}

async function main() {
  await human.tf.ready();
  await human.load();
  console.log(`Human v${human.version} | Models loaded: ${human.models.loaded()}`);

  const ffmpeg = spawn('ffmpeg', [
    '-loglevel', 'quiet',
    '-i', './worker-stream.mp4',
    '-an', '-c:v', 'mjpeg', '-pix_fmt', 'yuvj422p',
    '-f', 'image2pipe', 'pipe:1',
  ]);

  pipe2jpeg.on('data', (jpegBuffer) => detect(jpegBuffer));
  ffmpeg.stdout.pipe(pipe2jpeg);

  ffmpeg.on('exit', () => {
    LOG_STREAM.end();
    console.log('Processing complete.');
  });
}

main();
```

---

## 7. Node.js vs Browser Approaches

| Aspect | Browser | Node.js |
|---|---|---|
| **Video Input** | `HTMLVideoElement` via webcam, `<video>` tag, WebRTC | `ffmpeg` pipe to JPEG, `node-webcam` snapshots, video file |
| **Tensor Decoding** | Automatic via `human.detect(input)` | Manual: `human.tf.node.decodeJpeg()` or `human.tf.node.decodeImage()` |
| **Backend** | `webgl`, `humangl`, `wasm` (browser-optimized) | `cpu`, `webgl` (via tfjs-node) |
| **Model Loading** | Automatic on first detect; models from CDN or local path | Requires `file://models/` path; models must be pre-downloaded |
| **Continuous Detection** | `human.video(element)` or manual `requestAnimationFrame` loop | Event-driven: process each frame as it arrives from `pipe2jpeg` |
| **Tensor Disposal** | Automatic (handled internally by `human.detect()`) | Manual: `human.tf.dispose(tensor)` required after each detect |
| **Logging Output** | Console, DOM elements, or `fetch()` to server | File streams (`fs.createWriteStream`), stdout, or network |
| **Performance** | GPU-accelerated via WebGL; ~30fps typical | CPU-bound; throughput depends on model and hardware |
| **Dependencies** | Only the Human library (self-contained ESM) | `@tensorflow/tfjs-node`, `pipe2jpeg`, `ffmpeg`, optional `node-webcam` |

### Key Differences to Keep in Mind

1. **Tensor Lifecycle**: In Node.js, you must manually decode JPEG buffers into tensors and dispose them after detection. In the browser, `human.detect()` handles all image-to-tensor conversion internally.

2. **Backend Selection**: In Node.js, use `backend: 'cpu'` for stability or `'webgl'` for GPU acceleration (requires `tfjs-node` with native bindings). In the browser, prefer `'webgl'` or `'humangl'` for best performance.

3. **Model Path**: In Node.js, models must be accessible via `file://` URLs. In the browser, models can be loaded from a CDN or relative paths.

4. **Real-time vs Batch**: Node.js is better suited for batch processing of recorded streams (e.g., processing an MP4 file after the fact). The browser is ideal for real-time processing of live webcam or WebRTC streams.

5. **Event System**: Both environments support the `human.events` event target for lifecycle notifications (`create`, `load`, `image`, `detect`, `warmup`, `error`). Use this in either environment to hook into processing stages for logging.

---

## References

- **Human Class API**: [src/human.ts](https://github.com/vladmandic/human/blob/main/src/human.ts) — main `Human` class with `detect()`, `video()`, `webcam`, `next()`, `load()`, `warmup()`
- **Result Types**: [src/result.ts](https://github.com/vladmandic/human/blob/main/src/result.ts) — `Result`, `PersonResult`, `FaceResult`, `BodyResult`, `HandResult`, `GestureResult`
- **WebCam Handling**: [src/util/webcam.ts](https://github.com/vladmandic/human/blob/main/src/util/webcam.ts) — `WebCam` class with `start()`, `stop()`, `pause()`, `play()`
- **Person Combining**: [src/util/persons.ts](https://github.com/vladmandic/human/blob/main/src/util/persons.ts) — merges face/body/hand into `PersonResult`
- **Temporal Interpolation**: [src/util/interpolate.ts](https://github.com/vladmandic/human/blob/main/src/util/interpolate.ts) — smoothens results across frames
- **Simple Webcam Demo**: [demo/video/index.html](https://github.com/vladmandic/human/blob/main/demo/video/index.html)
- **Tracking Demo**: [demo/tracker/index.ts](https://github.com/vladmandic/human/blob/main/demo/tracker/index.ts)
- **Node.js Video Processing**: [demo/nodejs/node-video.js](https://github.com/vladmandic/human/blob/main/demo/nodejs/node-video.js)
- **Node.js Webcam**: [demo/nodejs/node-webcam.js](https://github.com/vladmandic/human/blob/main/demo/nodejs/node-webcam.js)
- **Node.js Event-Based Processing**: [demo/nodejs/node-event.js](https://github.com/vladmandic/human/blob/main/demo/nodejs/node-event.js)