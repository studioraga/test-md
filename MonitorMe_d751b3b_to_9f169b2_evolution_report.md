# MonitorMe evolution report: commit d751b3b → commit 9f169b2

Source inspected: `/mnt/data/MonitorMe-0.4.2-p20.tar.gz` extracted as a git repository. The repository also contains an earlier `2f2322d Initial commit`, but this report follows the user-requested baseline `d751b3b` through top commit `9f169b2`.

## Executive summary

From `d751b3b` to `9f169b2`, MonitorMe grows from a standalone Node1-local CCTV evidence assistant into a layered evidence, assistant, native CPU/CUDA workload, evidence-index, dashboard, Grafana, and profiling platform. The design consistently keeps privacy and evidence traceability as first-class requirements: frames and artifacts remain local; positive assistant claims must be backed by event/session/frame/artifact/model references; optional LLM/VLM paths are disabled or guarded; GPU/native modules emit workload and signal-processing facts rather than semantic claims.

Major growth arcs:

1. **Evidence-first CCTV foundation**: local C922 capture, motion gate, YOLO-after-trigger, artifacts, overlays, SQLite schema, evidence packs, reports, feedback, API, and tests.
2. **Assistant safety arc**: v0.1 deterministic event contracts/summaries, v0.2 optional Gemma/MAX strict JSON explanations, v0.3 optional Qwen keyframe analysis, and v0.4.1 SmolVLM2 structured clip experiments.
3. **Native non-LLM GPU lab arc**: C++/CUDA sidecar grows from tile-mask routing into CPU/CUDA ISP, sparse ROI, mixed region, dense full-frame, overlay-heavy, AudioBox, storage planning, and evidence fingerprint/dedup workloads.
4. **Evidence-index productization arc**: capture-run integration, decoded keyframe fingerprints, normalized persistence, summary API, retention, scheduler, rebuild, operator dashboard, charts, Prometheus/Grafana, and Nsight Compute profiling.

## Inspection and validation performed in this review

- **compileall**: PASSED for monitor_me and tests in sandbox
- **pytest**: PASSED: 172 collected; 92 passed; 80 skipped because native CPU/CUDA binaries were not built in sandbox
- **git_range**: d751b3b..9f169b2: 158 tracked files changed, +28,902 insertions, -15 deletions
- **phase20_artifacts**: Uploaded runtime results include baseline JSON, .ncu-rep, .ncu.txt, and manifest for six workloads under results/node1_gpu_lab/nsight_compute/phase20_20260706_212820

The sandbox does not include the Node1 CUDA build artifacts, so native-binary tests correctly skipped. Uploaded Phase 20 runtime results are present and are summarized later.

## Commit-by-commit functional and non-functional summary

### 01. `d751b3b` — AI system in local infra for Camera used as evidence assistant

- **Phase / milestone:** Foundation / Step 17A-E
- **Tracked change size:** 67 files, +5985/-1. Areas: root=5, docs=15, migrations=1, monitor_me=23, scripts=11, tests=12.
- **Functional additions:** Created the standalone MonitorMe Node1-local evidence assistant: SQLite evidence DB, local C922 capture, frame-difference motion gate, local artifacts/manifests, YOLO ONNX detection after motion, detector health checks, object-query grounding, evidence packs, incident reports, feedback/audit records, FastAPI routes, CLI tools, overlays, model registry, and validation scripts.
- **Non-functional / safety / reliability additions:** Evidence-first and local-only design; raw frames stay local; facts must reference event/session/frame/artifact/model metadata; object rows are only created from detector output; overlays preserve raw keyframes; safety guard rejects unsupported claims; API payloads parsed as JSON bodies; docs and test extras established.
- **Implementation surface:** Python package under monitor_me/*; migration 001 schema; scripts for API/capture/model download; tests for routes, grounding, camera devices, detector health, normalization, overlays, evidence packs, feedback/correlation, YOLO pipeline.
- **Architecture view at this commit:**

```text
C922 /dev/video0 -> OpenCV local_capture -> motion gate -> keyframe/manifest artifacts
    -> events table: motion_detected
    -> optional YOLO ONNX -> events table: object_detected child rows
    -> evidence_pack / incident_report / feedback / audit_log
    -> MonitorMeAssistant + FactGuard -> evidence-backed answer/API/CLI
```

### 02. `da3f66a` — Add Node1 AI Camera Assistant v0.1 event contracts and summaries

- **Phase / milestone:** Node1 AI Camera Assistant v0.1
- **Tracked change size:** 26 files, +1445/-7. Areas: root=2, docs=10, migrations=1, monitor_me=11, scripts=1, tests=1.
- **Functional additions:** Added event contracts, deterministic capture/review policy, automatic assistant summaries after motion/YOLO events, summary and event-contract CLI/API routes, and incident report summary inclusion.
- **Non-functional / safety / reliability additions:** Keeps YOLO as visual-fact producer and Node1 policy as deterministic decision layer; no Gemma yet; summaries are generated only from stored evidence; hallucination/non-invention tests prove unsupported facts are not emitted.
- **Implementation surface:** migration 002 event_contracts; assistant_summary.py; capture_policy.py; event_contract.py; yolo_client.py; DB APIs; CLI/routes extensions; validation script for v0.1.
- **Architecture view at this commit:**

```text
motion/object event -> event_contract.py -> event_contracts table
                   -> capture_policy.py -> policy_decision_json
                   -> assistant_summary.py -> assistant_summaries table
                   -> assistant/report/evidence_pack -> operator-facing grounded narrative
```

### 03. `1ee6a57` — feat: add Node1 AI Camera Assistant v0.2 with validated MAX Gemma startup

- **Phase / milestone:** Node1 AI Camera Assistant v0.2 / MAX Gemma
- **Tracked change size:** 30 files, +3373/-18. Areas: root=3, docs=8, monitor_me=5, scripts=10, tests=4.
- **Functional additions:** Added optional local Gemma 3 1B via MAX as a strict JSON explanation layer; added llm-health, Gemma summary source, deterministic fallback summaries, and live MAX helper scripts for TERM1/TERM2 validation.
- **Non-functional / safety / reliability additions:** Gemma is disabled by default, receives structured facts rather than raw frames, and cannot decide actions. Invalid/unavailable LLM output falls back to deterministic v0.1 summary. Helper scripts avoid accidental Pixi/MAX mutation, preserve Mojo caches, and document RTX 5060 Ti startup constraints.
- **Implementation surface:** llm_client.py; assistant_summary.py strict JSON path; MAX/Pixi scripts under scripts/max; docs/MAX_GEMMA_NODE1.md; tests for MAX helper behavior and v0.2 summaries.
- **Architecture view at this commit:**

```text
events + event_contract + deterministic policy + artifact refs
       -> optional Gemma/MAX OpenAI-compatible /v1/chat/completions
       -> strict JSON validator
       -> valid: assistant_summaries(summary_source=gemma_max)
       -> invalid/unavailable: deterministic fallback summary + audit reason
```

### 04. `1138403` — Add Node1 Assistant v0.3 Qwen VLM keyframe analysis

- **Phase / milestone:** Node1 AI Camera Assistant v0.3 / Qwen VLM keyframe analysis
- **Tracked change size:** 18 files, +1022/-7. Areas: root=2, docs=5, migrations=1, monitor_me=8, scripts=1, tests=1.
- **Functional additions:** Added optional Qwen VLM keyframe analysis after a trigger and after a stored keyframe artifact exists; added VLM health, manual VLM analyze/list commands and API routes.
- **Non-functional / safety / reliability additions:** Disabled by default, local-only by default, remote endpoint rejected unless explicitly allowed. VLM output is companion context, not detector truth, cannot create YOLO rows, and is validated against identity/intent/threat/weapon/person-profile claims.
- **Implementation surface:** migration 003 vlm_keyframe_analyses; vlm_client.py; keyframe_vlm.py; local_capture integration; routes/CLI; model registry; v0.3 validation tests.
- **Architecture view at this commit:**

```text
motion/YOLO trigger -> stored keyframe artifact
       -> optional QwenVLMOpenAIClient
       -> strict keyframe JSON with event_id/artifact_id citations
       -> vlm_keyframe_analyses table
       -> assistant may read as companion visual context, not detector fact
```

### 05. `8fcd276` — fix: constrain SmolVLM2 short clip output with structured JSON schema

- **Phase / milestone:** Node1 AI Camera Assistant v0.4.1 / SmolVLM2 structured clip experiment
- **Tracked change size:** 20 files, +1441/-7. Areas: root=3, docs=4, migrations=1, monitor_me=9, scripts=2, tests=1.
- **Functional additions:** Constrained SmolVLM2 short-clip output using vLLM structured_outputs JSON schema; introduced bounded visual-state fields and health/analyze/list routes and CLI.
- **Non-functional / safety / reliability additions:** Fixes freeform/truncated/unsafe VLM output risk; disabled by default and local-only; const-binds event_id and artifact_id; rejects identity, profiling, intent, threat, weapon, and unsupported evidence-ID claims.
- **Implementation surface:** migration 004 smolvlm2_clip_experiments; smolvlm2_client.py; short_clip_vlm.py; fact_guard updates; local_capture clip artifact path; tests and validation scripts for v0.4/v0.4.1.
- **Architecture view at this commit:**

```text
trigger event + local short clip artifact
       -> SmolVLM2 OpenAI-compatible request with structured_outputs={json:schema}
       -> bounded fields: visible_scene/person_like_presence/vehicle_like_presence/motion_claim/safe_observation/unsupported_claims
       -> smolvlm2_clip_experiments table
       -> safe experimental companion output
```

### 06. `d814a12` — Add Node1 non-LLM C++ CUDA GPU workload optimization module

- **Phase / milestone:** Native Node1 non-LLM GPU workload lab v0.1
- **Tracked change size:** 15 files, +1776/-1. Areas: root=2, docs=1, monitor_me=4, native=8.
- **Functional additions:** Added optional C++/CUDA sidecar for deterministic CPU/GPU workload profiling; analyzes previous/current frames, builds tile masks, classifies sparse/mixed/dense paths, emits gpu_workload_profiled child evidence rows, and exposes gpu-lab-health/gpu-lab-synthetic.
- **Non-functional / safety / reliability additions:** Runs only behind --gpu-lab-enabled after local motion/keyframe trigger. Emits workload metadata only; no identity/object/person/intent/weapon claims. Python bridge has safe fallback. Native build outputs ignored.
- **Implementation surface:** native/node1_non_llm_gpu_inference_lab CMake project; gpu_lab_cpu.cpp, gpu_lab_cuda.cu, gpu_lab_main.cpp; monitor_me/non_llm_gpu_lab.py; local_capture integration; docs and smoke scripts.
- **Architecture view at this commit:**

```text
capture-run trigger -> previous/current frames
       -> Python bridge -> native C++/CUDA sidecar
       -> CPU/CUDA tile mask + changed ratio + path sparse|mixed|dense
       -> gpu_workload_profiled child event with privacy facts
       -> capture manifest references workload profile
```

### 07. `b7518be` — refactor: harden Node1 non-LLM GPU lab Phase 0

- **Phase / milestone:** GPU lab Phase 0 hardening
- **Tracked change size:** 19 files, +1033/-323. Areas: docs=1, monitor_me=1, native=16, tests=1.
- **Functional additions:** Refactored native sidecar into reusable structs, route helpers, validation, JSON serialization, host timing, CUDA error/event timing, and self-test executable.
- **Non-functional / safety / reliability additions:** Established stable contracts before adding ISP/ROI/dense/overlay/audio/storage modules; Python fallback validation now mirrors native rules; timing facts added without semantic claims.
- **Implementation surface:** gpu_lab_types.hpp/cpp; gpu_lab_json.hpp/cpp; gpu_lab_timing.hpp/cpp; gpu_lab_cuda_utils.cuh/cu; gpu_lab_selftest.cpp; Phase 0 pytest and script.
- **Architecture view at this commit:**

```text
native CLI modes -> shared config/types validation
              -> CPU/CUDA kernels
              -> shared JSON emitters + timing
              -> selftest coverage
Python fallback -> same tile-contract validation rules
```

### 08. `ef6f6b2` — Add CPU ISP rolling line-buffer filters to Node1 GPU lab

- **Phase / milestone:** GPU lab Phase 1 CPU ISP rolling line-buffer filters
- **Tracked change size:** 16 files, +1128/-2. Areas: root=1, docs=1, monitor_me=2, native=11, tests=1.
- **Functional additions:** Added CPU-only 3x3 ISP filters: blur, sharpen, edge/conv-edge, sobel-x, sobel-y, sobel-mag; added P5/P6 PGM/PPM I/O and gpu-lab-isp-synthetic CLI.
- **Non-functional / safety / reliability additions:** Implements true rolling 3-row buffer for memory-locality baseline; facts-only schema reports image processing metrics, not visual semantics; CUDA intentionally deferred to Phase 2.
- **Implementation surface:** isp_filters.hpp; isp_filters_cpu.cpp; isp_pnm_io.cpp; native isp-synthetic/isp-pgm modes; tests for rolling output vs naive reference and PGM/PPM path.
- **Architecture view at this commit:**

```text
PGM/PPM or synthetic gray image
       -> rolling line buffer y-1/y/y+1
       -> 3x3 CPU filter
       -> output metrics: mean/edge/focus/noise/lighting/saturation
       -> facts-only ISP JSON
```

### 09. `6de5e80` — Add CUDA ISP filters and CPU parity metrics to Node1 GPU lab

- **Phase / milestone:** GPU lab Phase 2 CUDA ISP filters
- **Tracked change size:** 13 files, +738/-6. Areas: root=1, docs=1, monitor_me=2, native=8, tests=1.
- **Functional additions:** Added shared-memory tiled CUDA versions of the Phase 1 ISP filters and CPU-vs-CUDA parity metrics.
- **Non-functional / safety / reliability additions:** CUDA uses 16x16 output tile plus 1-pixel halo; comparison proves output/metrics parity with mismatch/max-diff fields; still facts-only.
- **Implementation surface:** CUDA ISP kernels in gpu_lab_cuda.cu; isp_cuda JSON fields; gpu-lab-isp-synthetic wrapper; Phase 2 CUDA selftest and pytest.
- **Architecture view at this commit:**

```text
gray image -> CPU rolling ISP reference
          -> CUDA shared tile [16+2][16+2] + 3x3 stencil
          -> isp + isp_cuda
          -> isp_cpu_cuda_comparison(output_equal, metrics_close, facts_only)
```

### 10. `35ce5aa` — Add sparse ROI crop resize normalize path

- **Phase / milestone:** GPU lab Phase 3 sparse ROI crop/resize/normalize
- **Tracked change size:** 17 files, +1127/-2. Areas: root=1, docs=1, monitor_me=2, native=12, tests=1.
- **Functional additions:** Added active-tile walking, ROI rectangle generation, grayscale crop, nearest-neighbor resize, uint8-to-float32 normalization, CUDA ROI kernel, and CLI/native modes.
- **Non-functional / safety / reliability additions:** Optimizes sparse activity path instead of full-frame processing; CPU/CUDA comparison validates ROI equality/output closeness; no semantic object/person claims.
- **Implementation surface:** sparse_roi.hpp/cpp; CUDA sparse_roi_resize_normalize and stats kernels; scripts for CPU/CUDA selftests; pytest for CPU, CUDA parity when available, and CLI wrapper.
- **Architecture view at this commit:**

```text
tile mask with few active tiles
       -> active tile walking
       -> ROI list
       -> crop/resize/normalize CPU + CUDA
       -> sparse_roi_cpu_cuda_comparison(rois_equal, output_close)
```

### 11. `a222d19` — Add mixed region connected tile grouping path

- **Phase / milestone:** GPU lab Phase 4 mixed-region connected tile grouping
- **Tracked change size:** 17 files, +1277/-6. Areas: root=1, docs=1, monitor_me=2, native=12, tests=1.
- **Functional additions:** Added 4-neighbor connected components over active tiles, contiguous/scattered classification, grouped crop batching, CUDA grouped crop/resize/normalize, and CLI/native modes.
- **Non-functional / safety / reliability additions:** Handles medium activity where ROI-by-tile is inefficient but dense processing is unnecessary; emits grouping and normalization facts only; CPU/CUDA groups and outputs must match.
- **Implementation surface:** mixed_region.hpp/cpp; CUDA mixed region kernels; CPU/CUDA scripts for contiguous/scattered/dense scenarios; tests for parity and CLI.
- **Architecture view at this commit:**

```text
tile mask with medium activity
       -> 4-neighbor connected components
       -> group bounding boxes + classification contiguous|scattered
       -> grouped crop/resize/normalize CPU + CUDA
       -> mixed_region_cpu_cuda_comparison(groups_equal, output_close)
```

### 12. `a15fc38` — Add dense full-frame diff histogram path

- **Phase / milestone:** GPU lab Phase 5 dense full-frame diff/histogram/reduction
- **Tracked change size:** 17 files, +1071/-1. Areas: root=1, docs=1, monitor_me=2, native=12, tests=1.
- **Functional additions:** Added high-activity dense full-frame path: absolute diff, changed-pixel reduction, 256-bin histogram, previous/current/diff reductions, lighting delta, full-frame normalization, CUDA path, and CLI.
- **Non-functional / safety / reliability additions:** Designed for dense route where sparse ROI wastes bandwidth; validates histogram/reduction/normalization parity; facts-only high-activity numeric metrics.
- **Implementation surface:** dense_full_frame.hpp/cpp; CUDA dense diff/hist/reduction kernels; selftest scripts; tests/test_node1_dense_full_frame_phase5.py.
- **Architecture view at this commit:**

```text
previous/current gray full frame
       -> abs diff + changed-pixel reduction
       -> 256-bin histogram + reductions + lighting delta
       -> dense normalize float32
       -> dense_full_frame_cpu_cuda_comparison(histogram_equal, reductions_close, normalized_close)
```

### 13. `3ab08bb` — Add overlay-heavy alpha blend heatmap path

- **Phase / milestone:** GPU lab Phase 6 overlay-heavy alpha/heatmap/thumbnail
- **Tracked change size:** 17 files, +1277/-1. Areas: root=1, docs=1, monitor_me=2, native=12, tests=1.
- **Functional additions:** Added visualization-heavy path: full-frame diff, motion heatmap, deterministic heat coloring, alpha blend against current frame, RGB overlay, thumbnail generation, before/after metrics, CUDA path, and CLI.
- **Non-functional / safety / reliability additions:** Optimizes artifact-generation workload without semantic interpretation; ensures heatmap/overlay/thumbnail parity; local facts-only visualization metadata.
- **Implementation surface:** overlay_heavy.hpp/cpp; CUDA heatmap/blend/thumbnail kernels; Phase 6 selftests and pytest.
- **Architecture view at this commit:**

```text
previous/current gray
       -> diff -> motion heatmap
       -> alpha blend -> RGB overlay
       -> thumbnail
       -> overlay_heavy_cpu_cuda_comparison(heatmap_equal, overlay_equal, thumbnail_equal)
```

### 14. `dab7f14` — Add audiobox rms peak sync drift path

- **Phase / milestone:** GPU lab Phase 7 AudioBox RMS/peak/sync drift
- **Tracked change size:** 17 files, +1314/-2. Areas: root=1, docs=1, monitor_me=2, native=12, tests=1.
- **Functional additions:** Added numeric audio workload path: per-window RMS/peak, silence mask, onset mask, bounded cross-correlation, sync drift in samples/ms, CUDA path, and CLI.
- **Non-functional / safety / reliability additions:** No speech transcription, speaker identity, behavior, intent, or semantic audio claims; validates masks, drift, RMS/peak/correlation parity.
- **Implementation surface:** audiobox.hpp; audiobox_cpu.cpp; CUDA audiobox_window and correlation kernels; native mode audiobox-synthetic; tests/test_node1_audiobox_phase7.py.
- **Architecture view at this commit:**

```text
primary/reference float32 audio
       -> RMS/peak windows
       -> silence/onset masks
       -> bounded cross-correlation [-max_lag,+max_lag]
       -> sync drift samples/ms
       -> audiobox_cpu_cuda_comparison(masks_equal, drift_equal, rms_close)
```

### 15. `8103b56` — Add storage batch planner and clip sampler path

- **Phase / milestone:** GPU lab Phase 8 storage batch planner / clip sampler
- **Tracked change size:** 15 files, +1095/-2. Areas: root=1, docs=1, monitor_me=2, native=10, tests=1.
- **Functional additions:** Added facts-only clip manifest scanner, priority scoring, constrained batch read plan, key moment selection with minimum gap, timeline features, synthetic/manifest modes, and CLI.
- **Non-functional / safety / reliability additions:** No media decoding and no visual/audio semantic claims; supports storage/batch-read optimization and deterministic key-moment selection.
- **Implementation surface:** storage_batch.hpp/cpp; storage-batch-synthetic and storage-batch-manifest native modes; tests/test_node1_storage_batch_phase8.py.
- **Architecture view at this commit:**

```text
clip manifest CSV
       -> metadata scan
       -> priority scoring
       -> constrained batch read plan
       -> key moments with min gap
       -> storage_batch facts: bytes/timeline/gaps/scores
```

### 16. `cc374ed` — Add evidence fingerprint dedup pipeline

- **Phase / milestone:** GPU lab Phase 9 evidence fingerprint/dedup pipeline
- **Tracked change size:** 15 files, +1299/-7. Areas: root=1, docs=1, monitor_me=2, native=10, tests=1.
- **Functional additions:** Added deterministic evidence-pipeline workload: storage planner reuse, synthetic visual fingerprints, ahash/dhash/fingerprint/histogram facts, duplicate grouping, dedup-aware key-moment selection, latency/throughput counters, and safety validator.
- **Non-functional / safety / reliability additions:** Does not decode media in this phase and emits no semantic visual/audio claims; validates manifest counts, batch constraints, fingerprint shape, dedup accounting, key-moment spacing, and timeline totals.
- **Implementation surface:** evidence_pipeline.hpp/cpp; evidence-pipeline-synthetic/manifest native modes; Python bridge CLI; Phase 9 selftest and pytest.
- **Architecture view at this commit:**

```text
clip manifest/timeline metadata
       -> storage batch planner
       -> deterministic fingerprints + histograms
       -> Hamming dedup groups
       -> dedup-aware key moments
       -> latency/throughput + safety validator
       -> evidence_pipeline JSON
```

### 17. `6b3092b` — Integrate capture-run evidence pipeline

- **Phase / milestone:** GPU lab Phase 10 capture-run evidence pipeline integration
- **Tracked change size:** 9 files, +625/-2. Areas: root=1, docs=1, monitor_me=4, native=2, tests=1.
- **Functional additions:** Integrated native evidence pipeline into real capture-run after local motion keyframes; writes evidence CSV manifest, native profile JSON artifact, evidence_pipeline_indexed session event, and final manifest references.
- **Non-functional / safety / reliability additions:** Facts-only, post-capture metadata path; no media upload, no identity/intent/person/object claims; native pipeline runs from local artifact metadata.
- **Implementation surface:** local_capture evidence-pipeline config/bridge; non_llm_gpu_lab.py runner additions; CLI options --evidence-pipeline-enabled and tuning options; Phase 10 selftest.
- **Architecture view at this commit:**

```text
capture-run keyframes
       -> capture manifest frame records
       -> evidence_pipeline_manifest.csv
       -> native evidence-pipeline-manifest
       -> evidence_pipeline_profile.json artifact
       -> evidence_pipeline_indexed event
       -> final manifest references both artifacts and event
```

### 18. `6f098fc` — Ingest decoded keyframe fingerprints

- **Phase / milestone:** GPU lab Phase 11 decoded keyframe fingerprints
- **Tracked change size:** 14 files, +612/-19. Areas: root=1, docs=1, monitor_me=3, native=7, tests=2.
- **Functional additions:** Added local decoded-keyframe fingerprint generation: ahash64, dhash64, combined fingerprint64, decoded dimensions, and 16-bin luminance histogram written into evidence CSV and ingested by native pipeline.
- **Non-functional / safety / reliability additions:** Decoding happens locally after keyframe already exists as evidence; still facts-only, no object/person/identity/intent claims; supports fallback metadata_synthetic fingerprints if real decode unavailable.
- **Implementation surface:** local_capture decode/fingerprint helpers; native manifest fields fingerprint_source/decoded_width/height/hash/histogram16; Python/native tests.
- **Architecture view at this commit:**

```text
stored keyframe JPEG
       -> local grayscale decode
       -> resize to fingerprint grid
       -> ahash64 + dhash64 + fingerprint64 + histogram16
       -> CSV manifest extended columns
       -> native evidence_pipeline real_media_ingestion facts
```

### 19. `499a567` — Persist evidence pipeline index

- **Phase / milestone:** GPU lab Phase 12 evidence index persistence
- **Tracked change size:** 9 files, +846/-0. Areas: root=1, docs=1, migrations=1, monitor_me=3, native=2, tests=1.
- **Functional additions:** Persisted evidence-pipeline profile output into normalized SQLite tables: profiles, fingerprints, dedup groups, and key moments; added CLI list/readback commands.
- **Non-functional / safety / reliability additions:** Moves from large JSON-only artifact to queryable index while keeping source event/artifact; stores hashes, histograms, duplicate accounting, timeline and safety facts only.
- **Implementation surface:** migration 005; MonitorMeDB persist/list helpers; local_capture calls persist after successful pipeline; tests/test_node1_evidence_index_phase12.py.
- **Architecture view at this commit:**

```text
evidence_pipeline_profile.json + evidence_pipeline_indexed event
       -> persist_evidence_pipeline_index
       -> evidence_pipeline_profiles
       -> evidence_fingerprints
       -> evidence_dedup_groups
       -> evidence_key_moments
       -> CLI/API-ready query index
```

### 20. `d7a8a55` — Expose evidence pipeline summary API

- **Phase / milestone:** GPU lab Phase 13 evidence pipeline summary API
- **Tracked change size:** 8 files, +654/-0. Areas: root=1, docs=2, monitor_me=2, native=2, tests=1.
- **Functional additions:** Exposed persisted evidence index through local FastAPI summary/detail endpoints for summaries, session summary, profile summary, fingerprints, dedup groups, and key moments.
- **Non-functional / safety / reliability additions:** API is read-only, bounded, local, and facts-only; no media decode, raw-frame upload, native rerun, or semantic claims during API calls.
- **Implementation surface:** routes.py evidence/pipeline endpoints; DB summary constructors; docs/API; Phase 13 selftest and pytest.
- **Architecture view at this commit:**

```text
SQLite evidence index tables
       -> DB summary helpers
       -> /evidence/pipeline/summaries
       -> /sessions/{id}/summary / profiles/{id}/summary
       -> bounded fingerprints/dedup/key-moments endpoints
       -> operator/report/dashboard consumers
```

### 21. `e11363e` — Add evidence index retention policies

- **Phase / milestone:** GPU lab Phase 14 evidence index retention/compaction
- **Tracked change size:** 10 files, +1001/-0. Areas: root=1, docs=2, migrations=1, monitor_me=3, native=2, tests=1.
- **Functional additions:** Added retention planning/apply for old normalized evidence index rows with keep-last-per-camera/session guards; added retention run audit table, CLI, and API.
- **Non-functional / safety / reliability additions:** Deletes only normalized index rows; retains source events, capture artifacts, keyframes, CSV manifests, JSON profile artifacts, and reportability. Destructive apply requires explicit confirmation; supports dry-run, WAL checkpoint, and optional VACUUM.
- **Implementation surface:** migration 006 evidence_retention_runs; MonitorMeDB plan/apply/list APIs; CLI evidence-retention-*; routes retention endpoints; tests/test_node1_evidence_retention_phase14.py.
- **Architecture view at this commit:**

```text
evidence_pipeline_profiles + child index rows
       -> retention policy older_than_days + keep-last guards
       -> dry-run plan or confirmed apply
       -> delete normalized index rows only
       -> evidence_retention_runs audit
       -> retained artifacts remain rebuildable
```

### 22. `23286dc` — Add operator evidence dashboard

- **Phase / milestone:** GPU lab Phase 15 operator evidence dashboard
- **Tracked change size:** 8 files, +630/-0. Areas: root=1, docs=2, monitor_me=2, native=2, tests=1.
- **Functional additions:** Added local read-only operator dashboard and /operator/dashboard/data JSON context over evidence index, retention audit rows, selected profile details, and links to JSON evidence APIs.
- **Non-functional / safety / reliability additions:** No external assets, media decode, capture run, destructive retention, identity/intent/speech inspection, or semantic claims from dashboard request path.
- **Implementation surface:** operator_dashboard.py; routes /operator/dashboard and /operator/dashboard/data; docs/API and native docs; tests/test_node1_operator_dashboard_phase15.py.
- **Architecture view at this commit:**

```text
SQLite evidence index + retention runs
       -> build_operator_dashboard_context
       -> /operator/dashboard/data JSON
       -> /operator/dashboard local HTML
       -> read-only evidence counts, safety, key moments, dedup, retention history
```

### 23. `7ba029f` — Automate evidence index retention scheduling

- **Phase / milestone:** GPU lab Phase 16 scheduled retention automation
- **Tracked change size:** 14 files, +1102/-0. Areas: root=1, docs=2, migrations=1, monitor_me=4, native=2, scripts=3, tests=1.
- **Functional additions:** Added configurable local evidence retention scheduler with default disabled dry-run schedule, CLI/API configuration, scheduler run audit rows, and systemd timer/service scripts.
- **Non-functional / safety / reliability additions:** Disabled by default and dry-run by default; destructive scheduled retention requires --apply --yes or API confirm=true; delegates to Phase 14 retention, so scope remains normalized index rows only.
- **Implementation surface:** migration 007 schedule and scheduler_runs; MonitorMeDB scheduler APIs; CLI evidence-retention-schedule-*; routes schedule endpoints; scripts/systemd; run_evidence_retention_scheduler_once.sh; pytest.
- **Architecture view at this commit:**

```text
evidence_retention_schedule(default disabled, dry-run)
       -> manual/cron/systemd scheduler runner
       -> due/skipped/forced decision
       -> Phase 14 apply_evidence_index_retention
       -> evidence_retention_scheduler_runs audit
```

### 24. `6c6a9a9` — Rebuild evidence index from retained artifacts

- **Phase / milestone:** GPU lab Phase 17 evidence index rebuild from retained artifacts
- **Tracked change size:** 11 files, +1023/-0. Areas: root=1, docs=2, migrations=1, monitor_me=4, native=2, tests=1.
- **Functional additions:** Added plan/apply rebuild workflow that scans retained evidence_pipeline_indexed events and retained evidence_pipeline_profile JSON artifacts, then rehydrates normalized SQLite index rows.
- **Non-functional / safety / reliability additions:** Closes retention loop without rerunning CUDA/native pipeline, decoding images, reading pixels, uploading frames, or making new semantic claims. Non-dry-run and replace-existing require explicit confirmation.
- **Implementation surface:** migration 008 evidence_index_rebuild_runs; DB rebuild APIs and get_artifact; CLI evidence-index-rebuild-*; API rebuild endpoints; Phase 17 tests.
- **Architecture view at this commit:**

```text
retained evidence_pipeline_indexed event
       -> retained evidence_pipeline_profile artifact
       -> read profile JSON result.evidence_pipeline
       -> rebuild evidence_pipeline_profiles/fingerprints/dedup/key_moments
       -> evidence_index_rebuild_runs audit
```

### 25. `200fad8` — Add operator dashboard charts

- **Phase / milestone:** GPU lab Phase 18 richer operator dashboard charts
- **Tracked change size:** 8 files, +527/-0. Areas: root=1, docs=2, monitor_me=2, native=2, tests=1.
- **Functional additions:** Added chart-ready facts and inline HTML/SVG dashboard charts: profile points, fingerprint composition, latency breakdown, key moment timeline, Hamming sample, operation audit, and safety checks.
- **Non-functional / safety / reliability additions:** Charts are derived only from persisted SQLite rows and audit rows; inline CSS/SVG only; no external JS/CSS/chart libraries/assets, media decode, native rerun, or semantic claims.
- **Implementation surface:** operator_dashboard.py chart builders/renderers; route data includes charts; docs and tests/test_node1_operator_dashboard_charts_phase17.py.
- **Architecture view at this commit:**

```text
operator dashboard context
       -> charts: profile_points/fingerprint_composition/latency/key_moments/hamming/audit/safety
       -> inline HTML + SVG
       -> richer local UI without external dependencies
```

### 26. `54b6fb9` — Add Grafana operator dashboard integration

- **Phase / milestone:** GPU lab Phase 19 Grafana/dashboard integration
- **Tracked change size:** 12 files, +1125/-1. Areas: root=1, configs=3, docs=2, monitor_me=3, native=2, tests=1.
- **Functional additions:** Exposed operator dashboard facts as Prometheus text metrics and a Grafana dashboard JSON model; added Prometheus and Grafana provisioning config files.
- **Non-functional / safety / reliability additions:** Reuses Phase 15-18 context, avoiding second interpretation layer. Metrics/Grafana remain facts-only/local-only: no media decode, raw-frame upload, native rerun, external calls, destructive actions, identity/intent/speech/danger claims.
- **Implementation surface:** grafana_dashboard.py; routes /operator/dashboard/metrics and /operator/dashboard/grafana/dashboard.json; configs/prometheus and configs/grafana; Phase 19 tests.
- **Architecture view at this commit:**

```text
SQLite evidence index + audit rows
       -> build_operator_dashboard_context
       -> /operator/dashboard/metrics Prometheus exposition
       -> Prometheus scrape
       -> Grafana dashboard JSON/provisioning
       -> local operator observability panels
```

### 27. `9f169b2` — Add Nsight Compute profiling pass

- **Phase / milestone:** GPU lab Phase 20 Nsight Compute profiling pass
- **Tracked change size:** 9 files, +740/-0. Areas: root=1, configs=1, docs=2, native=4, tests=1.
- **Functional additions:** Added reproducible Nsight Compute profiling workflow covering Phase 2-7 CUDA workloads: ISP Sobel, sparse ROI, mixed region, dense full-frame, overlay-heavy, and AudioBox. Supports dry-run, execute, workload selection, output-dir, ncu set/metrics, no-build, baseline JSON, .ncu-rep, .ncu.txt, and manifest output.
- **Non-functional / safety / reliability additions:** Profiles synthetic native workloads only; separate from API/dashboard/capture paths; output goes under results/ and must not be staged. Build targets CUDA 13.3, sm_120, RTX 5060 Ti environment. CI can dry-run without GPU/Nsight.
- **Implementation surface:** configs/nsight_compute/node1_gpu_lab_phase20_profile_plan.json; profile_node1_gpu_lab_nsight_compute.py/.sh; Phase 20 selftest and pytest.
- **Architecture view at this commit:**

```text
Phase 2-7 CUDA synthetic workloads
       -> profile plan JSON
       -> profile_node1_gpu_lab_nsight_compute.sh --execute
       -> CUDA build sm_120
       -> baseline JSON + Nsight .ncu-rep + .ncu.txt per workload
       -> phase20_nsight_compute_manifest.json
```

## Consolidated architecture at commit 9f169b2

```text
Node1 local camera and artifact plane
  C922 /dev/video0
    -> OpenCV local_capture.py
    -> deterministic frame-difference motion gate
    -> keyframes / clips / overlays / manifests under data/captures
    -> normalized SQLite rows: capture_sessions, capture_artifacts, events, audit_log

Fast visual fact plane
  motion_detected event
    -> optional YOLO ONNX after trigger
    -> object_detected child rows only when detector returns them
    -> detector_health + model_registry

Assistant and explanation plane
  events + artifacts + event contracts + deterministic policy
    -> AssistantSummaryService deterministic summary
    -> optional Gemma/MAX strict JSON summary over structured facts
    -> optional Qwen VLM keyframe analysis after stored artifact
    -> optional SmolVLM2 structured short-clip experiment
    -> FactGuard + source refs -> grounded Q&A, reports, evidence packs

Native non-LLM CPU/CUDA workload plane
  previous/current gray frames + audio/storage metadata
    -> Phase 0 shared C++ contracts/timing/JSON/CUDA helpers
    -> Phase 1 CPU ISP rolling line buffer
    -> Phase 2 CUDA ISP shared-memory tiled filters
    -> Phase 3 sparse ROI crop/resize/normalize
    -> Phase 4 mixed-region connected components/group batching
    -> Phase 5 dense full-frame diff/histogram/reductions
    -> Phase 6 overlay-heavy heatmap/blend/thumbnail
    -> Phase 7 AudioBox RMS/peak/onset/sync drift
    -> Phase 8 storage batch planner/key moments
    -> Phase 9 evidence fingerprint/dedup/safety pipeline

Evidence index and operator plane
  capture-run evidence pipeline
    -> CSV manifest + profile JSON artifact + evidence_pipeline_indexed event
    -> decoded keyframe fingerprints
    -> normalized SQLite index: profiles/fingerprints/dedup/key_moments
    -> summary API
    -> retention + scheduler + rebuild
    -> operator dashboard + inline charts
    -> Prometheus metrics + Grafana dashboard JSON
    -> Nsight Compute profile plan/results for synthetic CUDA workloads
```

## Current code/module map

| Area | Key files | Role |
|---|---|---|
| Core DB/evidence | `monitor_me/db.py`, `migrations/*.sql`, `event_tools.py`, `evidence_pack.py`, `report_tools.py` | SQLite schema authority, event/artifact persistence, evidence packs, incident reports, retention/rebuild/index helpers. |
| Capture and vision | `local_capture.py`, `camera_devices.py`, `yolo_onnx.py`, `yolo_client.py`, `overlays.py`, `detector_health.py` | Real local C922 capture, motion gate, YOLO child rows, raw/overlay artifacts, model health. |
| Assistant safety | `assistant.py`, `fact_guard.py`, `assistant_summary.py`, `event_contract.py`, `capture_policy.py` | Grounded Q&A, non-invention guard, event contracts, deterministic policy, summaries. |
| Optional model clients | `llm_client.py`, `vlm_client.py`, `keyframe_vlm.py`, `smolvlm2_client.py`, `short_clip_vlm.py` | Optional Gemma/MAX, Qwen VLM, and SmolVLM2 local-only guarded paths. |
| Native bridge | `monitor_me/non_llm_gpu_lab.py` | Python bridge to C++/CUDA sidecar and fallback analysis. |
| Native C++/CUDA lab | `native/node1_non_llm_gpu_inference_lab/include`, `src`, `scripts` | CPU/CUDA workload modules, JSON/timing helpers, selftests, profile scripts. |
| API/UI/observability | `routes.py`, `operator_dashboard.py`, `grafana_dashboard.py`, `configs/prometheus`, `configs/grafana` | FastAPI endpoints, local dashboard, Prometheus exposition, Grafana JSON/provisioning. |
| Validation | `tests/*.py`, `scripts/validate_*`, `native/.../scripts/run_node1_gpu_lab_phase*.sh` | Regression, offline validation, phase-specific selftests, native build/selftest wrappers. |

## Phase 20 uploaded Nsight Compute runtime evidence

The extracted source contains untracked Phase 20 results under `results/node1_gpu_lab/nsight_compute/phase20_20260706_212820`. These should not be committed, but they prove the Phase 20 runner produced baseline JSON, `.ncu-rep`, `.ncu.txt`, and a manifest for each planned workload.

| Workload | Kernels observed in `.ncu.txt` | Baseline/parity evidence |
|---|---|---|
| `isp_sobel_mag` | isp_filter_3x3_tiled_kernel, isp_metrics_kernel | ok=true; output_equal=true; mismatch_count=0; max_abs_diff=0; facts_only=true |
| `sparse_roi` | frame_diff_tile_mask_kernel, sparse_roi_resize_normalize_kernel, sparse_roi_stats_kernel | ok=true; rois_equal=true; output_close=true; mismatch_count=0; max_abs_diff=0; facts_only=true |
| `mixed_region` | frame_diff_tile_mask_kernel, mixed_region_batch_resize_normalize_kernel, mixed_region_stats_kernel | ok=true; groups_equal=true; output_close=true; mismatch_count=0; max_abs_diff=0; facts_only=true |
| `dense_full_frame` | frame_diff_tile_mask_kernel, dense_full_frame_diff_hist_normalize kernel | ok=true; histogram_equal=true; normalized_close=true; mismatch_count=0; max_abs_diff=0; facts_only=true |
| `overlay_heavy` | frame_diff_tile_mask_kernel, overlay_heavy_heat_blend_kernel, overlay_thumbnail_kernel | ok=true; heatmap_equal=true; overlay_equal=true; thumbnail_equal=true; mismatch_count=0; max_abs_diff=0; facts_only=true |
| `audiobox` | audiobox_window_kernel, audiobox_correlation_kernel | ok=true; rms_close=true; peaks_close=true; masks_equal=true; drift_equal=true; mismatch_count=0; max_abs_diff≈1.19e-7; facts_only=true |

## Presentation document text view

Use the following as a slide-by-slide talk track. It is intentionally ordered by commit so the audience sees requirement → design → implementation → validation growth.

### Slide 1: Title
- **Headline:** MonitorMe evolution: evidence-first local AI camera assistant to profiled GPU/evidence dashboard platform
- **Talk track:** Position the work as a disciplined progression, not a random set of patches.

### Slide 2: Problem / requirement
- **Headline:** Build a Node1-local CCTV evidence assistant that answers only from local evidence and later grows into GPU-aware evidence indexing and operations.
- **Talk track:** Stress privacy, traceability, deterministic capture, and no invented facts.

### Slide 3: Baseline architecture
- **Headline:** C922 local camera, OpenCV capture, motion gate, YOLO-after-trigger, SQLite events/artifacts, evidence packs/reports/API/CLI.
- **Talk track:** Show commit d751b3b as the first meaningful product baseline.

### Slide 4: Commit 01: d751b3b
- **Headline:** AI system in local infra for Camera used as evidence assistant
- **Talk track:** Requirement/design: Foundation / Step 17A-E. Functional growth: Created the standalone MonitorMe Node1-local evidence assistant: SQLite evidence DB, local C922 capture, frame-difference motion gate, local artifacts/manifests, YOLO ONNX detection after motion, detector health checks, object-query grounding, evidence packs, incident reports, feedback/audit records, FastAPI routes, CLI tools, overlays, model registry, and validation scripts. Non-functional value: Evidence-first and local-only design; raw frames stay local; facts must reference event/session/frame/artifact/model metadata; object rows are only created from detector output; overlays preserve raw keyframes; safety guard rejects unsupported claims; API payloads parsed as JSON bodies; docs and test extras established. Implementation: Python package under monitor_me/*; migration 001 schema; scripts for API/capture/model download; tests for routes, grounding, camera devices, detector health, normalization, overlays, evidence packs, feedback/correlation, YOLO pipeline. Architecture: C922 /dev/video0 -> OpenCV local_capture -> motion gate -> keyframe/manifest artifacts ->     -> events table: motion_detected ->     -> optional YOLO ONNX -> events table: object_detected child rows ->     -> evidence_pack / incident_report / feedback / audit_log ->     -> MonitorMeAssistant + FactGuard -> evidence-backed answer/API/CLI

### Slide 5: Commit 02: da3f66a
- **Headline:** Add Node1 AI Camera Assistant v0.1 event contracts and summaries
- **Talk track:** Requirement/design: Node1 AI Camera Assistant v0.1. Functional growth: Added event contracts, deterministic capture/review policy, automatic assistant summaries after motion/YOLO events, summary and event-contract CLI/API routes, and incident report summary inclusion. Non-functional value: Keeps YOLO as visual-fact producer and Node1 policy as deterministic decision layer; no Gemma yet; summaries are generated only from stored evidence; hallucination/non-invention tests prove unsupported facts are not emitted. Implementation: migration 002 event_contracts; assistant_summary.py; capture_policy.py; event_contract.py; yolo_client.py; DB APIs; CLI/routes extensions; validation script for v0.1. Architecture: motion/object event -> event_contract.py -> event_contracts table ->                    -> capture_policy.py -> policy_decision_json ->                    -> assistant_summary.py -> assistant_summaries table ->                    -> assistant/report/evidence_pack -> operator-facing grounded narrative

### Slide 6: Commit 03: 1ee6a57
- **Headline:** feat: add Node1 AI Camera Assistant v0.2 with validated MAX Gemma startup
- **Talk track:** Requirement/design: Node1 AI Camera Assistant v0.2 / MAX Gemma. Functional growth: Added optional local Gemma 3 1B via MAX as a strict JSON explanation layer; added llm-health, Gemma summary source, deterministic fallback summaries, and live MAX helper scripts for TERM1/TERM2 validation. Non-functional value: Gemma is disabled by default, receives structured facts rather than raw frames, and cannot decide actions. Invalid/unavailable LLM output falls back to deterministic v0.1 summary. Helper scripts avoid accidental Pixi/MAX mutation, preserve Mojo caches, and document RTX 5060 Ti startup constraints. Implementation: llm_client.py; assistant_summary.py strict JSON path; MAX/Pixi scripts under scripts/max; docs/MAX_GEMMA_NODE1.md; tests for MAX helper behavior and v0.2 summaries. Architecture: events + event_contract + deterministic policy + artifact refs ->        -> optional Gemma/MAX OpenAI-compatible /v1/chat/completions ->        -> strict JSON validator ->        -> valid: assistant_summaries(summary_source=gemma_max) ->        -> invalid/unavailable: deterministic fallback summary + audit reason

### Slide 7: Commit 04: 1138403
- **Headline:** Add Node1 Assistant v0.3 Qwen VLM keyframe analysis
- **Talk track:** Requirement/design: Node1 AI Camera Assistant v0.3 / Qwen VLM keyframe analysis. Functional growth: Added optional Qwen VLM keyframe analysis after a trigger and after a stored keyframe artifact exists; added VLM health, manual VLM analyze/list commands and API routes. Non-functional value: Disabled by default, local-only by default, remote endpoint rejected unless explicitly allowed. VLM output is companion context, not detector truth, cannot create YOLO rows, and is validated against identity/intent/threat/weapon/person-profile claims. Implementation: migration 003 vlm_keyframe_analyses; vlm_client.py; keyframe_vlm.py; local_capture integration; routes/CLI; model registry; v0.3 validation tests. Architecture: motion/YOLO trigger -> stored keyframe artifact ->        -> optional QwenVLMOpenAIClient ->        -> strict keyframe JSON with event_id/artifact_id citations ->        -> vlm_keyframe_analyses table ->        -> assistant may read as companion visual context, not detector fact

### Slide 8: Commit 05: 8fcd276
- **Headline:** fix: constrain SmolVLM2 short clip output with structured JSON schema
- **Talk track:** Requirement/design: Node1 AI Camera Assistant v0.4.1 / SmolVLM2 structured clip experiment. Functional growth: Constrained SmolVLM2 short-clip output using vLLM structured_outputs JSON schema; introduced bounded visual-state fields and health/analyze/list routes and CLI. Non-functional value: Fixes freeform/truncated/unsafe VLM output risk; disabled by default and local-only; const-binds event_id and artifact_id; rejects identity, profiling, intent, threat, weapon, and unsupported evidence-ID claims. Implementation: migration 004 smolvlm2_clip_experiments; smolvlm2_client.py; short_clip_vlm.py; fact_guard updates; local_capture clip artifact path; tests and validation scripts for v0.4/v0.4.1. Architecture: trigger event + local short clip artifact ->        -> SmolVLM2 OpenAI-compatible request with structured_outputs={json:schema} ->        -> bounded fields: visible_scene/person_like_presence/vehicle_like_presence/motion_claim/safe_observation/unsupported_claims ->        -> smolvlm2_clip_experiments table ->        -> safe experimental companion output

### Slide 9: Commit 06: d814a12
- **Headline:** Add Node1 non-LLM C++ CUDA GPU workload optimization module
- **Talk track:** Requirement/design: Native Node1 non-LLM GPU workload lab v0.1. Functional growth: Added optional C++/CUDA sidecar for deterministic CPU/GPU workload profiling; analyzes previous/current frames, builds tile masks, classifies sparse/mixed/dense paths, emits gpu_workload_profiled child evidence rows, and exposes gpu-lab-health/gpu-lab-synthetic. Non-functional value: Runs only behind --gpu-lab-enabled after local motion/keyframe trigger. Emits workload metadata only; no identity/object/person/intent/weapon claims. Python bridge has safe fallback. Native build outputs ignored. Implementation: native/node1_non_llm_gpu_inference_lab CMake project; gpu_lab_cpu.cpp, gpu_lab_cuda.cu, gpu_lab_main.cpp; monitor_me/non_llm_gpu_lab.py; local_capture integration; docs and smoke scripts. Architecture: capture-run trigger -> previous/current frames ->        -> Python bridge -> native C++/CUDA sidecar ->        -> CPU/CUDA tile mask + changed ratio + path sparse|mixed|dense ->        -> gpu_workload_profiled child event with privacy facts ->        -> capture manifest references workload profile

### Slide 10: Commit 07: b7518be
- **Headline:** refactor: harden Node1 non-LLM GPU lab Phase 0
- **Talk track:** Requirement/design: GPU lab Phase 0 hardening. Functional growth: Refactored native sidecar into reusable structs, route helpers, validation, JSON serialization, host timing, CUDA error/event timing, and self-test executable. Non-functional value: Established stable contracts before adding ISP/ROI/dense/overlay/audio/storage modules; Python fallback validation now mirrors native rules; timing facts added without semantic claims. Implementation: gpu_lab_types.hpp/cpp; gpu_lab_json.hpp/cpp; gpu_lab_timing.hpp/cpp; gpu_lab_cuda_utils.cuh/cu; gpu_lab_selftest.cpp; Phase 0 pytest and script. Architecture: native CLI modes -> shared config/types validation ->               -> CPU/CUDA kernels ->               -> shared JSON emitters + timing ->               -> selftest coverage -> Python fallback -> same tile-contract validation rules

### Slide 11: Commit 08: ef6f6b2
- **Headline:** Add CPU ISP rolling line-buffer filters to Node1 GPU lab
- **Talk track:** Requirement/design: GPU lab Phase 1 CPU ISP rolling line-buffer filters. Functional growth: Added CPU-only 3x3 ISP filters: blur, sharpen, edge/conv-edge, sobel-x, sobel-y, sobel-mag; added P5/P6 PGM/PPM I/O and gpu-lab-isp-synthetic CLI. Non-functional value: Implements true rolling 3-row buffer for memory-locality baseline; facts-only schema reports image processing metrics, not visual semantics; CUDA intentionally deferred to Phase 2. Implementation: isp_filters.hpp; isp_filters_cpu.cpp; isp_pnm_io.cpp; native isp-synthetic/isp-pgm modes; tests for rolling output vs naive reference and PGM/PPM path. Architecture: PGM/PPM or synthetic gray image ->        -> rolling line buffer y-1/y/y+1 ->        -> 3x3 CPU filter ->        -> output metrics: mean/edge/focus/noise/lighting/saturation ->        -> facts-only ISP JSON

### Slide 12: Commit 09: 6de5e80
- **Headline:** Add CUDA ISP filters and CPU parity metrics to Node1 GPU lab
- **Talk track:** Requirement/design: GPU lab Phase 2 CUDA ISP filters. Functional growth: Added shared-memory tiled CUDA versions of the Phase 1 ISP filters and CPU-vs-CUDA parity metrics. Non-functional value: CUDA uses 16x16 output tile plus 1-pixel halo; comparison proves output/metrics parity with mismatch/max-diff fields; still facts-only. Implementation: CUDA ISP kernels in gpu_lab_cuda.cu; isp_cuda JSON fields; gpu-lab-isp-synthetic wrapper; Phase 2 CUDA selftest and pytest. Architecture: gray image -> CPU rolling ISP reference ->           -> CUDA shared tile [16+2][16+2] + 3x3 stencil ->           -> isp + isp_cuda ->           -> isp_cpu_cuda_comparison(output_equal, metrics_close, facts_only)

### Slide 13: Commit 10: 35ce5aa
- **Headline:** Add sparse ROI crop resize normalize path
- **Talk track:** Requirement/design: GPU lab Phase 3 sparse ROI crop/resize/normalize. Functional growth: Added active-tile walking, ROI rectangle generation, grayscale crop, nearest-neighbor resize, uint8-to-float32 normalization, CUDA ROI kernel, and CLI/native modes. Non-functional value: Optimizes sparse activity path instead of full-frame processing; CPU/CUDA comparison validates ROI equality/output closeness; no semantic object/person claims. Implementation: sparse_roi.hpp/cpp; CUDA sparse_roi_resize_normalize and stats kernels; scripts for CPU/CUDA selftests; pytest for CPU, CUDA parity when available, and CLI wrapper. Architecture: tile mask with few active tiles ->        -> active tile walking ->        -> ROI list ->        -> crop/resize/normalize CPU + CUDA ->        -> sparse_roi_cpu_cuda_comparison(rois_equal, output_close)

### Slide 14: Commit 11: a222d19
- **Headline:** Add mixed region connected tile grouping path
- **Talk track:** Requirement/design: GPU lab Phase 4 mixed-region connected tile grouping. Functional growth: Added 4-neighbor connected components over active tiles, contiguous/scattered classification, grouped crop batching, CUDA grouped crop/resize/normalize, and CLI/native modes. Non-functional value: Handles medium activity where ROI-by-tile is inefficient but dense processing is unnecessary; emits grouping and normalization facts only; CPU/CUDA groups and outputs must match. Implementation: mixed_region.hpp/cpp; CUDA mixed region kernels; CPU/CUDA scripts for contiguous/scattered/dense scenarios; tests for parity and CLI. Architecture: tile mask with medium activity ->        -> 4-neighbor connected components ->        -> group bounding boxes + classification contiguous|scattered ->        -> grouped crop/resize/normalize CPU + CUDA ->        -> mixed_region_cpu_cuda_comparison(groups_equal, output_close)

### Slide 15: Commit 12: a15fc38
- **Headline:** Add dense full-frame diff histogram path
- **Talk track:** Requirement/design: GPU lab Phase 5 dense full-frame diff/histogram/reduction. Functional growth: Added high-activity dense full-frame path: absolute diff, changed-pixel reduction, 256-bin histogram, previous/current/diff reductions, lighting delta, full-frame normalization, CUDA path, and CLI. Non-functional value: Designed for dense route where sparse ROI wastes bandwidth; validates histogram/reduction/normalization parity; facts-only high-activity numeric metrics. Implementation: dense_full_frame.hpp/cpp; CUDA dense diff/hist/reduction kernels; selftest scripts; tests/test_node1_dense_full_frame_phase5.py. Architecture: previous/current gray full frame ->        -> abs diff + changed-pixel reduction ->        -> 256-bin histogram + reductions + lighting delta ->        -> dense normalize float32 ->        -> dense_full_frame_cpu_cuda_comparison(histogram_equal, reductions_close, normalized_close)

### Slide 16: Commit 13: 3ab08bb
- **Headline:** Add overlay-heavy alpha blend heatmap path
- **Talk track:** Requirement/design: GPU lab Phase 6 overlay-heavy alpha/heatmap/thumbnail. Functional growth: Added visualization-heavy path: full-frame diff, motion heatmap, deterministic heat coloring, alpha blend against current frame, RGB overlay, thumbnail generation, before/after metrics, CUDA path, and CLI. Non-functional value: Optimizes artifact-generation workload without semantic interpretation; ensures heatmap/overlay/thumbnail parity; local facts-only visualization metadata. Implementation: overlay_heavy.hpp/cpp; CUDA heatmap/blend/thumbnail kernels; Phase 6 selftests and pytest. Architecture: previous/current gray ->        -> diff -> motion heatmap ->        -> alpha blend -> RGB overlay ->        -> thumbnail ->        -> overlay_heavy_cpu_cuda_comparison(heatmap_equal, overlay_equal, thumbnail_equal)

### Slide 17: Commit 14: dab7f14
- **Headline:** Add audiobox rms peak sync drift path
- **Talk track:** Requirement/design: GPU lab Phase 7 AudioBox RMS/peak/sync drift. Functional growth: Added numeric audio workload path: per-window RMS/peak, silence mask, onset mask, bounded cross-correlation, sync drift in samples/ms, CUDA path, and CLI. Non-functional value: No speech transcription, speaker identity, behavior, intent, or semantic audio claims; validates masks, drift, RMS/peak/correlation parity. Implementation: audiobox.hpp; audiobox_cpu.cpp; CUDA audiobox_window and correlation kernels; native mode audiobox-synthetic; tests/test_node1_audiobox_phase7.py. Architecture: primary/reference float32 audio ->        -> RMS/peak windows ->        -> silence/onset masks ->        -> bounded cross-correlation [-max_lag,+max_lag] ->        -> sync drift samples/ms ->        -> audiobox_cpu_cuda_comparison(masks_equal, drift_equal, rms_close)

### Slide 18: Commit 15: 8103b56
- **Headline:** Add storage batch planner and clip sampler path
- **Talk track:** Requirement/design: GPU lab Phase 8 storage batch planner / clip sampler. Functional growth: Added facts-only clip manifest scanner, priority scoring, constrained batch read plan, key moment selection with minimum gap, timeline features, synthetic/manifest modes, and CLI. Non-functional value: No media decoding and no visual/audio semantic claims; supports storage/batch-read optimization and deterministic key-moment selection. Implementation: storage_batch.hpp/cpp; storage-batch-synthetic and storage-batch-manifest native modes; tests/test_node1_storage_batch_phase8.py. Architecture: clip manifest CSV ->        -> metadata scan ->        -> priority scoring ->        -> constrained batch read plan ->        -> key moments with min gap ->        -> storage_batch facts: bytes/timeline/gaps/scores

### Slide 19: Commit 16: cc374ed
- **Headline:** Add evidence fingerprint dedup pipeline
- **Talk track:** Requirement/design: GPU lab Phase 9 evidence fingerprint/dedup pipeline. Functional growth: Added deterministic evidence-pipeline workload: storage planner reuse, synthetic visual fingerprints, ahash/dhash/fingerprint/histogram facts, duplicate grouping, dedup-aware key-moment selection, latency/throughput counters, and safety validator. Non-functional value: Does not decode media in this phase and emits no semantic visual/audio claims; validates manifest counts, batch constraints, fingerprint shape, dedup accounting, key-moment spacing, and timeline totals. Implementation: evidence_pipeline.hpp/cpp; evidence-pipeline-synthetic/manifest native modes; Python bridge CLI; Phase 9 selftest and pytest. Architecture: clip manifest/timeline metadata ->        -> storage batch planner ->        -> deterministic fingerprints + histograms ->        -> Hamming dedup groups ->        -> dedup-aware key moments ->        -> latency/throughput + safety validator ->        -> evidence_pipeline JSON

### Slide 20: Commit 17: 6b3092b
- **Headline:** Integrate capture-run evidence pipeline
- **Talk track:** Requirement/design: GPU lab Phase 10 capture-run evidence pipeline integration. Functional growth: Integrated native evidence pipeline into real capture-run after local motion keyframes; writes evidence CSV manifest, native profile JSON artifact, evidence_pipeline_indexed session event, and final manifest references. Non-functional value: Facts-only, post-capture metadata path; no media upload, no identity/intent/person/object claims; native pipeline runs from local artifact metadata. Implementation: local_capture evidence-pipeline config/bridge; non_llm_gpu_lab.py runner additions; CLI options --evidence-pipeline-enabled and tuning options; Phase 10 selftest. Architecture: capture-run keyframes ->        -> capture manifest frame records ->        -> evidence_pipeline_manifest.csv ->        -> native evidence-pipeline-manifest ->        -> evidence_pipeline_profile.json artifact ->        -> evidence_pipeline_indexed event ->        -> final manifest references both artifacts and event

### Slide 21: Commit 18: 6f098fc
- **Headline:** Ingest decoded keyframe fingerprints
- **Talk track:** Requirement/design: GPU lab Phase 11 decoded keyframe fingerprints. Functional growth: Added local decoded-keyframe fingerprint generation: ahash64, dhash64, combined fingerprint64, decoded dimensions, and 16-bin luminance histogram written into evidence CSV and ingested by native pipeline. Non-functional value: Decoding happens locally after keyframe already exists as evidence; still facts-only, no object/person/identity/intent claims; supports fallback metadata_synthetic fingerprints if real decode unavailable. Implementation: local_capture decode/fingerprint helpers; native manifest fields fingerprint_source/decoded_width/height/hash/histogram16; Python/native tests. Architecture: stored keyframe JPEG ->        -> local grayscale decode ->        -> resize to fingerprint grid ->        -> ahash64 + dhash64 + fingerprint64 + histogram16 ->        -> CSV manifest extended columns ->        -> native evidence_pipeline real_media_ingestion facts

### Slide 22: Commit 19: 499a567
- **Headline:** Persist evidence pipeline index
- **Talk track:** Requirement/design: GPU lab Phase 12 evidence index persistence. Functional growth: Persisted evidence-pipeline profile output into normalized SQLite tables: profiles, fingerprints, dedup groups, and key moments; added CLI list/readback commands. Non-functional value: Moves from large JSON-only artifact to queryable index while keeping source event/artifact; stores hashes, histograms, duplicate accounting, timeline and safety facts only. Implementation: migration 005; MonitorMeDB persist/list helpers; local_capture calls persist after successful pipeline; tests/test_node1_evidence_index_phase12.py. Architecture: evidence_pipeline_profile.json + evidence_pipeline_indexed event ->        -> persist_evidence_pipeline_index ->        -> evidence_pipeline_profiles ->        -> evidence_fingerprints ->        -> evidence_dedup_groups ->        -> evidence_key_moments ->        -> CLI/API-ready query index

### Slide 23: Commit 20: d7a8a55
- **Headline:** Expose evidence pipeline summary API
- **Talk track:** Requirement/design: GPU lab Phase 13 evidence pipeline summary API. Functional growth: Exposed persisted evidence index through local FastAPI summary/detail endpoints for summaries, session summary, profile summary, fingerprints, dedup groups, and key moments. Non-functional value: API is read-only, bounded, local, and facts-only; no media decode, raw-frame upload, native rerun, or semantic claims during API calls. Implementation: routes.py evidence/pipeline endpoints; DB summary constructors; docs/API; Phase 13 selftest and pytest. Architecture: SQLite evidence index tables ->        -> DB summary helpers ->        -> /evidence/pipeline/summaries ->        -> /sessions/{id}/summary / profiles/{id}/summary ->        -> bounded fingerprints/dedup/key-moments endpoints ->        -> operator/report/dashboard consumers

### Slide 24: Commit 21: e11363e
- **Headline:** Add evidence index retention policies
- **Talk track:** Requirement/design: GPU lab Phase 14 evidence index retention/compaction. Functional growth: Added retention planning/apply for old normalized evidence index rows with keep-last-per-camera/session guards; added retention run audit table, CLI, and API. Non-functional value: Deletes only normalized index rows; retains source events, capture artifacts, keyframes, CSV manifests, JSON profile artifacts, and reportability. Destructive apply requires explicit confirmation; supports dry-run, WAL checkpoint, and optional VACUUM. Implementation: migration 006 evidence_retention_runs; MonitorMeDB plan/apply/list APIs; CLI evidence-retention-*; routes retention endpoints; tests/test_node1_evidence_retention_phase14.py. Architecture: evidence_pipeline_profiles + child index rows ->        -> retention policy older_than_days + keep-last guards ->        -> dry-run plan or confirmed apply ->        -> delete normalized index rows only ->        -> evidence_retention_runs audit ->        -> retained artifacts remain rebuildable

### Slide 25: Commit 22: 23286dc
- **Headline:** Add operator evidence dashboard
- **Talk track:** Requirement/design: GPU lab Phase 15 operator evidence dashboard. Functional growth: Added local read-only operator dashboard and /operator/dashboard/data JSON context over evidence index, retention audit rows, selected profile details, and links to JSON evidence APIs. Non-functional value: No external assets, media decode, capture run, destructive retention, identity/intent/speech inspection, or semantic claims from dashboard request path. Implementation: operator_dashboard.py; routes /operator/dashboard and /operator/dashboard/data; docs/API and native docs; tests/test_node1_operator_dashboard_phase15.py. Architecture: SQLite evidence index + retention runs ->        -> build_operator_dashboard_context ->        -> /operator/dashboard/data JSON ->        -> /operator/dashboard local HTML ->        -> read-only evidence counts, safety, key moments, dedup, retention history

### Slide 26: Commit 23: 7ba029f
- **Headline:** Automate evidence index retention scheduling
- **Talk track:** Requirement/design: GPU lab Phase 16 scheduled retention automation. Functional growth: Added configurable local evidence retention scheduler with default disabled dry-run schedule, CLI/API configuration, scheduler run audit rows, and systemd timer/service scripts. Non-functional value: Disabled by default and dry-run by default; destructive scheduled retention requires --apply --yes or API confirm=true; delegates to Phase 14 retention, so scope remains normalized index rows only. Implementation: migration 007 schedule and scheduler_runs; MonitorMeDB scheduler APIs; CLI evidence-retention-schedule-*; routes schedule endpoints; scripts/systemd; run_evidence_retention_scheduler_once.sh; pytest. Architecture: evidence_retention_schedule(default disabled, dry-run) ->        -> manual/cron/systemd scheduler runner ->        -> due/skipped/forced decision ->        -> Phase 14 apply_evidence_index_retention ->        -> evidence_retention_scheduler_runs audit

### Slide 27: Commit 24: 6c6a9a9
- **Headline:** Rebuild evidence index from retained artifacts
- **Talk track:** Requirement/design: GPU lab Phase 17 evidence index rebuild from retained artifacts. Functional growth: Added plan/apply rebuild workflow that scans retained evidence_pipeline_indexed events and retained evidence_pipeline_profile JSON artifacts, then rehydrates normalized SQLite index rows. Non-functional value: Closes retention loop without rerunning CUDA/native pipeline, decoding images, reading pixels, uploading frames, or making new semantic claims. Non-dry-run and replace-existing require explicit confirmation. Implementation: migration 008 evidence_index_rebuild_runs; DB rebuild APIs and get_artifact; CLI evidence-index-rebuild-*; API rebuild endpoints; Phase 17 tests. Architecture: retained evidence_pipeline_indexed event ->        -> retained evidence_pipeline_profile artifact ->        -> read profile JSON result.evidence_pipeline ->        -> rebuild evidence_pipeline_profiles/fingerprints/dedup/key_moments ->        -> evidence_index_rebuild_runs audit

### Slide 28: Commit 25: 200fad8
- **Headline:** Add operator dashboard charts
- **Talk track:** Requirement/design: GPU lab Phase 18 richer operator dashboard charts. Functional growth: Added chart-ready facts and inline HTML/SVG dashboard charts: profile points, fingerprint composition, latency breakdown, key moment timeline, Hamming sample, operation audit, and safety checks. Non-functional value: Charts are derived only from persisted SQLite rows and audit rows; inline CSS/SVG only; no external JS/CSS/chart libraries/assets, media decode, native rerun, or semantic claims. Implementation: operator_dashboard.py chart builders/renderers; route data includes charts; docs and tests/test_node1_operator_dashboard_charts_phase17.py. Architecture: operator dashboard context ->        -> charts: profile_points/fingerprint_composition/latency/key_moments/hamming/audit/safety ->        -> inline HTML + SVG ->        -> richer local UI without external dependencies

### Slide 29: Commit 26: 54b6fb9
- **Headline:** Add Grafana operator dashboard integration
- **Talk track:** Requirement/design: GPU lab Phase 19 Grafana/dashboard integration. Functional growth: Exposed operator dashboard facts as Prometheus text metrics and a Grafana dashboard JSON model; added Prometheus and Grafana provisioning config files. Non-functional value: Reuses Phase 15-18 context, avoiding second interpretation layer. Metrics/Grafana remain facts-only/local-only: no media decode, raw-frame upload, native rerun, external calls, destructive actions, identity/intent/speech/danger claims. Implementation: grafana_dashboard.py; routes /operator/dashboard/metrics and /operator/dashboard/grafana/dashboard.json; configs/prometheus and configs/grafana; Phase 19 tests. Architecture: SQLite evidence index + audit rows ->        -> build_operator_dashboard_context ->        -> /operator/dashboard/metrics Prometheus exposition ->        -> Prometheus scrape ->        -> Grafana dashboard JSON/provisioning ->        -> local operator observability panels

### Slide 30: Commit 27: 9f169b2
- **Headline:** Add Nsight Compute profiling pass
- **Talk track:** Requirement/design: GPU lab Phase 20 Nsight Compute profiling pass. Functional growth: Added reproducible Nsight Compute profiling workflow covering Phase 2-7 CUDA workloads: ISP Sobel, sparse ROI, mixed region, dense full-frame, overlay-heavy, and AudioBox. Supports dry-run, execute, workload selection, output-dir, ncu set/metrics, no-build, baseline JSON, .ncu-rep, .ncu.txt, and manifest output. Non-functional value: Profiles synthetic native workloads only; separate from API/dashboard/capture paths; output goes under results/ and must not be staged. Build targets CUDA 13.3, sm_120, RTX 5060 Ti environment. CI can dry-run without GPU/Nsight. Implementation: configs/nsight_compute/node1_gpu_lab_phase20_profile_plan.json; profile_node1_gpu_lab_nsight_compute.py/.sh; Phase 20 selftest and pytest. Architecture: Phase 2-7 CUDA synthetic workloads ->        -> profile plan JSON ->        -> profile_node1_gpu_lab_nsight_compute.sh --execute ->        -> CUDA build sm_120 ->        -> baseline JSON + Nsight .ncu-rep + .ncu.txt per workload ->        -> phase20_nsight_compute_manifest.json

### Slide 31: Final system architecture
- **Headline:** The top commit combines local evidence capture, optional guarded assistant models, C++/CUDA workload modules, normalized evidence index, retention/rebuild, dashboard/Grafana, and Nsight profiling.
- **Talk track:** End with the consolidated architecture diagram above.

### Slide 32: Validation story
- **Headline:** Commit messages record phase-specific validation. Current sandbox review passed compileall and pytest with native binary tests skipped because build artifacts were absent. Uploaded Phase 20 results include successful baseline/parity JSON and Nsight report files.
- **Talk track:** Explain that native CUDA validation was performed on Node1 and runtime results were kept untracked as intended.

### Slide 33: CTO/stakeholder close
- **Headline:** The product is not merely “LLM over camera.” It is evidence-first, privacy-preserving, deterministic, GPU-aware, observable, and testable.
- **Talk track:** This is the strategic message: every AI layer is subordinate to traceable local facts and measurable workload paths.

## Recommended narrative for stakeholder presentation

1. Start with the safety/evidence contract: MonitorMe answers only with local evidence references. This reduces hallucination risk before adding model capability.
2. Show that models are layered by responsibility: YOLO gives fast visual facts; deterministic policy decides severity/actions; Gemma/Qwen/SmolVLM2 are optional explanation/context layers with strict validation.
3. Show the non-LLM GPU lab as a performance-engineering track: frame activity is measured, routed, processed by specialized CPU/CUDA paths, and validated by CPU-vs-CUDA parity.
4. Show the evidence-index lifecycle: capture integration, decoded fingerprints, normalized persistence, APIs, retention/scheduler, rebuild, dashboard, Grafana, and profiling.
5. End with Phase 20: Nsight Compute profiling closes the engineering loop from feature implementation to measurable GPU kernel analysis.