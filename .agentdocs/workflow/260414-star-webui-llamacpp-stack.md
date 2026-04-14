# STAR Web UI Plan Using `llama.cpp` Server Patterns

## Decision Summary

Yes: the `llama.cpp` server approach is a good fit for STAR, but only at the infrastructure level.

Use the same native C/C++ HTTP server + JSON API + static web assets pattern, not the LLM-specific product surface. For STAR, this should become a local job runner and results browser around `STAR.exe`, not an in-process REST rewrite of STAR internals.

## Why This Fits STAR

- STAR is already a native CLI with clear top-level run modes in `source/STAR.cpp:85-133` (`alignReads`, `genomeGenerate`, `soloCellFiltering`).
- STAR already emits file-based state that a UI can consume without invasive algorithm changes:
  - `Log.out` and `Log.std.out` in `source/Parameters.cpp:369-389`
  - `Log.progress.out` for `alignReads` in `source/Parameters.cpp:584-585`
  - `Log.final.out`, `Solo.out/*`, `SJ.out.tab`, BAM/SAM outputs in multiple existing code paths
- The repo already has a report exporter in `scripts/starsolo_qc_report.sh:1-120`, so a web UI can build on existing output conventions rather than inventing a new reporting format.
- The build already accepts vendored/fetched dependencies in `source/CMakeLists.txt:1-110`, so adding a thin server stack is operationally consistent.

## What To Copy From `llama.cpp`

- Single native binary serving HTTP + static assets
- `cpp-httplib` + `nlohmann/json` style lightweight stack
- `--host`, `--port`, `--path` style server options
- `GET /health`, `GET /props`, optional `GET /metrics`
- Browser UI served by the executable, with no Node/Electron runtime

## What Not To Copy

- OpenAI-compatible APIs
- Slot management, prompt cache, batching, token streaming machinery
- Multi-tenant request concurrency assumptions
- Heavy frontend build pipelines
- Electron or embedded browser runtimes

STAR jobs are heavy CPU, RAM, and disk workloads. The server should default to one active job at a time plus a small queue.

## Binary Size Rule

Treat binary growth as a hard constraint for this feature.

Rules:
- Keep the server stack header-only or near-header-only where practical.
- Prefer plain HTML, CSS, and JS assets over React, Vue, or other compiled SPA frameworks.
- Do not add Electron, WebView runtimes, or Node-based desktop packaging.
- Do not bundle large font packs, icon packs, or chart libraries unless they are clearly justified.
- Prefer server-side generated JSON plus simple browser rendering over shipping a large frontend framework.
- If asset size becomes noticeable, allow serving files from a sibling `webui/` directory instead of embedding everything into `STAR.exe`.

Acceptance target:
- Keep `STAR.exe` growth modest and review binary size as part of every WebUI phase.
- If a phase causes a noticeable size jump, require an explicit justification and compare embedded-assets vs sidecar-assets packaging.

## Recommended Architecture

### Backend shape

Add a web mode to the main binary:

```text
STAR.exe --runMode webui --host 127.0.0.1 --port 8080
```

Implementation detail: the web server should spawn child `STAR.exe` processes for actual jobs instead of calling the current pipeline in-process.

Why:
- STAR still has broad process-global state and file-oriented lifecycle.
- Subprocess isolation makes cancel/retry/log tailing much simpler.
- A crashed run should not take down the server.

### Suggested endpoints

- `GET /health` => server readiness
- `GET /props` => version, build info, supported run modes, platform limits
- `POST /jobs` => submit `alignReads`, `genomeGenerate`, or `soloCellFiltering`
- `GET /jobs/{id}` => status, command preview, output directory, timestamps
- `POST /jobs/{id}/cancel` => terminate the child process and mark job failed/cancelled
- `GET /jobs/{id}/logs` => merged `Log.out`, `Log.progress.out`, `Log.final.out`
- `GET /jobs/{id}/artifacts` => discover output files and generated reports
- `POST /jobs/{id}/report/starsolo` => generate HTML QC from STARsolo outputs
- `GET /metrics` => optional Prometheus-style process/job metrics

### UI scope

Phase 1 UI should be form-driven, not a general file manager:

- Launch `genomeGenerate`
- Launch `alignReads`
- Launch `soloCellFiltering`
- Show live status/log tail
- Show artifact links when complete
- Render STARsolo QC/report links when `Solo.out` exists

Use plain HTML/CSS/JS with bundled static assets. Do not introduce React/Vite unless the UI outgrows simple forms and polling.

## Implementation Phases

### Phase 0: Server skeleton

- Add a new server module under `source/webui/`
- Add vendored or fetched `cpp-httplib` and `nlohmann/json`
- Wire `--runMode webui`, `--host`, `--port`, `--path`, `--metrics`
- Implement `GET /health` and `GET /props`
- Record baseline `STAR.exe` size before and after this phase

### Phase 1: Job runner

- Add a job registry with states: queued, running, succeeded, failed, cancelled
- Launch child `STAR.exe` processes with validated argument sets
- Restrict writes to explicit output directories
- Tail `Log.out`, `Log.progress.out`, and `Log.final.out`

### Phase 2: Minimal UI

- Add pages for new job, job list, job detail, and output browser
- Display command preview before launch
- Surface Windows-specific limitations from `README.md` and `source/Parameters.cpp`
- Keep Phase 2 assets framework-free unless a concrete limitation appears

### Phase 3: STARsolo integration

- Detect `Solo.out` outputs and offer report generation
- Replace the current shell-report dependency with an internal or bundled-asset implementation
- Keep the report offline-capable; do not depend on CDN assets
- Prefer tiny local charting or server-rendered summaries over large JS visualization bundles

### Phase 4: Hardening

- Add cancellation cleanup for temp and output files
- Add path validation and local-only default binding
- Add basic metrics and bounded queueing
- Add crash recovery for interrupted server restarts

## Verification Plan

- Unit tests for argument construction, path validation, and job-state transitions
- CTest integration test: start server, hit `/health`, submit a tiny smoke job, verify artifacts
- Windows CI lane for `STAR.exe --runMode webui`
- Manual validation on a STARsolo sample to confirm live logs and report generation
- Track release-binary size before and after each phase

## WebUI Test Plan

### Unit tests

Add focused tests for pure server logic before adding broader end-to-end coverage:

- CLI parsing for `--runMode webui`, `--host`, `--port`, `--path`, `--metrics`
- Request validation for allowed run modes and required arguments
- Command-line construction for child `STAR.exe` launches
- Path normalization and path traversal rejection
- Job-state transitions: queued -> running -> succeeded, failed, cancelled
- Log discovery and log-merging logic for `Log.out`, `Log.progress.out`, and `Log.final.out`
- Artifact discovery for `Solo.out`, BAM, SAM, `SJ.out.tab`, and generated reports
- Report-trigger gating so STARsolo report actions only appear when expected outputs exist

Prefer these as `doctest` cases under the existing `test/` CMake lane instead of introducing a second test framework.

### Component tests

Add process-level tests around small isolated helpers:

- Start and stop the HTTP server on an ephemeral port
- Probe `GET /health` and `GET /props`
- Submit a job into a fake or stubbed runner
- Cancel a running stub job and verify final state
- Verify concurrent submissions are queued or rejected according to configured limits

These tests should avoid full genome mapping and use stub runners where possible.

### Integration tests

Add a minimal end-to-end CTest flow for real execution:

- Launch `STAR.exe --runMode webui` in the background
- Wait for `/health`
- Submit one tiny smoke job using existing small fixtures
- Poll `/jobs/{id}` until completion
- Verify `Log.out` and `Log.final.out` are exposed through the API
- Verify artifact listing contains the expected output files
- If the fixture is STARsolo-capable, verify report generation succeeds and returns a local HTML artifact

### Windows-specific coverage

Because this feature is primarily motivated by local Windows usage, add explicit Windows checks for:

- Child-process creation and termination
- Quoting of paths with spaces
- Port binding failures
- Output directory permission failures
- Cleanup behavior after cancel or crash

### Non-goals for first test wave

- Browser automation for every UI detail
- Multi-user or internet-facing deployment tests
- Large real-dataset regression runs inside the WebUI test lane

Keep first-wave WebUI tests fast and deterministic. Full STAR validation remains in the existing smoke and release-comparison flows.

## Risks And Constraints

- In-process reuse of STAR internals is the wrong first move; subprocess execution is safer.
- Parallel heavy jobs can exhaust RAM and disk quickly; concurrency must be conservative.
- Quoting and path handling on Windows must be tested explicitly.
- Remote exposure should be out of scope initially; bind to `127.0.0.1` by default.

## Recommendation

Proceed with a `llama.cpp`-style native server wrapper for STAR.

Best choice:
- self-contained C++ server
- bundled static web UI
- child-process execution model

Not recommended:
- Electron
- Python or Flask sidecar as the primary product path
- rewriting STAR's existing run pipeline into an in-process async service in the first iteration
