# Activity Logging Guide

This guide explains how to log worker activity from Human library detection results on a live video stream. It covers understanding the result structure, tracking workers across frames, extracting relevant data, formatting log entries, and production best practices.

## Table of Contents

1. [Understanding the Result Object](#understanding-the-result-object)
2. [Tracking Workers Across Frames](#tracking-workers-across-frames)
3. [Extracting Worker-Relevant Data](#extracting-worker-relevant-data)
4. [Formatting and Writing Log Entries](#formatting-and-writing-log-entries)
5. [Handling Intermittent Detections](#handling-intermittent-detections)
6. [Performance Monitoring](#performance-monitoring)
7. [Complete Logging Pipeline Example](#complete-logging-pipeline-example)
8. [Production Tips](#production-tips)

---

## Understanding the Result Object

Every call to `human.detect()` produces a `Result` object containing all detection data for that frame. The `Result` interface is defined in `src/result.ts`.

### Result Structure

```typescript
interface Result {
  face: FaceResult[];        // Face detection & analysis results
  body: BodyResult[];        // Body pose detection results
  hand: HandResult[];        // Hand detection & analysis results
  gesture: GestureResult[];  // Gesture recognition results
  object: ObjectResult[];    // General object detection results
  performance: Record<string, number>;  // Timing data for each operation
  canvas?: AnyCanvas | null;  // Optional processed canvas
  timestamp: number;          // Milliseconds since UNIX epoch
  persons: PersonResult[];    // Unified per-person view (getter)
  error: string | null;       // Last error message
  width: number;              // Resolution width
  height: number;             // Resolution height
}
```

### FaceResult

```typescript
interface FaceResult {
  id: number;
  score: number;       // Overall face score
  boxScore: number;    // Detection score
  faceScore: number;   // Mesh score
  box: Box;            // [x, y, width, height] in pixels
  boxRaw: Box;         // Normalized to 0..1
  mesh: Point[];       // Face mesh keypoints
  meshRaw: Point[];    // Normalized mesh
  annotations: Record<FaceLandmark, Point[]>;
  age?: number;
  gender?: 'male' | 'female' | 'unknown';
  genderScore?: number;
  emotion?: { score: number; emotion: Emotion }[];
  race?: { score: number; race: Race }[];
  embedding?: number[];
  distance?: number;   // Distance from camera
  real?: number;       // Anti-spoofing confidence
  live?: number;       // Liveness confidence
  rotation?: {
    angle: { roll: number; yaw: number; pitch: number };
    matrix: [number, number, ...];
    gaze: { bearing: number; strength: number };
  };
}
```

### BodyResult

```typescript
interface BodyKeypoint {
  part: BodyLandmark;
  position: Point;        // [x, y, z?] in pixels
  positionRaw: Point;     // Normalized to 0..1
  distance?: Point;       // Relative to body center in meters
  score: number;
}

interface BodyResult {
  id: number;
  score: number;
  box: Box;
  boxRaw: Box;
  keypoints: BodyKeypoint[];
  annotations: Record<BodyAnnotation, Point[][]>;
}
```

### HandResult

```typescript
interface HandResult {
  id: number;
  score: number;
  boxScore: number;
  fingerScore: number;
  box: Box;
  boxRaw: Box;
  keypoints: Point[];
  label: HandType;  // 'hand' | 'fist' | 'pinch' | 'point' | 'face' | 'tip' | 'pinchtip'
  annotations: Record<Finger, Point[]>;
  landmarks: Record<Finger, { curl: FingerCurl; direction: FingerDirection }>;
}
```

### GestureResult

Gestures are discriminated unions keyed by the part they belong to:

```typescript
type GestureResult =
  | { face: number; gesture: FaceGesture }
  | { iris: number; gesture: IrisGesture }
  | { body: number; gesture: BodyGesture }
  | { hand: number; gesture: HandGesture };
```

The numeric ID references the corresponding face, body, or hand result.

### PersonResult

The `persons` property on `Result` is a getter that combines all individual detections into unified person objects. It is computed by `join()` in `src/util/persons.ts`.

```typescript
interface PersonResult {
  id: number;
  face: FaceResult;
  body: BodyResult | null;
  hands: { left: HandResult | null; right: HandResult | null };
  gestures: GestureResult[];
  box: Box;           // Combined bounding box
  boxRaw?: Box;       // Normalized combined box
}
```

The person combining logic matches body and hands to faces based on spatial overlap (bounding box intersection). Each face becomes a person entry, and associated body/hand results are joined if their boxes overlap.

### Performance Object

The `performance` record contains millisecond timings for each detection stage:

```typescript
{
  face: number,       // Face detection + analysis time
  body: number,       // Body detection time
  hand: number,       // Hand detection time
  gesture: number,    // Gesture recognition time
  object: number,     // Object detection time
  total: number,      // Total detection time
  interpolate?: number // Interpolation time (if enabled)
}
```

---

## Tracking Workers Across Frames

### Person IDs

Each `PersonResult` has a sequential `id` assigned by the `join()` function in `src/util/persons.ts`. This ID is only valid within a single frame — it resets on each `detect()` call. To track a worker across frames, you need to match persons between consecutive frames using spatial proximity.

### Spatial Matching

The simplest approach is to match persons by the overlap or distance between their bounding boxes:

```typescript
function matchPersons(prevPersons: PersonResult[], currPersons: PersonResult[]): Map<number, number> {
  const mapping = new Map<number, number>(); // prevId -> currId

  for (const prev of prevPersons) {
    let bestMatch: PersonResult | null = null;
    let bestIoU = 0;

    for (const curr of currPersons) {
      const iou = computeIoU(prev.box, curr.box);
      if (iou > bestIoU && iou > 0.3) {
        bestIoU = iou;
        bestMatch = curr;
      }
    }

    if (bestMatch) {
      mapping.set(prev.id, bestMatch.id);
    }
  }

  return mapping;
}

function computeIoU(boxA: Box, boxB: Box): number {
  const [ax, ay, aw, ah] = boxA;
  const [bx, by, bw, bh] = boxB;

  const x1 = Math.max(ax, bx);
  const y1 = Math.max(ay, by);
  const x2 = Math.min(ax + aw, bx + bw);
  const y2 = Math.min(ay + ah, by + bh);

  const intersection = Math.max(0, x2 - x1) * Math.max(0, y2 - y1);
  const areaA = aw * ah;
  const areaB = bw * bh;

  return intersection / (areaA + areaB - intersection);
}
```

### Using the Tracker Utility

The `demo/tracker/index.ts` demonstrates using a dedicated tracking module (`tracker.js`) that handles matching with configurable parameters:

```typescript
const trackerConfig = {
  unMatchedFramesTolerance: 100, // Frames before considering a track gone
  iouLimit: 0.05,                // Minimum IOU for matching
  fastDelete: false,             // Remove immediately if unmatched
  distanceLimit: 1e4,            // Maximum distance for matching
  matchingAlgorithm: 'kdTree',   // 'kdTree' or 'munkres'
};

// Feed tracked items each frame
const items = tracking.map((obj) => ({
  x: obj.box[0] + obj.box[2] / 2,
  y: obj.box[1] + obj.box[3] / 2,
  w: obj.box[2],
  h: obj.box[3],
  name: obj.label || 'person',
  confidence: obj.score,
}));

tracker.updateTrackedItemsWithNewFrame(items, frameNumber);
const trackedData = tracker.getJSONOfTrackedItems(true);
```

Tracked items include a persistent `id` across frames, `isZombie` flag for lost tracks, and bearing/distance information.

### Temporal Smoothing

Use `calc()` from `src/util/interpolate.ts` to smooth detection results between frames. This reduces jitter in position data:

```typescript
import { calc } from '@vladmandic/human/dist/interpolate.esm.js';

let lastResult: Result | null = null;

// In your detection loop:
const rawResult = await human.detect(video, config);
const smoothedResult = calc(rawResult, config);
lastResult = smoothedResult;
```

The interpolation formula blends previous and current values using a buffered factor based on detection delay. At low delays, results converge ~28% toward live data per frame; at high delays (>1s), live data is used directly.

---

## Extracting Worker-Relevant Data

### Person Detection

Access unified person data via `result.persons`. This is the recommended entry point for worker activity logging because it already combines face, body, and hands:

```typescript
for (const person of result.persons) {
  const hasFace = !!person.face;
  const hasBody = !!person.body;
  const hasLeftHand = !!person.hands.left;
  const hasRightHand = !!person.hands.right;
  const gestureCount = person.gestures.length;
}
```

### Pose and Body Position

Extract body keypoints for position tracking:

```typescript
for (const person of result.persons) {
  if (!person.body) continue;

  const keypoints = person.body.keypoints;
  const positions: Record<string, { x: number; y: number; score: number }> = {};

  for (const kp of keypoints) {
    positions[kp.part] = {
      x: kp.position[0],
      y: kp.position[1],
      score: kp.score,
    };
  }

  // Key landmarks for worker activity
  const nose = positions['nose'];
  const leftWrist = positions['leftWrist'];
  const rightWrist = positions['rightWrist'];
  const leftShoulder = positions['leftShoulder'];
  const rightShoulder = positions['rightShoulder'];
}
```

### Gesture Recognition

Gestures tell you what actions a worker is performing:

```typescript
for (const person of result.persons) {
  for (const gesture of person.gestures) {
    if ('face' in gesture) {
      console.log(`Face gesture: ${gesture.gesture} on face ${gesture.face}`);
    } else if ('body' in gesture) {
      console.log(`Body gesture: ${gesture.gesture} on body ${gesture.body}`);
    } else if ('hand' in gesture) {
      console.log(`Hand gesture: ${gesture.hand} (${gesture.gesture})`);
    } else if ('iris' in gesture) {
      console.log(`Iris gesture: ${gesture.gesture}`);
    }
  }
}
```

### Emotion Detection

Emotion data is available on `FaceResult.emotion`:

```typescript
for (const person of result.persons) {
  if (person.face.emotion) {
    const sorted = [...person.face.emotion].sort((a, b) => b.score - a.score);
    const topEmotion = sorted[0];
    console.log(`Primary emotion: ${topEmotion.emotion} (${(topEmotion.score * 100).toFixed(1)}%)`);

    for (const e of sorted) {
      console.log(`  ${e.emotion}: ${(e.score * 100).toFixed(1)}%`);
    }
  }
}
```

### Timestamps

Use `result.timestamp` for precise frame timing:

```typescript
const timestamp = result.timestamp; // milliseconds since UNIX epoch
const date = new Date(timestamp);
const isoString = date.toISOString();
```

---

## Formatting and Writing Log Entries

### JSON Lines (Recommended)

JSON Lines (one JSON object per line) is the most flexible format for streaming log data:

```typescript
import fs from 'fs';

const logStream = fs.createWriteStream('activity-log.jsonl', { flags: 'a' });

function logFrame(result: Result, workerId: string) {
  for (const person of result.persons) {
    const entry = {
      ts: result.timestamp,
      workerId,
      personId: person.id,
      detected: true,
      face: person.face ? {
        score: person.face.score,
        age: person.face.age,
        gender: person.face.gender,
        box: person.face.box,
        emotion: person.face.emotion?.map(e => ({
          emotion: e.emotion,
          score: e.score,
        })),
      } : null,
      body: person.body ? {
        score: person.body.score,
        box: person.body.box,
        keypoints: person.body.keypoints.map(kp => ({
          part: kp.part,
          x: kp.position[0],
          y: kp.position[1],
          score: kp.score,
        })),
      } : null,
      hands: person.hands ? {
        left: person.hands.left ? {
          label: person.hands.left.label,
          score: person.hands.left.score,
          box: person.hands.left.box,
        } : null,
        right: person.hands.right ? {
          label: person.hands.right.label,
          score: person.hands.right.score,
          box: person.hands.right.box,
        } : null,
      } : null,
      gestures: person.gestures.map(g => {
        if ('face' in g) return { part: 'face', gesture: g.gesture };
        if ('body' in g) return { part: 'body', gesture: g.gesture };
        if ('hand' in g) return { part: 'hand', gesture: g.gesture };
        if ('iris' in g) return { part: 'iris', gesture: g.gesture };
        return { part: 'unknown', gesture: g.gesture };
      }),
      box: person.box,
      performance: result.performance,
    };

    logStream.write(JSON.stringify(entry) + '\n');
  }
}
```

### CSV Format

For simpler analysis in spreadsheets:

```typescript
const csvHeaders = [
  'timestamp', 'worker_id', 'person_id', 'detected',
  'face_score', 'age', 'gender', 'primary_emotion',
  'body_score', 'nose_x', 'nose_y',
  'left_hand_label', 'right_hand_label',
  'gesture_count', 'fps',
];

function toCsvRow(result: Result, workerId: string, fps: number): string {
  const person = result.persons[0]; // Primary person
  const emotion = person?.face?.emotion?.sort((a, b) => b.score - a.score)?.[0];
  const noseKp = person?.body?.keypoints.find(kp => kp.part === 'nose');

  return [
    result.timestamp,
    workerId,
    person?.id ?? '',
    person ? 1 : 0,
    person?.face?.score ?? '',
    person?.face?.age ?? '',
    person?.face?.gender ?? '',
    emotion?.emotion ?? '',
    person?.body?.score ?? '',
    noseKp?.position[0] ?? '',
    noseKp?.position[1] ?? '',
    person?.hands?.left?.label ?? '',
    person?.hands?.right?.label ?? '',
    person?.gestures.length ?? 0,
    fps,
  ].join(',');
}
```

### Structured Logging

Use a logging library for structured output:

```typescript
import * as log from '@vladmandic/pilogger';

log.add({ console: { format: 'json' } });

for (const person of result.persons) {
  log.data('worker.frame', {
    timestamp: result.timestamp,
    workerId,
    personId: person.id,
    face: person.face ? { score: person.face.score, age: person.face.age } : null,
    body: person.body ? { score: person.body.score } : null,
    gestures: person.gestures.map(g => g.gesture),
  });
}
```

---

## Handling Intermittent Detections

Workers will periodically leave and re-enter the frame. Logging should handle these gaps gracefully.

### Detection State Machine

```typescript
interface WorkerState {
  workerId: string;
  lastSeen: number;
  consecutiveMisses: number;
  currentPersonId: number | null;
  activityLog: ActivityEntry[];
}

interface ActivityEntry {
  timestamp: number;
  event: 'enter' | 'exit' | 'detected';
  data?: any;
}

const MAX_MISSES = 30; // Consider worker gone after 30 consecutive misses

function updateWorkerState(
  state: WorkerState,
  result: Result,
  matchedPersonIds: Set<number>,
): void {
  const wasDetected = matchedPersonIds.has(state.currentPersonId ?? -1);

  if (wasDetected) {
    state.consecutiveMisses = 0;
    state.lastSeen = result.timestamp;

    if (!state.currentPersonId) {
      state.activityLog.push({
        timestamp: result.timestamp,
        event: 'enter',
      });
      state.currentPersonId = matchedPersonIds.values().next().value;
    }

    state.activityLog.push({
      timestamp: result.timestamp,
      event: 'detected',
      data: extractPersonData(result, state.currentPersonId),
    });
  } else {
    state.consecutiveMisses++;

    if (state.currentPersonId && state.consecutiveMisses >= MAX_MISSES) {
      state.activityLog.push({
        timestamp: result.timestamp,
        event: 'exit',
        data: { lastSeen: state.lastSeen },
      });
      state.currentPersonId = null;
      state.consecutiveMisses = 0;
    }
  }
}
```

### Smoothing with Interpolation

Use temporal interpolation to fill gaps during brief occlusions:

```typescript
import { calc } from '@vladmandic/human/dist/interpolate.esm.js';

// The interpolator smooths jitter and provides best-guess positions
// during brief detection gaps (up to a few frames)
const smoothed = calc(rawResult, config);
```

The interpolator uses a convergence formula that naturally handles intermittent detections by blending toward the last known position.

### Log Gaps Explicitly

Always log when a worker enters or exits the frame, even if no detection data is available:

```typescript
function logWorkerEvent(workerId: string, event: 'enter' | 'exit' | 'gap', timestamp: number) {
  const entry = { ts: timestamp, workerId, event };
  logStream.write(JSON.stringify(entry) + '\n');
}
```

---

## Performance Monitoring

### Using the Performance Object

Every `Result` includes a `performance` record with per-stage timings:

```typescript
for (const person of result.persons) {
  const perf = result.performance;
  console.log(`Face: ${perf.face?.toFixed(1)}ms`);
  console.log(`Body: ${perf.body?.toFixed(1)}ms`);
  console.log(`Hand: ${perf.hand?.toFixed(1)}ms`);
  console.log(`Gesture: ${perf.gesture?.toFixed(1)}ms`);
  console.log(`Object: ${perf.object?.toFixed(1)}ms`);
  console.log(`Total: ${perf.total?.toFixed(1)}ms`);
  if (perf.interpolate) {
    console.log(`Interpolate: ${perf.interpolate.toFixed(1)}ms`);
  }
}
```

### FPS Tracking

Track detection and draw FPS separately (as shown in `demo/tracker/index.ts`):

```typescript
const fps = {
  detectFPS: 0,
  frames: 0,
  averageMs: 0,
};
const timestamps = { detect: 0, start: 0 };

async function detectionLoop() {
  if (!video.paused && video.readyState >= 2) {
    if (timestamps.start === 0) timestamps.start = human.now();
    await human.detect(video, config);
    fps.detectFPS = Math.round(1000 * 1000 / (human.now() - timestamps.detect)) / 1000;
    fps.frames++;
    fps.averageMs = Math.round(1000 * (human.now() - timestamps.start) / fps.frames) / 1000;
  }
  timestamps.detect = human.now();
  requestAnimationFrame(detectionLoop);
}
```

### Memory Monitoring

Monitor tensor memory to detect leaks:

```typescript
const tensors = human.tf.memory().numTensors;
if (tensors - lastTensorCount !== 0) {
  console.warn('Tensor leak detected:', tensors - lastTensorCount);
}
lastTensorCount = tensors;
```

### Latency Logging

Log detection latency alongside activity data:

```typescript
const detectStart = performance.now();
await human.detect(video, config);
const detectLatency = performance.now() - detectStart;

logStream.write(JSON.stringify({
  ts: result.timestamp,
  type: 'performance',
  detectLatencyMs: detectLatency,
  fps: fps.detectFPS,
  tensorCount: human.tf.memory().numTensors,
  performance: result.performance,
}) + '\n');
```

---

## Complete Logging Pipeline Example

The following example shows a complete pipeline that combines all concepts into a working activity logger.

```typescript
import * as H from '@vladmandic/human';
import fs from 'fs';

const config: Partial<H.Config> = {
  backend: 'webgl',
  modelBasePath: 'https://vladmandic.github.io/human-models/models',
  face: {
    enabled: true,
    detector: { maxDetected: 10, minConfidence: 0.3 },
    mesh: { enabled: true },
    emotion: { enabled: true },
  },
  body: { enabled: true, maxDetected: 6, modelPath: 'movenet-multipose.json' },
  hand: { enabled: true },
  gesture: { enabled: true },
  object: { enabled: false },
};

interface WorkerSession {
  workerId: string;
  label: string;
  state: 'present' | 'absent';
  consecutiveMisses: number;
  lastSeen: number;
  currentPersonId: number | null;
}

class ActivityLogger {
  private human: H.Human;
  private workers: Map<string, WorkerSession> = {};
  private logStream: fs.WriteStream;
  private fps = { frames: 0, start: 0, detectFPS: 0, lastDetect: 0 };
  private prevTimestamp = 0;

  constructor(private video: HTMLVideoElement, private workerIds: string[]) {
    this.human = new H.Human(config);
    this.logStream = fs.createWriteStream('activity-log.jsonl', { flags: 'a' });
  }

  async init() {
    await this.human.load();
    await this.human.warmup();

    for (const id of this.workerIds) {
      this.workers.set(id, {
        workerId: id,
        label: `Worker ${id}`,
        state: 'absent',
        consecutiveMisses: 0,
        lastSeen: 0,
        currentPersonId: null,
      });
    }
  }

  async processFrame() {
    if (this.video.paused || this.video.readyState < 2) return;

    const detectStart = performance.now();
    await this.human.detect(this.video, config);
    const detectLatency = performance.now() - detectStart;

    const result = this.human.result;
    this.updateFPS();

    const matchedPersonIds = new Set(result.persons.map(p => p.id));

    for (const [workerId, session] of this.workers) {
      const isMatched = session.currentPersonId !== null &&
        matchedPersonIds.has(session.currentPersonId);

      if (isMatched) {
        session.consecutiveMisses = 0;
        session.lastSeen = result.timestamp;

        if (session.state === 'absent') {
          session.state = 'present';
          this.logEvent(workerId, result.timestamp, 'enter');
        }

        const person = result.persons.find(p => p.id === session.currentPersonId);
        if (person) {
          this.logDetection(workerId, result, person, detectLatency);
        }

        session.currentPersonId = person?.id ?? null;
      } else {
        session.consecutiveMisses++;

        if (session.state === 'present' && session.consecutiveMisses >= 30) {
          session.state = 'absent';
          this.logEvent(workerId, result.timestamp, 'exit', {
            lastSeen: session.lastSeen,
          });
          session.currentPersonId = null;
        }
      }
    }

    // Log frame-level performance
    this.logPerformance(result, detectLatency);
  }

  private logDetection(workerId: string, result: H.Result, person: H.PersonResult, latency: number) {
    const primaryEmotion = person.face?.emotion
      ?.sort((a, b) => b.score - a.score)?.[0];

    const gestures = person.gestures.map(g => {
      if ('face' in g) return `face:${g.gesture}`;
      if ('body' in g) return `body:${g.gesture}`;
      if ('hand' in g) return `hand:${g.gesture}`;
      if ('iris' in g) return `iris:${g.gesture}`;
      return `unknown:${g.gesture}`;
    });

    const entry = {
      ts: result.timestamp,
      workerId,
      type: 'detection',
      personId: person.id,
      face: person.face ? {
        score: person.face.score,
        age: person.face.age,
        gender: person.face.gender,
        emotion: primaryEmotion ? {
          label: primaryEmotion.emotion,
          score: primaryEmotion.score,
        } : null,
        box: person.face.box,
      } : null,
      body: person.body ? {
        score: person.body.score,
        box: person.body.box,
        keypointCount: person.body.keypoints.length,
        keypoints: person.body.keypoints.map(kp => ({
          part: kp.part,
          x: kp.position[0],
          y: kp.position[1],
          z: kp.position[2],
          score: kp.score,
        })),
      } : null,
      hands: {
        left: person.hands.left ? {
          label: person.hands.left.label,
          score: person.hands.left.score,
          box: person.hands.left.box,
        } : null,
        right: person.hands.right ? {
          label: person.hands.right.label,
          score: person.hands.right.score,
          box: person.hands.right.box,
        } : null,
      },
      gestures,
      detectLatencyMs: latency,
    };

    this.logStream.write(JSON.stringify(entry) + '\n');
  }

  private logEvent(workerId: string, timestamp: number, event: 'enter' | 'exit', data?: any) {
    this.logStream.write(JSON.stringify({
      ts: timestamp,
      workerId,
      type: 'event',
      event,
      ...data,
    }) + '\n');
  }

  private logPerformance(result: H.Result, detectLatency: number) {
    this.logStream.write(JSON.stringify({
      ts: result.timestamp,
      type: 'performance',
      detectLatencyMs: detectLatency,
      fps: this.fps.detectFPS,
      performance: result.performance,
    }) + '\n');
  }

  private updateFPS() {
    this.fps.frames++;
    const now = this.human.now();
    if (this.fps.start === 0) this.fps.start = now;
    this.fps.detectFPS = Math.round(1000 * this.fps.frames / (now - this.fps.start)) / 1000;
  }

  start() {
    const loop = async () => {
      await this.processFrame();
      requestAnimationFrame(loop);
    };
    loop();
  }
}

// Usage
const video = document.querySelector('video') as HTMLVideoElement;
const logger = new ActivityLogger(video, ['worker-1', 'worker-2', 'worker-3']);
await logger.init();
logger.start();
```

---

## Production Tips

### Filtering

Enable the Human library's built-in filtering to reduce noise:

```typescript
const config: Partial<H.Config> = {
  filter: {
    enabled: true,
    equalization: false,
    flip: false,
  },
  face: {
    detector: { minConfidence: 0.5 },  // Filter low-confidence detections
    emotion: { minConfidence: 0.3 },
  },
  body: {
    minConfidence: 0.3,
  },
};
```

### Batching Writes

Buffer log entries and write in batches to reduce I/O overhead:

```typescript
class BatchWriter {
  private buffer: string[] = [];
  private readonly BATCH_SIZE = 100;
  private flushInterval: NodeJS.Timeout;

  constructor(private stream: fs.WriteStream, flushMs = 5000) {
    this.flushInterval = setInterval(() => this.flush(), flushMs);
  }

  write(entry: string) {
    this.buffer.push(entry);
    if (this.buffer.length >= this.BATCH_SIZE) {
      this.flush();
    }
  }

  flush() {
    if (this.buffer.length > 0) {
      this.stream.write(this.buffer.join('') + '\n');
      this.buffer = [];
    }
  }

  close() {
    clearInterval(this.flushInterval);
    this.flush();
    this.stream.end();
  }
}
```

### Error Handling

Wrap detection in error handling to prevent pipeline crashes:

```typescript
async function safeDetect(human: H.Human, video: HTMLVideoElement, config: Partial<H.Config>): Promise<H.Result | null> {
  try {
    await human.detect(video, config);
    if (human.result.error) {
      console.error('Detection error:', human.result.error);
      return null;
    }
    return human.result;
  } catch (err) {
    console.error('Detection failed:', err);
    return null;
  }
}
```

### Worker Identity Management

Assign stable IDs to workers using face embeddings when available:

```typescript
const knownEmbeddings: Map<string, number[]> = new Map();

function identifyWorker(face: FaceResult): string | null {
  if (!face.embedding || knownEmbeddings.size === 0) return null;

  let bestMatch = '';
  let bestDistance = Infinity;

  for (const [workerId, knownEmbedding] of knownEmbeddings) {
    const distance = cosineDistance(face.embedding, knownEmbedding);
    if (distance < bestDistance && distance < 0.5) {
      bestDistance = distance;
      bestMatch = workerId;
    }
  }

  return bestMatch || null;
}

function cosineDistance(a: number[], b: number[]): number {
  let dot = 0, normA = 0, normB = 0;
  for (let i = 0; i < a.length; i++) {
    dot += a[i] * b[i];
    normA += a[i] * a[i];
    normB += b[i] * b[i];
  }
  return 1 - dot / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

### Graceful Shutdown

Ensure all buffered data is flushed on shutdown:

```typescript
async function shutdown(logger: ActivityLogger) {
  logger['batchWriter'].close();

  // Log final state for all workers
  for (const [workerId, session] of logger['workers']) {
    if (session.state === 'present') {
      logger.logEvent(workerId, Date.now(), 'exit', {
        reason: 'shutdown',
      });
    }
  }

  logger['logStream'].end();
  await human.tf.disposeVariables();
}
```

### Configuration for Production

Recommended baseline configuration for worker monitoring:

```typescript
const productionConfig: Partial<H.Config> = {
  debug: false,
  async: true,
  backend: 'webgl',
  filter: { enabled: true },
  face: {
    enabled: true,
    detector: { maxDetected: 10, minConfidence: 0.4, rotation: false },
    mesh: { enabled: true },
    emotion: { enabled: true, minConfidence: 0.4 },
    iris: { enabled: false },
    description: { enabled: false },
    antispoof: { enabled: false },
    liveness: { enabled: false },
  },
  body: { enabled: true, maxDetected: 6, modelPath: 'movenet-multipose.json' },
  hand: { enabled: true },
  gesture: { enabled: true },
  object: { enabled: false },
  segmentation: { enabled: false },
};
```

---

## Reference

- `src/result.ts` - Result, PersonResult, FaceResult, BodyResult, HandResult, GestureResult types
- `src/util/persons.ts` - Person combining logic (`join()` function)
- `src/util/interpolate.ts` - Temporal smoothing (`calc()` function)
- `demo/tracker/index.ts` - Tracking example with configurable matcher
- `demo/nodejs/node-event.js` - Event-based processing in Node.js
