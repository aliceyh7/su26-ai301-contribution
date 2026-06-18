# Contribution [1]: llama-server hot swapping cvectors via API like LoRA adapters

**Contribution Number:** [1]  
**Student:** [Alice]  
**Issue:** https://github.com/ggml-org/llama.cpp/issues/10685  
**Status:** Phase III — feature implemented on `fix-issue-10685`, tests + docs added, preparing PR

---

## Why I Chose This Issue

I chose llama.cpp issue #10685, "Feature Request: llama-server hot swapping cvectors via API like we can do with LoRA adapters now," because it connects directly to my interest in LLM systems, inference infrastructure, and model steering. The issue proposes exposing control-vector loading/unloading and scaling through the `llama-server` API, similar to the existing LoRA adapter API, so users can adjust model behavior without restarting the server.

This issue stood out to me because it has both practical product value and strong technical learning value. From reading the issue thread, I understand that control vectors can already be used through command-line flags, but changing their scale or toggling them currently requires restarting the server. A useful contribution would be to add API endpoints such as `GET /cvectors` and `POST /cvectors`, following the existing `/lora-adapters` pattern. This matches my interest in post-training-adjacent workflows, runtime model adaptation, and LLM serving systems, while giving me a chance to work in a widely used C/C++ AI codebase.

I also liked that the issue is labeled `enhancement` and `good first issue`, has a clear user motivation, and provides a concrete possible implementation. The main risk is that someone previously commented that they planned to work on it, so I plan to confirm that there is no active PR or assignee before fully committing.

---

## Understanding the Issue

### Problem Description

Control vectors ("cvectors") are files that steer a model's behavior/style at runtime. In `llama-server` they can only be loaded once, when the server starts, via command-line flags (`--control-vector`, `--control-vector-scaled`). There is no way to view, add, remove, or rescale them while the server is running. The nearly identical LoRA-adapter feature *can* be managed live through the API, so this is an inconsistency the issue asks to close.

### Expected Behavior

`llama-server` should expose `GET /cvectors` (list the loaded control vectors with their ids, paths, and scales) and `POST /cvectors` (set new scales / toggle them), mirroring the existing `GET`/`POST /lora-adapters` endpoints — so users can adjust control vectors live, without restarting the server.

### Current Behavior

No `/cvectors` endpoint exists — requests return HTTP 404. The only way to change a control vector or its scale is to stop the server and restart it with different command-line flags.

### Affected Components

- `tools/server/server.cpp` — where HTTP routes are registered (the `/lora-adapters` routes live here)
- `tools/server/server-task.cpp` / `server-common.cpp` — request parsing and server-side state/handling (the `/lora-adapters` logic lives here)
- `common/common.cpp` — where control vectors are loaded and combined at startup (`common_control_vector_load`)
- `include/llama.h` — the `llama_set_adapter_cvec` API that applies a control vector to a running context

---

## Reproduction Process

### Environment Setup

Platform: macOS (Apple Silicon / arm64).

Challenges I hit and how I solved them:

1. **`cmake` was not installed**, so the build failed immediately. Fixed with Homebrew:
   ```bash
   brew install cmake   # got cmake 4.3.3
   ```
2. **Building the project.** llama.cpp is C++, so it must be compiled before it can run:
   ```bash
   cmake -B build                                  # configure (detected Metal + OpenSSL)
   cmake --build build -j -t llama-server          # build the server binary
   cmake --build build -j -t llama-cvector-generator  # build the cvector tool
   ```
3. **I had no model or control-vector file to test with.** I downloaded a small model and generated my own control vector from it (generating it from the same model guarantees the dimensions match):
   ```bash
   # small model into ./models/
   curl -L -o models/qwen2.5-0.5b-instruct-q4_k_m.gguf \
     "https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q4_k_m.gguf?download=true"
   # generate a control vector from that model
   ./build/bin/llama-cvector-generator -m models/qwen2.5-0.5b-instruct-q4_k_m.gguf -ngl 99 \
     --positive-file tools/cvector-generator/positive.txt \
     --negative-file tools/cvector-generator/negative.txt \
     -o control_vector_happy.gguf
   ```

### Steps to Reproduce

1. Build `llama-server` (see Environment Setup in repo).
2. Start the server with a control vector loaded:
   ```bash
   ./build/bin/llama-server -m models/qwen2.5-0.5b-instruct-q4_k_m.gguf \
     --control-vector control_vector_happy.gguf -ngl 99 --port 8080
   ```
3. In another terminal, query the existing LoRA endpoint and the requested control-vector endpoint:
   ```bash
   curl -w "\n%{http_code}\n" http://localhost:8080/lora-adapters   # returns []  with HTTP 200
   curl -w "\n%{http_code}\n" http://localhost:8080/cvectors        # returns HTTP 404
   ```
4. **Observed result:** `/lora-adapters` works (HTTP 200), but `/cvectors` returns **HTTP 404 Not Found** — the endpoint does not exist. The control vector *is* loaded (the server started with it), but there is no API to inspect or change it. The only way to change it is to stop and restart the server. Reproduced consistently (ran it twice, same result).

### Reproduction Evidence
- **Branch link:** https://github.com/aliceyh7/llama.cpp/tree/fix-issue-10685
- **Logs / output:**
  ```text
  GET /lora-adapters  ->  []                                                  HTTP 200
  GET /cvectors       ->  {"error":{"message":"File Not Found",...,"code":404}} HTTP 404
  POST /cvectors      ->  {"error":{"message":"File Not Found",...,"code":404}} HTTP 404
  ```
- **My findings:** There is no lora-adapter equivalent for control vectors anywhere in the server. Control vectors are only handled once, at model-load time in `common/common.cpp`. This confirms the feature is genuinely missing rather than broken.

---

## Solution Approach

### Analysis

Control vectors are only wired into the startup path. They are loaded and applied once in `common/common.cpp` (`common_control_vector_load` → `llama_set_adapter_cvec`), and `llama-server` exposes no route to change them afterward. LoRA adapters, by contrast, have a full runtime path (routes + parsing + state + apply), which is why they can be hot-swapped.

### Proposed Solution

Add `GET /cvectors` and `POST /cvectors` endpoints to `llama-server`, following the existing `/lora-adapters` pattern: a GET that lists the loaded control vectors (id, path, scale) and a POST that accepts a list of `{id, scale}` objects and re-applies the control vectors at the new scales.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Control vectors can currently only be set at startup; there is no API to view or change them at runtime. Add API endpoints so they can be managed live, matching how LoRA adapters already work.

**Match:** The `/lora-adapters` feature is the template to mirror:
- Routes registered in `tools/server/server.cpp` (the `get_lora_adapters` / `post_lora_adapters` handlers).
- Request parsing helpers in `tools/server/server-common.*` (e.g. `parse_lora_request`).
- Per-request handling and result serialization in `tools/server/server-task.cpp`.
- Underlying engine call: `llama_set_adapters_lora(...)` in `include/llama.h` (the control-vector equivalent is `llama_set_adapter_cvec`).

**Plan (step-by-step):**
1. Store the loaded control vectors as separate entries (id, path, scale, and the raw vector data) instead of discarding them after the initial combine.
2. Add a `GET /cvectors` route + handler that returns that list as JSON.
3. Add a `POST /cvectors` route + handler that parses a `[{id, scale}]` body and updates the stored scales.
4. On update, **re-combine** the stored vectors at the new scales and re-apply them with `llama_set_adapter_cvec` (passing `NULL` clears them).
5. Update `tools/server/README.md` to document the two new endpoints.
6. Add a test mirroring `tools/server/tests/unit/test_lora.py`.

**Implement:** *(Phase III — done.)* Implemented on branch https://github.com/aliceyh7/llama.cpp/tree/fix-issue-10685 across four commits (GET endpoint, POST endpoint, tests, docs). The control vectors loaded at startup are already kept in `params_base.control_vectors` (each entry holds its file path and strength), so no new storage struct was needed - the POST handler re-combines those entries at the new scales via `common_control_vector_load` and re-applies them with `llama_set_adapter_cvec`.

**Review:** Before opening a PR I will re-read `CONTRIBUTING.md` and `AGENTS.md`, match the project's code style, sync my branch with `upstream/master`, and confirm no one else has an active PR for this issue. I will write the PR description and commit messages myself (the project rejects AI-written PR text).

**Evaluate:** See Testing Strategy below — primarily an automated test that starts a server with a control vector, reads it via `GET /cvectors`, changes a scale via `POST /cvectors`, and confirms the change took effect, plus manual confirmation that generated text shifts when the scale changes.

---

## Testing Strategy

> Implemented in `tools/server/tests/unit/test_cvector.py`, modeled on `tools/server/tests/unit/test_lora.py`. I also added a `control_vector_files` option to the test harness (`tools/server/tests/utils.py`) so a server can be started with `--control-vector`.

### Unit Tests

- [x] `GET /cvectors` returns `[]` when no control vector is loaded at startup.
- [x] `POST /cvectors` with an out-of-range `id` returns HTTP 400.
- [x] `POST /cvectors` with a non-array body returns HTTP 400.
- [x] `GET /cvectors` lists the startup control vector with correct id/path/scale (skipped unless `CVECTOR_TEST_MODEL` / `CVECTOR_TEST_FILE` env vars point at a model + vector).
- [x] `POST /cvectors` with a new scale updates the stored scale, confirmed by a follow-up `GET` (same env-gated skip).

The two vector-backed tests are gated behind env vars because, unlike LoRA, there is no tiny public control-vector file to download in CI - it has to be generated from a specific model. The three error/empty-state tests run unconditionally.

### Manual Testing

Done for reproduction (confirmed `/cvectors` returned 404 before the change). For Phase III I started `llama-server` with `control_vector_happy.gguf`, confirmed `GET /cvectors` lists it at scale 1.0, changed the scale via `POST /cvectors`, and confirmed the follow-up `GET` reflected the new scale.

> Gotcha worth recording: a leftover `llama-server` from manual testing was still bound to port 8080, so the pytest harness silently ran against the wrong process (its `/health` check passed against the stale server). Killing the stray process fixed the failing tests - the tests themselves were correct.

---

## Implementation Notes

### Week 2 Progress

Reproduced the issue, documented findings, and made an UMPIRE plan to implement the fix. 

### Week 3 Progress

Implemented the feature on `fix-issue-10685`: added `GET /cvectors` and `POST /cvectors`, wrote unit tests, and documented both endpoints in the server README. Verified the build, ran the tests green, and prepared the branch for a PR.

### Code Changes

- **Files modified:**
  - `tools/server/server.cpp` - register the two routes
  - `tools/server/server-context.cpp` / `server-context.h` - GET/POST handlers and the `SET_CVEC` / `GET_CVEC` task handling
  - `tools/server/server-task.cpp` / `server-task.h` - task + result types and JSON serialization
  - `tools/server/server-common.cpp` / `server-common.h` - `parse_cvector_request` helper
  - `tools/server/README.md` - endpoint docs
  - `tools/server/tests/unit/test_cvector.py` - new tests
  - `tools/server/tests/utils.py` - `control_vector_files` harness option
- **Key commits** (`upstream/master..fix-issue-10685`):
  - `5829394` server: add GET /cvectors endpoint to list loaded control vectors
  - `436ee14` server: add POST /cvectors endpoint to hot-swap control vectors
  - `1eecdcf` server : add unit tests for GET/POST /cvectors
  - `3304916` server : document GET/POST /cvectors in README
- **Approach decisions:**
  - Mirrored the `/lora-adapters` design end-to-end (routes -> parse helper -> task -> result) so the change blends into existing server conventions.
  - Reused the existing `params_base.control_vectors` list rather than introducing a new storage struct; the POST handler re-combines and re-applies on each scale change.
  - Validate the requested `id` against the loaded-vector count and return HTTP 400 on out-of-range or non-array input.
  - KV-cache behavior: POST applies the new scale immediately without flushing slot caches, matching `POST /lora-adapters`. Chose consistency with the existing endpoint over adding a mandatory cache-flush; documented the trade-off in the server README so reviewers can weigh in.

---

## Pull Request

**PR Link:** https://github.com/ggml-org/llama.cpp/pull/24740 (opened 2026-06-18)

**PR Description:** Submitted as PR #24740, "server : add GET/POST /cvectors for control vector hot-swap" (base `master`, head `aliceyh7:fix-issue-10685`). Summary:

> Adds server support for managing loaded control vectors through HTTP endpoints:
> 1. `GET /cvectors` to list currently loaded control vectors
> 2. `POST /cvectors` to hot-swap control vectors at runtime
> 3. Unit tests covering the new GET and POST endpoints
> 4. README documentation describing the new API usage
>
> Related issue: #10685. All relevant server tests pass locally.

AI usage was disclosed in the PR per the project's requirements.

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** Open - awaiting maintainer review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
