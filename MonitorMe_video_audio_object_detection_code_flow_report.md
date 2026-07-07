# MonitorMe 0.4.2-p20 — Video/Audio Object-Detection Use-Case and Source-Code Flow Study

**Repository reviewed:** `MonitorMe-0.4.2-p20.tar(1).gz`  
**Git head:** `9f169b2`  
**Study scope:** all listed Python application modules, tests, and the C++/CUDA `native/node1_non_llm_gpu_inference_lab` code that supplements the camera/evidence use case.

---

## 1. Executive conclusion

MonitorMe already contains a strong **local, evidence-first camera analytics pipeline**, but the exact requested use case—**real-time video and audio recording that starts because an object was detected**—is not yet implemented end to end.

What exists today is:

```text
C922 frames
  -> bounded OpenCV/V4L2 read loop
  -> frame-difference motion gate
  -> write JPEG keyframe
  -> create parent motion_detected event
  -> optionally run YOLO ONNX on that motion frame
  -> create child object_detected events
  -> optionally write annotated JPEG
  -> optionally run local Qwen keyframe VLM
  -> optionally write a recent-frame JPEG bundle for SmolVLM2
  -> optionally profile frame workload through the native C++/CUDA sidecar
  -> after capture, optionally fingerprint/deduplicate/index evidence
  -> SQLite/API/dashboard/reporting/retention
```

What does **not** exist today:

```text
object_detected
  -> start encoded MP4/MKV recording
  -> include pre-roll frames
  -> continue post-roll after object disappears
  -> capture microphone/AudioBox PCM continuously
  -> synchronize audio and video timestamps
  -> encode/mux audio + video
  -> close and persist a playable A/V clip
```

The current SmolVLM2 “short clip” is **not a video container**. It is a directory containing recent JPEG frames plus `clip_manifest.json`. The current AudioBox path is **signal analysis only** over synthetic or externally supplied float32 samples; it does not open ALSA/PulseAudio/PipeWire and is not connected to the live camera runner.

This distinction is central when presenting the system:

- **Implemented production path:** motion-triggered still evidence plus optional object facts.
- **Implemented native acceleration lab:** deterministic CPU/CUDA workloads that can support future real-time media processing.
- **Implemented post-capture evidence plane:** fingerprints, deduplication, storage planning, key moments, APIs, dashboards, retention, and rebuild.
- **Target next-stage use case:** object-confirmed, pre-roll/post-roll, synchronized A/V recording.

---

## 2. Source-code scope and responsibility split

The codebase has three major planes.

### 2.1 Python application and evidence plane

```text
monitor_me/
  camera discovery and capture
  deterministic motion detection
  YOLO ONNX object inference
  normalized event/artifact persistence
  evidence overlays
  deterministic policy and summaries
  optional local Gemma/Qwen/SmolVLM2 clients
  grounded assistant and reports
  evidence-index persistence, API, dashboard, retention, rebuild
  Python-to-native sidecar bridge
```

### 2.2 Native deterministic workload plane

```text
native/node1_non_llm_gpu_inference_lab/
  C++17 CPU reference implementations
  optional CUDA implementations
  shared validation, timing, JSON and type contracts
  synthetic/file-backed command-line modes
  CPU-vs-CUDA parity validation
  Nsight Compute profiling orchestration
```

### 2.3 Test and validation plane

```text
tests/
  camera and motion event normalization
  real/fake YOLO behavior
  assistant grounding and anti-hallucination
  VLM structured-output validation
  evidence pack/report generation
  native phase 0–20 functional and parity checks
  API/dashboard/Grafana/retention/rebuild behavior
```

The architecture deliberately keeps semantic camera claims in Python and uses the native sidecar for **facts-only workload metadata and deterministic signal/image operations**. This prevents a CUDA workload result such as “dense path” from being confused with a semantic claim such as “person detected.”

---

## 3. Use cases identified from the source

## 3.1 Use case A — Local bounded camera capture

**Status:** implemented.

The operator starts a bounded capture through the CLI or FastAPI. The source is normally the Logitech C922 at `/dev/video0`. OpenCV uses the Linux V4L2 backend and requests width, height, frame rate, and MJPG fourcc.

Primary code:

- `monitor_me/camera_devices.py`
- `monitor_me/local_capture.py::OpenCVFrameSource`
- `monitor_me/cli.py::cmd_capture_run`
- `monitor_me/routes.py::POST /camera/capture/run`

The run ends when either:

- `duration_sec` expires,
- `max_frames` is reached, or
- the frame source returns failure/end-of-stream.

This is a bounded evidence acquisition session, not a continuous NVR loop.

---

## 3.2 Use case B — Motion-triggered local evidence capture

**Status:** implemented.

Every frame passes through a deterministic motion gate:

```text
BGR frame
  -> grayscale
  -> Gaussian blur 21x21
  -> absdiff(previous, current)
  -> binary threshold
  -> dilation x2
  -> changed-pixel count
  -> changed ratio and percentage score
  -> union bounding box of external contours
  -> compare score against motion_threshold
```

When motion is accepted and the minimum event gap has elapsed, MonitorMe writes a raw JPEG keyframe and stores:

- SHA-256,
- artifact row,
- `motion_detected` event,
- normalized motion bounding box,
- changed ratio,
- camera format metadata,
- session/frame/timestamp references.

Primary code: `monitor_me/local_capture.py:221-270` and `459-520`.

Important behavior:

- The first frame initializes the previous-frame state and cannot emit motion.
- The gate compares each frame to the immediately previous blurred frame, not to a long-term background model.
- `min_event_gap_sec` throttles the entire trigger path. During the gap, YOLO is not run because YOLO is downstream of the accepted motion trigger.
- The event “confidence” for motion is a percentage-like score from `0` to `100`, whereas object confidence is normally `0` to `1`. This is usable but should be documented or normalized to avoid analytics confusion.

---

## 3.3 Use case C — Motion-gated YOLO object detection

**Status:** implemented and optional.

YOLO is not continuously run on every camera frame. It executes only after a motion event has already caused a keyframe to be stored. This reduces inference load and ensures the motion evidence survives even if the model is missing or fails.

```text
accepted motion frame
  -> stored raw keyframe and parent motion event
  -> YoloOnnxDetector.detect(frame)
       -> letterbox to model shape
       -> BGR to RGB
       -> float32 normalization
       -> NCHW tensor
       -> ONNX Runtime session
       -> support [1,84,N], [1,N,84], [1,N,85], or decoded [1,N,6]
       -> confidence filtering
       -> map boxes back to original frame
       -> normalized xyxy boxes
       -> class-aware NMS
  -> one child object_detected event per kept detection
```

Primary code:

- `monitor_me/yolo_onnx.py`
- `monitor_me/yolo_client.py`
- `monitor_me/local_capture.py:1265-1369`
- `monitor_me/detector_health.py`

Semantic normalization:

```text
car, truck, bus, motorcycle, bicycle, train, boat
  -> canonical label "vehicle"
  -> exact COCO class retained in attrs.raw_label
```

This makes assistant queries such as “vehicle” useful while preserving the original model output.

Fault behavior is deliberately conservative:

```text
detector disabled/missing/load failure/runtime failure
  -> keep motion event and raw keyframe
  -> write audit warning
  -> emit zero object events
  -> never fabricate a person/vehicle/object label
```

---

## 3.4 Use case D — Person/object policy recommendation

**Status:** implemented as a deterministic recommendation, not as a recording controller.

`capture_policy.py` evaluates the strict event contract. If a person detection has confidence at least `0.60`, the policy returns:

```text
decision = allow
action = request_capture_review
severity = review
duration_sec = 90
```

Other object evidence returns `record_evidence_only`; motion without objects returns `record_motion_only`.

Crucial limitation: `duration_sec=90` is metadata in a policy decision. No code consumes it to start or extend a 90-second video/audio recording. Therefore this module expresses the intended business action but does not implement the media action.

---

## 3.5 Use case E — Annotated evidence image

**Status:** implemented.

When YOLO returns detections, MonitorMe creates a derived JPEG containing:

- normalized box converted to pixel coordinates,
- label,
- confidence,
- event identifier,
- model identifier,
- title with camera/frame/session.

The raw keyframe remains unchanged. The overlay is a separate `annotated_keyframe` artifact with its own hash.

Primary code: `monitor_me/overlays.py` and `local_capture.py:556-598,1457-1484`.

The native Phase 6 overlay-heavy path is a separate synthetic/file-independent acceleration lab and is not the implementation used by this Python overlay function.

---

## 3.6 Use case F — Optional Qwen keyframe analysis

**Status:** implemented as an optional local experiment.

After the keyframe and normalized event exist, `KeyframeVLMAnalysisService` can send the stored image to a loopback OpenAI-compatible Qwen endpoint. It uses a constrained JSON contract and validates:

- event references,
- artifact references,
- allowed observations,
- absence of unsupported identity/intent claims.

The VLM cannot create the original motion/object event. It is a post-trigger enrichment layer.

Primary code:

- `monitor_me/keyframe_vlm.py`
- `monitor_me/vlm_client.py`
- migration `003_node1_assistant_v03.sql`

---

## 3.7 Use case G — Optional SmolVLM2 short-frame-bundle experiment

**Status:** implemented, but it is not encoded video.

A bounded `deque` keeps the most recent N frames while capture runs. On an accepted motion trigger, the runner writes those frames as JPEG files and writes a JSON manifest.

```text
recent_frames deque
  -> clip_<trigger frame>/
       frame_000101.jpg
       frame_000102.jpg
       ...
       clip_manifest.json
  -> short_clip_frame artifacts
  -> short_clip_manifest artifact
  -> constrained SmolVLM2 experiment
```

Characteristics:

- primarily pre-trigger/recent context,
- no post-trigger collection phase,
- no frame timing stream beyond IDs/manifest metadata,
- no codec,
- no MP4/MKV,
- no audio,
- no A/V synchronization.

Primary code:

- `local_capture.py:452,470-471,625-645,1371-1455`
- `short_clip_vlm.py`
- `smolvlm2_client.py`
- migration `004_node1_assistant_v04.sql`

---

## 3.8 Use case H — Non-LLM workload-shape profiling after a trigger

**Status:** partially integrated.

When `gpu_lab_enabled` is true, the capture runner sends the previous and current frames to `Node1NonLLMGpuLabRunner` after a motion trigger.

```text
BGR previous/current frames
  -> Python NumPy grayscale conversion
  -> temporary prev.gray + curr.gray files
  -> native subprocess --mode analyze-raw-gray
  -> CPU and optionally CUDA frame-difference tile mask
  -> active tile count
  -> sparse | mixed | dense route classification
  -> gpu_workload_profiled child event
```

Stored facts include:

- tile mask,
- low/high mask occupancy,
- changed pixels and ratio,
- active tiles,
- route decision,
- CPU/CUDA timings and backend availability.

Important integration boundary:

The live path currently executes the **tile-mask classifier only**. It does not automatically dispatch to the specialized Phase 3 sparse ROI, Phase 4 mixed region, or Phase 5 dense full-frame processing functions after choosing a route. Those specialized paths are exposed through native synthetic CLI modes and Python lab commands.

Also, the bridge currently incurs per-trigger overhead from:

- NumPy conversion,
- temporary-file writes,
- subprocess creation,
- native process startup,
- CUDA allocation/copy/free inside the native function.

This is valid for evidence/profiling but not yet an optimal real-time in-process path.

---

## 3.9 Use case I — Audio energy and AudioBox signal analysis

**Status:** native lab implemented; live audio capture not integrated.

Two native audio paths exist.

### Basic audio energy

```text
float32 samples
  -> fixed-size windows
  -> RMS per window
  -> threshold
  -> 32-bit active-window event mask
```

### AudioBox Phase 7

```text
primary float32 samples + reference float32 samples
  -> per-window RMS
  -> per-window peak
  -> silence mask
  -> onset mask
  -> normalized cross-correlation for lag [-max_lag,+max_lag]
  -> best sync drift in samples and milliseconds
```

CPU and CUDA implementations exist and have parity tests. However:

- no Python module opens an audio input device,
- no ALSA/PulseAudio/PipeWire/GStreamer/FFmpeg audio ingestion exists,
- no audio ring buffer exists,
- no captured samples are passed from `LocalCameraCaptureRunner` to AudioBox,
- capture-run evidence rows set `audio_score` explicitly to `0.0`,
- no audio artifact or media track is written.

The AudioBox code is therefore a ready analytical building block for a future A/V recorder, not an existing live recorder.

---

## 3.10 Use case J — Post-capture evidence fingerprinting and deduplication

**Status:** implemented and optional.

After the bounded camera loop finishes, the capture runner can create an evidence CSV from the stored motion keyframes.

For each keyframe, Python locally decodes the JPEG and computes:

- average hash,
- difference hash,
- combined 64-bit fingerprint,
- 16-bin luminance histogram,
- decoded dimensions.

It also supplies:

- motion score,
- changed pixels,
- derived lighting delta,
- artifact size,
- timeline start/duration.

`audio_score` is currently zero.

The native evidence pipeline then performs:

```text
CSV manifest
  -> deterministic storage batch plan
  -> use supplied decoded-media fingerprint or metadata-synthetic fallback
  -> Hamming-distance dedup union/find groups
  -> dedup-aware key-moment selection
  -> timeline and throughput metrics
  -> safety-contract validation
  -> JSON profile
```

The Python DB layer persists normalized rows for:

- profiles,
- fingerprints,
- duplicate groups,
- key moments.

This is a useful foundation for real A/V clips because the same manifest schema already contains `path`, `duration_ms`, `bytes`, `motion_score`, and `audio_score`. At present, the integrated rows refer to JPEG keyframes, while synthetic tests use `.mkv` names only as metadata examples.

---

## 3.11 Use case K — Grounded operator assistant, evidence packs, reports, dashboards

**Status:** implemented.

The system can query normalized events and produce answers that expose event/session/frame references. FactGuard blocks unsupported identity, weapon, intent, or suspiciousness claims unless matching normalized labels exist.

The evidence plane also supports:

- deterministic and optional local Gemma/MAX summaries,
- strict event contracts,
- evidence packs,
- incident reports,
- operator feedback,
- read-only operator dashboard,
- chart-ready JSON and inline SVG,
- Prometheus exposition,
- Grafana dashboard JSON,
- evidence-index retention/scheduling,
- evidence-index rebuild from retained profile artifacts.

These components are directly reusable once real video/audio clip artifacts are added.

---

# 4. Current end-to-end architecture text view

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ Operator entry points                                                    │
│                                                                          │
│ CLI: python -m monitor_me.cli capture-run ...                            │
│ API: POST /camera/capture/run                                            │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │ LocalCaptureConfig
                                   v
┌──────────────────────────────────────────────────────────────────────────┐
│ Policy, registration and session setup                                   │
│                                                                          │
│ allow_node1_local_camera()                                                │
│ register camera/model records                                             │
│ create capture_sessions row                                               │
│ create data/captures/<session>/                                           │
│   keyframes/ overlays/ [short_clips/]                                     │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   v
┌──────────────────────────────────────────────────────────────────────────┐
│ OpenCV/V4L2 camera loop                                                   │
│                                                                          │
│ cv2.VideoCapture(/dev/video0, CAP_V4L2)                                  │
│ set MJPG + width + height + fps                                           │
│ read BGR frames until deadline/max_frames                                 │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   v
┌──────────────────────────────────────────────────────────────────────────┐
│ Deterministic motion gate                                                 │
│                                                                          │
│ gray -> GaussianBlur -> absdiff -> threshold -> dilate                    │
│ -> changed ratio -> motion bbox -> min-event-gap gate                     │
└───────────────┬───────────────────────────────────────────────┬──────────┘
                │ no accepted motion                            │ accepted
                │                                               v
                │                           ┌────────────────────────────────┐
                │                           │ Raw evidence first             │
                │                           │ write frame_N.jpg              │
                │                           │ hash + artifact row            │
                │                           │ motion_detected parent event   │
                │                           └──────────────┬─────────────────┘
                │                                          v
                │                           ┌────────────────────────────────┐
                │                           │ Optional YOLO ONNX             │
                │                           │ letterbox -> inference -> NMS  │
                │                           │ child object_detected rows     │
                │                           └───────┬───────────────┬────────┘
                │                                   │               │
                │                                   │ detections    │ previous frame available
                │                                   v               v
                │                        ┌───────────────────┐  ┌──────────────────────┐
                │                        │ annotated JPEG    │  │ native tile profile  │
                │                        │ derived artifact  │  │ sparse/mixed/dense   │
                │                        └───────────────────┘  │ routing facts event  │
                │                                               └──────────────────────┘
                │
                │                           ┌────────────────────────────────┐
                │                           │ Event enrichment               │
                │                           │ strict event contract          │
                │                           │ deterministic policy           │
                │                           │ deterministic/Gemma summary    │
                │                           │ optional Qwen keyframe VLM     │
                │                           │ optional JPEG frame bundle     │
                │                           │ + SmolVLM2 experiment          │
                │                           └────────────────────────────────┘
                │
                └──────────────────────────────────────────────────────────> loop

After loop:

manifest frames
  -> optional Python decoded-keyframe fingerprints
  -> evidence CSV
  -> native storage/fingerprint/dedup/key-moment/safety pipeline
  -> JSON profile + evidence_pipeline_indexed event
  -> normalized SQLite evidence index
  -> capture manifest artifact
  -> complete capture session
  -> API / assistant / evidence pack / report / dashboard / Grafana
```

---

# 5. Exact live code flow

## 5.1 Configuration and entry

### CLI path

```text
cli.py::build_parser()
  -> capture-run options
cli.py::cmd_capture_run(args)
  -> LocalCaptureConfig(...)
  -> LocalCameraCaptureRunner(db, config).run()
  -> JSON result and exit status
```

The CLI exposes the complete feature set, including GPU-lab and evidence-pipeline options.

### API path

```text
routes.py::CaptureRequest
  -> POST /camera/capture/run
  -> LocalCaptureConfig(**req.model_dump())
  -> LocalCameraCaptureRunner.run()
```

Current API gap: `CaptureRequest` includes camera, YOLO, overlays, Qwen, and SmolVLM2 fields, but not the GPU-lab or post-capture evidence-pipeline fields that exist in `LocalCaptureConfig` and CLI. Therefore the API cannot currently enable those deeper native paths without code changes.

---

## 5.2 Session initialization

`LocalCameraCaptureRunner.run()` performs:

```text
1. create local-camera allow policy
2. register default models
3. upsert camera
4. upsert YOLO model metadata
5. upsert Qwen VLM metadata
6. upsert SmolVLM2 metadata
7. upsert native workload-profiler metadata
8. upsert evidence-pipeline metadata
9. create capture session status=running
10. create session directory tree
11. construct motion gate
12. load detector if enabled
13. construct optional enrichment/native services
```

All optional model paths are disabled by default, preserving a working motion-only baseline.

---

## 5.3 Frame loop

Per frame:

```text
read frame
increment frame_id / frames_seen
append a copy to recent-frame deque if SmolVLM2 enabled
motion = MotionGate.evaluate(frame)
save current frame as previous frame for next native comparison
should_emit = motion.motion AND min-event-gap elapsed
```

On accepted motion:

```text
write JPEG keyframe
calculate SHA-256
insert keyframe artifact
insert parent motion event
run optional YOLO
insert child object events
run optional native tile workload profile
write optional annotated keyframe
create strict event contracts and summaries
run optional keyframe VLM
write optional JPEG frame bundle and run short-clip VLM
append complete trigger metadata to in-memory manifest_frames
```

The order is intentional: raw evidence and parent motion event are committed before optional inference/enrichment.

---

## 5.4 End-of-run finalization

```text
close camera in finally block
build capture manifest structure
if evidence pipeline enabled:
    create CSV rows from motion keyframes
    invoke native evidence pipeline
    store CSV + JSON artifacts
    create evidence_pipeline_indexed event
    persist normalized evidence index
write final manifest.json
insert capture_manifest artifact
update session completed/failed
write audit record
return LocalCaptureResult
```

---

# 6. Parent/child data model for object detection

```text
capture_sessions.session_id
  |
  +-- capture_artifacts: raw keyframe JPEG
  |
  +-- events: motion_detected                  parent
        event_id = evt_motion
        frame_id = N
        artifact_id = raw keyframe
        confidence = motion percentage
        bbox = motion union bbox
        |
        +-- events: object_detected            child
        |     parent_event_id = evt_motion
        |     label = person | vehicle | ...
        |     confidence = YOLO score
        |     bbox = object bbox
        |     model_id = YOLO model
        |     artifact_id = same raw keyframe
        |
        +-- events: gpu_workload_profiled      child
              parent_event_id = evt_motion
              label = sparse | mixed | dense
              confidence = changed ratio
              attrs = tile mask/timing/backend
```

Derived records can then reference the event/session:

```text
event_contracts
assistant_summaries
vlm_keyframe_analyses
smolvlm2_clip_experiments
evidence_packs
incident_reports
event_feedback
audit_log
```

---

# 7. Why the present code is not yet object-triggered video/audio recording

The source search shows no implementation of:

- `cv2.VideoWriter` for output,
- FFmpeg subprocess recording,
- GStreamer recording/muxing pipeline,
- MP4/MKV writer lifecycle,
- PCM/WAV writer,
- ALSA/PulseAudio/PipeWire microphone capture,
- pre-roll video ring buffer,
- audio ring buffer,
- post-roll collection timer,
- A/V PTS/DTS synchronization,
- object-presence state machine,
- codec close/finalize handling.

The only `VideoWriter` reference is `cv2.VideoWriter_fourcc`, used to request the **camera input format**, not to write a video file.

The `.mkv` paths in storage/evidence synthetic data are illustrative manifest strings. The code does not create those files.

The policy action `request_capture_review` and duration `90` are not consumed by a recorder.

The SmolVLM2 frame bundle is a set of JPEGs and JSON, not a playable clip.

The evidence pipeline’s integrated `audio_score` is zero.

Therefore the accurate current product statement is:

> MonitorMe performs local bounded camera acquisition, motion-triggered JPEG evidence capture, and optional motion-gated object detection. It contains native image/audio workload primitives and metadata-oriented clip planning, but it does not yet encode synchronized object-triggered video/audio recordings.

---

# 8. Target code flow for true object-triggered A/V recording

The current components can be evolved into the following architecture.

```text
                     CONTINUOUS INGEST PLANE
┌─────────────────────────────────────────────────────────────────────────┐
│ Video capture thread                       Audio capture thread          │
│ C922/V4L2 frames                           AudioBox/ALSA PCM             │
│ monotonic timestamp + frame sequence       monotonic timestamp          │
│        |                                          |                      │
│        v                                          v                      │
│ video pre-roll ring buffer                 audio pre-roll ring buffer    │
└───────────────┬──────────────────────────────────┬───────────────────────┘
                │                                  │
                v                                  │
     FAST TRIGGER / SEMANTIC PLANE                 │
┌────────────────────────────────────────────┐     │
│ frame diff / native tile mask              │     │
│  -> sparse/mixed/dense route               │     │
│  -> optional ISP                           │     │
│  -> YOLO only on selected frames/ROIs      │     │
│  -> temporal confirmation / class policy   │     │
└────────────────────┬───────────────────────┘     │
                     │ confirmed object             │
                     v                              v
              RECORDING COORDINATOR / STATE MACHINE
┌─────────────────────────────────────────────────────────────────────────┐
│ IDLE                                                                    │
│   -> PREPARING on object confirmation                                    │
│   -> RECORDING                                                           │
│        flush video and audio pre-roll                                    │
│        append live frames and samples                                    │
│        extend stop deadline on matching detections                       │
│        collect AudioBox RMS/onset/drift facts                            │
│   -> FINALIZING after post-roll expires                                  │
│        close encoder/muxer atomically                                    │
│        hash media                                                        │
│        write clip manifest                                               │
│   -> IDLE                                                                │
└────────────────────┬────────────────────────────────────────────────────┘
                     v
                 MEDIA ARTIFACT PLANE
┌─────────────────────────────────────────────────────────────────────────┐
│ clip_<event>.mkv or .mp4                                                 │
│ optional raw/derived keyframes                                           │
│ audio/video timing manifest                                              │
│ overlay/thumbnail                                                        │
│ SHA-256 and normalized DB artifact                                       │
│ parent motion + child object events + recording_started/completed events │
└────────────────────┬────────────────────────────────────────────────────┘
                     v
                 EVIDENCE PIPELINE
┌─────────────────────────────────────────────────────────────────────────┐
│ storage batch plan -> fingerprints -> dedup -> key moments -> retention  │
│ assistant/report/dashboard/Grafana                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

## 8.1 Recommended state transitions

```text
IDLE
  motion but no accepted object -> retain keyframe evidence only
  accepted object -> RECORDING

RECORDING
  same configured object continues -> extend post-roll deadline
  additional object class appears -> link child event to same clip
  no configured object -> continue until post-roll deadline
  duration/max-size limit -> rotate/finalize clip

FINALIZING
  flush encoder
  close muxer
  hash file
  persist artifact and manifest
  run evidence indexing asynchronously or after closure
  return to IDLE
```

## 8.2 Reuse map from current code

| Needed recorder capability | Existing reusable component | Required addition |
|---|---|---|
| Camera discovery/input | `camera_devices.py`, `OpenCVFrameSource` | long-running ingest thread and timestamp envelope |
| Cheap trigger | `MotionGate`, native tile mask | persistent native context and configurable frame sampling |
| Object confirmation | `YoloOnnxDetector` | class allowlist, temporal debounce, tracking/object persistence |
| Pre-roll | SmolVLM2 recent-frame deque concept | time-based encoded/raw ring buffer with memory limits |
| Post-roll | policy duration concept | recorder deadline/state machine |
| Video writing | none | encoder/muxer abstraction |
| Audio capture | none | ALSA/PulseAudio/PipeWire input abstraction |
| Audio facts | native basic audio + AudioBox | feed live timestamped PCM blocks |
| A/V drift | AudioBox correlation | establish common clock/reference channel and correction policy |
| Overlays/thumbnails | Python overlay, native Phase 6 | integrate native output or encode sidecar assets |
| Clip batching | native Phase 8 | point manifest entries at actual closed clips |
| Fingerprint/dedup | Phase 9–12 evidence index | add representative video-frame/audio fingerprints as desired |
| Reporting | assistant/evidence pack/report/dashboard | include playable clip artifact and A/V metrics |

---

# 9. Python source-code study by module

## 9.1 `monitor_me/local_capture.py`

This is the principal use-case orchestrator.

### Main responsibilities

- capture configuration,
- real OpenCV/V4L2 source,
- test iterable source,
- frame-difference motion gate,
- session lifecycle,
- keyframe and artifact writing,
- parent/child events,
- optional detector/VLM/native paths,
- manifest generation,
- post-capture evidence pipeline.

### Important design strengths

- raw evidence is written before optional inference;
- detector failure cannot erase motion evidence;
- every artifact is hashed;
- all enrichment is opt-in;
- a `finally` block releases the camera;
- privacy and facts-only semantics are recorded in manifests/events;
- frame source and detector protocols make testing possible without hardware/model files.

### Important gaps

- `write_every_motion_keyframe` exists in configuration but is not used in the run loop;
- no output video writer;
- no audio source;
- no post-trigger frame collection for short clips;
- no object-triggered recorder state machine;
- GPU bridge uses per-trigger temp files/subprocess;
- specialized sparse/mixed/dense native paths are not automatically dispatched;
- motion confidence scale differs from object confidence scale;
- evidence pipeline’s audio score is hard-coded to zero.

---

## 9.2 `monitor_me/yolo_onnx.py`

Implements the ONNX Runtime detector.

### Processing stages

```text
model initialization
  -> import onnxruntime
  -> validate model file
  -> prefer CUDAExecutionProvider if available
  -> resolve NCHW input dimensions

detect(frame)
  -> letterbox with value 114
  -> BGR to RGB
  -> normalize /255
  -> transpose HWC to NCHW and add batch
  -> session.run
  -> normalize output layout to rows
  -> decode boxes/classes/confidence
  -> map letterboxed coordinates back to original frame
  -> clip and normalize coordinates
  -> class-aware NMS
  -> cap at max_detections
```

The `ObjectDetection` dataclass carries the normalized semantic contract used by DB insertion.

---

## 9.3 `monitor_me/yolo_client.py`

A thin interface layer around the detector. It provides a small client abstraction so higher-level code can depend on `detect(frame)` rather than ONNX Runtime details.

---

## 9.4 `monitor_me/camera_devices.py`

Enumerates `/dev/video*`, identifies character devices, reports major/minor numbers, and optionally runs bounded `v4l2-ctl --list-formats-ext`. It also warns that C922 devices may expose multiple video nodes.

This should remain the preflight source-selection layer for a future recorder.

---

## 9.5 `monitor_me/detector_health.py`

Validates the detector deployment without opening the camera:

- model existence and size,
- SHA-256 and optional expected hash,
- ONNX Runtime installation/version,
- available/selected providers,
- optional session load,
- input/output tensor metadata.

It is suitable for a recorder readiness endpoint because it separates model readiness from media acquisition.

---

## 9.6 `monitor_me/policy.py`

A simple local permission decision for Node1 camera capture. It records that the path is local and external upload is false. It is an authorization/intent boundary rather than an object action policy.

---

## 9.7 `monitor_me/capture_policy.py`

Converts normalized event contracts into deterministic operator actions. It never lets an LLM decide whether an action should occur.

For a true recorder, this is the natural module to evolve from “recommend action” to a validated `RecordingDirective`, while keeping actual device control in a dedicated recorder service.

---

## 9.8 `monitor_me/event_contract.py`

Builds the strict facts passed to policy and summary layers:

- requested/current event,
- parent motion event,
- object children,
- detector metadata,
- artifacts,
- session metadata,
- explicit privacy restrictions.

One consideration: it currently includes all session artifacts rather than only artifacts directly linked to the selected event. For long multi-clip sessions, event-to-artifact linkage should be made explicit.

---

## 9.9 `monitor_me/assistant_summary.py`

Creates an event contract, evaluates deterministic policy, builds evidence references, and creates an operator summary. A local Gemma/MAX structured JSON result may replace the deterministic wording only if validation and FactGuard succeed. Otherwise the deterministic result remains authoritative.

---

## 9.10 `monitor_me/assistant.py`

Implements DB-grounded question answering:

```text
question
  -> reject unsupported sensitive requests early
  -> parse labels/time/correlation intent
  -> query normalized event rows
  -> enforce same-session correlation when requested
  -> build evidence refs
  -> deterministic answer
  -> optional LLM proposal
  -> FactGuard
  -> assistant run + summary persistence
```

It exposes exact event/session/frame references, which is valuable for linking future A/V clips.

---

## 9.11 `monitor_me/event_tools.py`

Parses CCTV query concepts:

- object labels,
- object-event preference,
- time filters such as recent/today,
- union versus strict co-occurrence semantics,
- same-session all-label matching,
- evidence reference construction.

---

## 9.12 `monitor_me/fact_guard.py`

Blocks unsupported terms such as identity, face recognition, weapon, suspiciousness, and intent unless normalized evidence has the required label. It also checks evidence references and prevents answers with no evidence from sounding affirmative.

This guard should also protect future audio transcription or sound-classification claims.

---

## 9.13 `monitor_me/llm_client.py`

Defines:

- null and fake clients,
- local Gemma/MAX configuration,
- OpenAI-compatible local chat request,
- strict JSON prompt construction,
- parser and validator,
- health reporting.

The deterministic policy is not delegated to the LLM.

---

## 9.14 `monitor_me/keyframe_vlm.py` and `monitor_me/vlm_client.py`

The service resolves a stored keyframe and event, invokes an optional local Qwen endpoint, validates the structured JSON, and records success/failure. The client defaults to loopback-only behavior and data-URL encoding of the local image.

This is post-event semantic enrichment, not the fast trigger path.

---

## 9.15 `monitor_me/short_clip_vlm.py` and `monitor_me/smolvlm2_client.py`

The service finds the stored short-clip manifest and runs a constrained experiment. The client builds a JSON schema that binds event/artifact IDs, limits observation fields, and validates unsupported claims.

The implementation should be renamed or clearly documented as a **frame bundle** until an actual encoded clip is introduced.

---

## 9.16 `monitor_me/non_llm_gpu_lab.py`

This is the Python/native boundary.

### Core bridge flow

```text
frame_to_gray_u8
  -> NumPy conversion and BGR luminance weights
analyze_frames
  -> validate/prepare previous and current arrays
  -> if native binary missing:
       optional Python tile-mask fallback
  -> else:
       write raw gray temp files
       subprocess native analyze-raw-gray
       parse one JSON object
```

It also exposes synthetic wrappers for ISP, sparse ROI, mixed region, dense full-frame, overlay-heavy, AudioBox, storage batching, and evidence pipeline.

The fallback exactly preserves the 32-bit tile-mask contract.

---

## 9.17 `monitor_me/overlays.py`

Renders evidence boxes into a copied image and writes the derived artifact. Normalized coordinates are clipped to `[0,1]` before conversion to pixels. It leaves the source keyframe untouched.

---

## 9.18 `monitor_me/db.py`

The central thread-safe SQLite authority. It applies migrations and manages:

- camera/model/session/artifact/event records,
- parent/child events,
- assistant runs/summaries,
- event contracts,
- VLM experiments,
- evidence pipeline index,
- retention and scheduler runs,
- index rebuild,
- feedback,
- evidence packs and reports,
- audit history.

For future video/audio, the existing `capture_artifacts` table can store media artifacts, but additional clip-specific fields or tables would improve queryability:

- start/end timestamp,
- pre/post-roll duration,
- video codec,
- audio codec/sample rate/channels,
- frame rate and dimensions,
- A/V drift,
- trigger event IDs,
- recorder status/finalization outcome.

---

## 9.19 `monitor_me/model_registry.py`

Defines known model identities, providers, roles, paths, metadata, and enabled state. This gives reports an auditable record of which detector/VLM/native processor was configured.

---

## 9.20 `monitor_me/evidence_pack.py`

Builds a file-based package with event, session, policy, artifacts, audit entries, contracts, summaries, and metadata. It is ready to include a future A/V clip artifact as long as the clip is registered.

---

## 9.21 `monitor_me/report_tools.py`

Builds incident reports over selected events and links evidence-pack identifiers. It should eventually display clip start/end, object trigger, pre/post-roll, and AudioBox facts.

---

## 9.22 `monitor_me/operator_dashboard.py`

Builds a read-only, local, facts-only UI model and HTML. It renders summary cards and local inline chart elements without external assets or media decoding.

---

## 9.23 `monitor_me/grafana_dashboard.py`

Converts dashboard context into Prometheus text exposition and a Grafana dashboard definition. The metrics derive from persisted database facts, not live media decoding.

Future recording metrics can include:

- recorder active state,
- clips created,
- encoder/muxer failures,
- video/audio queue depth,
- dropped frames/samples,
- pre/post-roll duration,
- A/V drift,
- bytes per clip.

---

## 9.24 `monitor_me/routes.py`

Creates FastAPI routes for:

- camera/device/capture,
- models and health,
- events/artifacts,
- assistant and reports,
- evidence index/details,
- retention/scheduler/rebuild,
- operator dashboard/metrics/Grafana.

Current capture API limitation: no native GPU/evidence-pipeline fields in `CaptureRequest`.

---

## 9.25 `monitor_me/cli.py`

The broadest operational surface. It exposes:

- capture and detector readiness,
- all native synthetic workloads,
- event/artifact/evidence queries,
- retention/scheduler/rebuild operations,
- assistant/VLM/report/feedback functions.

For the present source, CLI is the only entry point that can enable native frame profiling and post-capture evidence indexing during capture-run.

---

## 9.26 Small utility/schema modules

- `schemas.py`: typed detection/evidence/assistant response dataclasses.
- `hash_utils.py`: streaming file SHA-256 and text SHA-256.
- `ids.py`: prefixed compact identifiers.
- `time_utils.py`: local-offset ISO timestamp.
- `tracker_tools.py`: operator feedback wrapper.
- `translation_tools.py`: operator-friendly alert wrapper.
- `__init__.py`: package/version surface.

---

# 10. Native C++/CUDA source study

## 10.1 Build and executable organization

`CMakeLists.txt` builds:

```text
node1_non_llm_gpu_lab_core      C++ CPU/reference library
node1_non_llm_gpu_lab           command-line JSON executable
node1_non_llm_gpu_lab_selftest  native self-test
optional CUDA object/library    when NODE1_NON_LLM_ENABLE_CUDA=ON
```

The implementation is C++17 and supports CPU-only development. CUDA builds target the configured architecture, documented for Node1 as `sm_120`.

---

## 10.2 Shared types, timing and validation

### `gpu_lab_types.hpp/.cpp`

Defines tile, frame, audio and route contracts. The tile grid is limited to 32 tiles because the occupancy mask is `uint32_t`.

Route decision:

```text
active_tiles <= sparse_threshold -> sparse
active_tiles >= dense_threshold  -> dense
otherwise                         -> mixed
```

### `gpu_lab_timing.hpp/.cpp`

Provides host stage timing and standard fields:

```text
h2d_ms
kernel_ms
d2h_ms
total_ms
```

### `gpu_lab_cuda_utils.cuh/.cu`

Wraps CUDA error formatting and CUDA-event timing.

### `gpu_lab_json.hpp/.cpp`

Serializes every analysis structure into stable JSON, including optional CPU-vs-CUDA comparison payloads and output vectors when requested.

---

## 10.3 Phase 0 — Tile-mask workload classifier

### CPU: `gpu_lab_cpu.cpp`

For each grayscale pixel:

```text
diff = abs(curr - prev)
if diff > threshold:
    locate tile by scaled x/y
    increment tile changed count
    set tile bit
    increment changed pixel count
```

After the scan it computes occupancy halves, popcounts, changed ratio, and route.

### CUDA: `frame_diff_tile_mask_kernel`

- one linear thread per pixel,
- coalesced reads of previous/current frames,
- global atomics for tile counts, mask, and changed count,
- 256 threads per block.

This is simple and deterministic. For high-density frames, global atomic pressure on `changed_pixels` and popular tile counters is a profiling target.

### Camera relevance

This is the only native frame path currently called from live capture. It decides workload shape but does not itself detect objects.

---

## 10.4 Basic audio energy path

### CPU

Sequential fixed-window sum-of-squares and RMS; sets a 32-bit event mask when RMS exceeds threshold.

### CUDA: `audio_energy_kernel`

- one block per audio window,
- each thread accumulates strided sample squares,
- shared-memory tree reduction,
- thread 0 writes RMS and event bit.

### Camera relevance

Useful as a low-cost sound activity gate, but no live samples are currently provided by Python.

---

## 10.5 Phase 1 — CPU ISP filters

Files:

- `isp_filters.hpp`
- `isp_filters_cpu.cpp`
- `isp_pnm_io.cpp`

Filters:

- box blur,
- sharpen,
- Laplacian-style edge,
- Sobel X,
- Sobel Y,
- Sobel magnitude.

The optimized CPU implementation uses a true rolling three-row buffer:

```text
load clamped rows y-1, y, y+1
compute output row
rotate three buffers
load only next incoming row
```

It avoids repeatedly loading all 3x3 neighbors from the full image and emits image-quality/workload facts such as edge energy, focus score, noise score, lighting delta, and saturation ratio.

`isp_pnm_io.cpp` supplies P5/P6 PGM/PPM file input/output for deterministic validation.

### Camera relevance

Potential pre-processing before motion/YOLO or quality monitoring. It is not currently in `LocalCameraCaptureRunner`.

---

## 10.6 Phase 2 — CUDA shared-memory ISP

CUDA kernel: `isp_filter_3x3_tiled_kernel`.

```text
16x16 output block
  -> cooperative load of 18x18 input tile including 1-pixel halo
  -> clamped border coordinates
  -> __syncthreads
  -> each valid thread reads its 3x3 window from shared memory
  -> unrolled loops
  -> output pixel
```

A second metrics kernel uses global reductions for sums, absolute differences, squared output, and saturation count.

### Camera relevance

Can reduce repeated global-memory reads for stencil filters. A persistent in-process GPU pipeline would be required for real-time benefit; per-trigger process/allocation overhead can exceed small-kernel savings.

---

## 10.7 Phase 3 — Sparse ROI crop/resize/normalize

Files:

- `sparse_roi.hpp`
- `sparse_roi_cpu.cpp`
- CUDA kernels in `gpu_lab_cuda.cu`

CPU first maps every active tile bit to a rectangular ROI. Each ROI is resized to a fixed target using deterministic nearest-neighbor sampling and normalized to float `[0,1]`.

CUDA kernel:

```text
one thread per output element across all ROIs
idx -> roi index + local output x/y
nearest-neighbor source coordinate
load one grayscale byte
write normalized float
```

A second kernel computes min/max/sum with atomics.

### Camera relevance

For localized motion, this can prepare compact tensors for a downstream detector/classifier instead of processing the entire frame. It is currently a lab mode, not connected to YOLO preprocessing.

---

## 10.8 Phase 4 — Mixed-region grouping and batch normalization

Files:

- `mixed_region.hpp`
- `mixed_region_cpu.cpp`
- CUDA kernels in `gpu_lab_cuda.cu`

CPU performs four-neighbor connected-component search on the tile mask. Each component becomes a grouped bounding region. Groups are classified as contiguous or scattered based on active-tile occupancy relative to the component bounding rectangle.

CUDA handles the expensive pixel extraction/resize/normalization, while component discovery remains on CPU.

### Camera relevance

Suitable when motion is neither a few independent tiles nor nearly the whole frame. It can batch several logical regions for downstream processing. It is not yet live-dispatched.

---

## 10.9 Phase 5 — Dense full-frame path

Files:

- `dense_full_frame.hpp`
- `dense_full_frame_cpu.cpp`
- CUDA `dense_full_frame_diff_hist_normalize_kernel`

Single-pass output facts:

- absolute difference,
- 256-bin difference histogram,
- changed pixel count and ratio,
- previous/current/diff sums and means,
- min/max values,
- lighting delta,
- normalized current frame.

CUDA maps one thread per pixel and uses global atomics for histogram and reduction values.

### Camera relevance

Best suited for full-scene changes such as lighting shifts, camera shake, or broad movement. The histogram and lighting delta can help distinguish widespread illumination changes from localized object motion, but no such suppression rule is presently wired into Python.

---

## 10.10 Phase 6 — Overlay-heavy path

Files:

- `overlay_heavy.hpp`
- `overlay_heavy_cpu.cpp`
- CUDA heat/blend and thumbnail kernels.

Processing:

```text
absolute diff
  -> grayscale heatmap
  -> deterministic red/yellow heat color
  -> fixed-point alpha blend with current grayscale
  -> RGB overlay
  -> nearest-neighbor thumbnail
  -> changed/mean/lighting metrics
```

CUDA uses one thread per full-frame pixel and a second kernel for thumbnail pixels.

### Camera relevance

Can accelerate motion heatmaps and thumbnail generation. The current live overlay is YOLO box rendering in Python/OpenCV, so Phase 6 is not yet used for production evidence images.

---

## 10.11 Phase 7 — AudioBox

Files:

- `audiobox.hpp`
- `audiobox_cpu.cpp`
- CUDA window/correlation kernels.

### CPU flow

```text
validate sample_count/rate/window/thresholds/max_lag
for each window:
  RMS
  peak
  silence flag
  onset based on first window or RMS rise
for each lag:
  normalized cross-correlation
select largest absolute correlation
break ties toward lag nearest zero
convert lag samples to milliseconds
```

### CUDA flow

`audiobox_window_kernel`:

- one block per window,
- two shared-memory arrays for sum-of-squares and peak,
- tree reduction.

`audiobox_correlation_kernel`:

- one block per candidate lag,
- shared reductions for dot product and both energies,
- normalized correlation written per lag.

The host finalizes masks and selects the best lag.

### Camera relevance

Directly useful for:

- sound activity metadata,
- onset-triggered enrichment,
- camera/microphone synchronization checks,
- comparing primary and reference audio paths.

It does not capture or encode audio.

---

## 10.12 Phase 8 — Storage batch planner and key-moment sampler

Files:

- `storage_batch.hpp`
- `storage_batch_cpu.cpp`

Manifest facts:

```text
clip_id, path, start_ms, duration_ms, bytes,
motion_score, audio_score, lighting_delta,
changed_pixels, active_tiles, optional media fingerprint
```

Priority score:

```text
motion_score
+ 0.5 * audio_score
+ 0.25 * clamp(changed_pixels / 100000)
+ 0.15 * clamp(active_tiles / 32)
+ 0.15 * clamp(lighting_delta / 255)
```

Batch planner:

- stable timeline ordering,
- maximum bytes per batch,
- maximum clips per batch,
- every manifest byte accounted for.

Key moments:

- sorted by deterministic priority,
- minimum time-gap enforcement,
- reason such as `motion_audio_peak`, `motion_peak`, `audio_peak`, `lighting_delta`, or `priority_score`.

### Camera relevance

This is already shaped for encoded clips, but integrated capture currently supplies JPEG keyframes and zero audio score. It can be reused almost directly once actual A/V clips exist.

---

## 10.13 Phase 9–11 — Evidence fingerprint and dedup pipeline

Files:

- `evidence_pipeline.hpp`
- `evidence_pipeline_cpu.cpp`

Fingerprint path:

```text
if valid decoded-keyframe fingerprint supplied:
  consume aHash, dHash, combined fingerprint, histogram
else:
  generate deterministic synthetic workload tile from metadata
  calculate aHash and dHash
  combine into fingerprint64
```

Dedup path:

- pairwise Hamming distance,
- union/find grouping for distances at or below threshold,
- representative is first group member,
- duplicate counts and nearest distance persisted.

Key-moment path:

- avoids selecting multiple members from the same duplicate group,
- applies timeline gap,
- marks group representatives.

Safety validator checks:

- non-empty/count-consistent manifest,
- batch constraints and byte accounting,
- fingerprint shape/source contract,
- duplicate group accounting,
- key-moment count/gap,
- timeline consistency,
- facts-only/no-semantic-claims flags.

Latency counters cover each stage and derive clips/ms and planned MB/s.

---

## 10.14 Phase 12–19 — Persistence and operator plane

These phases are primarily Python/SQLite/API/UI additions rather than new native kernels:

- Phase 12: normalized evidence-index tables.
- Phase 13: evidence summary/detail APIs.
- Phase 14: retention, compaction, optional VACUUM, history.
- Phase 15: read-only local dashboard.
- Phase 16: retention schedule and scheduler runs.
- Phase 17: rebuild normalized index from retained JSON artifacts.
- Phase 18: local chart-ready data and inline SVG.
- Phase 19: Prometheus metrics and Grafana dashboard definition.

---

## 10.15 Phase 20 — Nsight Compute profiling

Files:

- profile plan JSON,
- `profile_node1_gpu_lab_nsight_compute.py`,
- shell wrapper,
- phase self-test.

The plan profiles synthetic CUDA workloads only:

- ISP Sobel magnitude,
- sparse ROI,
- mixed region,
- dense full-frame,
- overlay-heavy,
- AudioBox.

Outputs include baseline JSON, `.ncu-rep`, text report, and a manifest. This provides profiler evidence without reading live CCTV media.

---

# 11. Native command dispatcher flow

`gpu_lab_main.cpp` parses a single command and emits one JSON envelope.

```text
main
  -> parse options
  -> choose mode
      synthetic
      analyze-raw-gray
      audio-raw-f32
      isp-synthetic / isp-pgm
      sparse-roi-synthetic
      mixed-region-synthetic
      dense-full-frame-synthetic
      overlay-heavy-synthetic
      audiobox-synthetic
      storage-batch-synthetic / manifest
      evidence-pipeline-synthetic / manifest
  -> create/read inputs
  -> execute CPU reference
  -> optionally execute CUDA implementation
  -> compute parity comparison
  -> serialize JSON
  -> return nonzero on invalid/failing mode
```

The executable does not open the camera or microphone. Python supplies raw frames for the one integrated live mode.

---

# 12. CPU/GPU optimization mapping to the target camera use case

```text
Stage in future recorder                 CPU/GPU code that supplements it
──────────────────────────────────────────────────────────────────────────
Camera frame quality/preprocessing       Phase 1 CPU ISP / Phase 2 CUDA ISP
Cheap scene activity classification      Phase 0 frame diff + tile mask
Localized object candidate preprocessing Phase 3 sparse ROI
Several moving regions                   Phase 4 mixed region grouping
Full-scene movement/lighting              Phase 5 dense full-frame metrics
Review artifact/thumbnail                 Phase 6 overlay-heavy
Audio activity/onset                      basic energy + Phase 7 AudioBox
Audio/video/reference drift               Phase 7 cross-correlation
Closed-clip read planning                 Phase 8 storage batches
Interesting moment ranking                Phase 8 priority/key moments
Near-duplicate evidence reduction         Phase 9 fingerprints/dedup
Audit/safety/performance                   Phase 9 latency+safety validator
Queryable operational view                Phases 12–19
Kernel optimization evidence              Phase 20 Nsight Compute
```

The missing layer is the **real-time media coordinator** that connects these primitives to live timestamped video/audio buffers and an encoder/muxer.

---

# 13. Performance and concurrency observations

## 13.1 Current Python capture loop is synchronous

Within the accepted motion branch, the capture thread can perform:

- JPEG disk write and hash,
- SQLite inserts,
- YOLO inference,
- native subprocess execution,
- overlay write and hash,
- summary construction,
- Qwen request,
- frame-bundle writes,
- SmolVLM2 request.

These operations can block subsequent camera reads. OpenCV may buffer or drop frames depending on backend behavior, but the code does not explicitly decouple capture, inference, recording, and persistence with queues/threads.

For continuous recording, split into:

```text
capture thread
trigger/inference worker
recorder/encoder worker
artifact/DB worker
post-processing worker
```

with bounded queues and explicit drop/backpressure policy.

## 13.2 Per-trigger native process overhead

The bridge starts a process and writes raw files for every accepted motion. A real-time path should use one of:

- in-process C++ extension,
- long-lived native service over shared memory/Unix socket,
- persistent worker process with reusable CUDA allocations,
- GStreamer/FFmpeg plugin integration.

## 13.3 CUDA kernel optimization observations

The source favors correctness and parity over maximum optimization:

- repeated `cudaMalloc/cudaFree` per call,
- synchronous H2D/D2H copies,
- global atomics for dense histograms and reductions,
- no streams/pinned memory/double buffering,
- no persistent device-resident frame ring,
- small result kernels followed by host finalization.

Phase 20 is appropriately designed to identify the dominant bottlenecks before optimization.

## 13.4 Object inference cadence

YOLO only runs on accepted motion events, which is efficient but may miss:

- objects entering during the minimum-gap period,
- static objects after movement stops,
- fast objects between trigger frames.

A future recorder should separate:

```text
motion evidence emission interval
from
YOLO sampling cadence
from
recording-state extension cadence
```

---

# 14. Validation evidence from this review

## 14.1 Static/source validation

- Python bytecode compilation succeeded for `monitor_me` and `tests`.
- Source search confirmed that current artifacts are JPEG, JSON, and CSV; no encoded A/V writer or audio-device ingest path exists.
- Source search confirmed `audio_score=0.0` in integrated capture evidence rows.

## 14.2 Focused Python tests

The camera/object/assistant/VLM/evidence-focused group completed successfully:

```text
34 tests passed
3 tests skipped
```

The skips explicitly required a repository-local prebuilt native CPU binary. They were not functional Python failures.

Covered behavior included:

- YOLO parent/child normalization,
- no fabricated objects when model missing,
- vehicle canonicalization,
- assistant grounding and refusal,
- evidence packs/reports,
- motion-only normalization,
- deterministic policy/contract/summary,
- Gemma structured fallback,
- Qwen and SmolVLM2 validation,
- capture evidence pipeline behavior,
- real decoded-keyframe fingerprints.

## 14.3 Native CPU build and execution

A fresh Release CPU build completed successfully. The native self-test returned:

```text
node1_non_llm_gpu_lab_selftest PASS
```

A mixed synthetic run returned the expected facts:

```text
tile mask 0x3C3C3C3C
active tiles 16
route mixed
changed ratio 0.5
```

A dense full-frame synthetic run returned `ok=true`, `dense_full_frame.ok=true`, and a changed ratio of `1.0` for the selected dense scenario.

## 14.4 Full pytest limitation

The entire suite did not finish within the sandbox time window because environment-dependent native/integration checks waited on resources not located at the repository’s expected build path. This does not replace Node1 CUDA 13.3, compute-sanitizer, live C922, live YOLO model, and Nsight Compute validation.

---

# 15. Key limitations and engineering risks

## P0 — Missing true A/V recorder

No object-triggered encoded video/audio artifact exists. This is the primary functional gap relative to the requested use case.

## P0 — No audio ingest

AudioBox is isolated from live capture. Add a timestamped PCM source and bounded ring buffer before claiming real-time audio recording or sync.

## P0 — Policy does not control media

`request_capture_review` and `duration_sec` are descriptive outputs. Add a recorder coordinator that accepts only validated deterministic directives.

## P1 — No pre/post-roll lifecycle

The JPEG deque provides recent frames only at trigger time. Add time-based pre-roll and post-roll buffers with memory and duration limits.

## P1 — Capture loop can be blocked

YOLO, disk I/O, native subprocess, and VLM can run synchronously in the camera loop. Separate workers are needed for sustained frame acquisition.

## P1 — Native route does not dispatch specialized path

The route classification is stored, but sparse/mixed/dense processing is not selected in the live path. Implement a common dispatcher contract.

## P1 — Per-event native process and allocations

Use a persistent worker/device context for real-time performance.

## P1 — API/CLI capability mismatch

FastAPI capture requests cannot enable GPU-lab/evidence pipeline options currently available in CLI.

## P2 — Confidence scale inconsistency

Motion uses percentage-like `0..100`; object detection uses `0..1`. Store `motion_score_percent` separately or normalize confidence.

## P2 — Unused configuration

`write_every_motion_keyframe` is declared but not honored.

## P2 — Event-to-artifact precision

Session-wide artifact listing can over-associate artifacts with an event. Add explicit event-artifact relation or recording table.

## P2 — No temporal object tracking

No tracker or object persistence confirms that the same object remains. Recording extension should use track IDs or at least class/bbox temporal matching.

## P2 — CSV parser limitations

The native manifest parser uses a simple comma split and does not support quoted commas in paths. Use a robust CSV parser or restrict/escape paths.

---

# 16. Recommended implementation sequence

## Step 1 — Introduce timestamped media envelopes

```text
VideoFrame {frame_id, monotonic_ns, wall_ts, BGR/encoded payload}
AudioChunk {sample_start_id, monotonic_ns, sample_rate, channels, float/PCM payload}
```

## Step 2 — Add bounded pre-roll rings

- video duration-based buffer,
- audio duration-based buffer,
- memory budget and drop metrics,
- one common monotonic clock.

## Step 3 — Add deterministic recording directive

```text
RecordingDirective {
  trigger_event_id,
  accepted_labels,
  pre_roll_ms,
  post_roll_ms,
  max_duration_ms,
  container,
  include_audio
}
```

Build it from `event_contract + capture_policy`, not from LLM/VLM text.

## Step 4 — Add recorder state machine and media writer abstraction

```text
start(directive)
append_video(frame)
append_audio(chunk)
extend(deadline)
finalize()
abort(error)
```

## Step 5 — Persist clip-specific events/artifacts

Recommended events:

```text
recording_requested
recording_started
recording_extended
recording_completed
recording_failed
```

Recommended artifact types:

```text
av_clip
video_clip
audio_clip
clip_manifest
clip_thumbnail
clip_overlay
```

## Step 6 — Integrate AudioBox

Run basic energy continuously or per buffered block. Run cross-correlation at clip finalization or on a lower cadence. Store only facts, not unsupported speech interpretation.

## Step 7 — Integrate route-specific native processing

```text
tile classifier
  sparse -> sparse ROI tensors -> YOLO ROI path
  mixed  -> connected groups -> batched inference
  dense  -> full-frame metrics -> full-frame inference or lighting suppression
```

## Step 8 — Point Phase 8/9 manifests at real closed clips

Populate nonzero audio score, actual duration/bytes, and clip path. Reuse existing batch, key-moment, fingerprint, dedup, safety, persistence, API, and dashboard code.

## Step 9 — Add real-time and failure tests

- object appears before trigger and pre-roll is present,
- object persists and post-roll extends,
- object disappears and file finalizes,
- camera read failure closes playable clip,
- audio device failure creates video-only clip with warning,
- A/V drift reported,
- no object means no encoded clip,
- detector failure still stores motion keyframe,
- DB failure does not corrupt media file,
- restart recovers or quarantines unfinished recordings.

---

# 17. Final presentation architecture

```text
                            MONITORME NODE1

CAMERA / AUDIO INGEST
  C922 V4L2 video                         AudioBox/USB audio
        |                                      |
        v                                      v
 timestamped video ring                  timestamped audio ring
        |                                      |
        +------------------+-------------------+
                           v
FAST DETERMINISTIC TRIGGER PLANE
  motion gate / native frame diff tile mask
        |
        +--> sparse ROI path
        +--> mixed grouped path
        +--> dense full-frame path
        |
        v
LOCAL YOLO OBJECT FACTS
  parent motion event -> child object events
        |
        v
DETERMINISTIC RECORDING POLICY
  class/confidence/persistence -> recording directive
        |
        v
A/V RECORDER STATE MACHINE
  pre-roll -> live recording -> post-roll -> mux/finalize
        |
        +--> encoded MP4/MKV + audio
        +--> raw and annotated keyframes
        +--> AudioBox RMS/peak/onset/drift facts
        +--> hashes + clip manifest
        |
        v
EVIDENCE AND OPERATIONS PLANE
  SQLite normalized events/artifacts/contracts
  evidence fingerprints/dedup/key moments
  assistant + reports + evidence packs
  dashboard + Prometheus + Grafana
  retention + scheduled cleanup + rebuild

Optional local semantic enrichment occurs only after evidence exists:
  Gemma/MAX summaries
  Qwen keyframe analysis
  SmolVLM2 encoded-clip/frame-bundle experiment
```

---

# 18. Final assessment

MonitorMe has already solved many of the difficult non-recording parts of an evidence-grade camera assistant:

- local capture and privacy boundary,
- deterministic motion gating,
- optional local object detection,
- normalized parent/child events,
- artifact hashing and session manifests,
- conservative fault handling,
- structured assistant/VLM validation,
- C++/CUDA workload primitives,
- evidence fingerprinting and deduplication,
- query, report, dashboard, retention, and rebuild planes,
- profiler-driven CUDA validation.

The next major architectural milestone is not another model. It is a **timestamped, concurrent, object-confirmed A/V recording coordinator** that connects the existing trigger, policy, AudioBox, native routing, artifact, and evidence-index components into one real-time media lifecycle.
