# llama.cpp — 7900xtx-vulkan-qwen3.6 branch

## Purpose

A minimal build of llama.cpp targeting a single hardware + model combination:

- **GPU:** AMD Radeon RX 7900 XTX (24GB VRAM, RDNA3 / gfx1100)
- **Backend:** Vulkan (primary) + CPU (fallback)
- **Model:** Qwen3.6-27B GGUF series (Qwen3.5/3.6, `LLM_ARCH_QWEN35`)
- **Target:** `llama-server` only — no CLI tools, no benchmarks, no tests

## Goal

Reduce compile time, binary size, and runtime overhead by stripping everything
that is not needed for this narrow use case.

---

## Completed changes

### 1. CMake build system

**Root `CMakeLists.txt`:**
- Removed WASM/Emscripten support, iOS support, deprecated option aliases
- Disabled: `BUILD_TESTS`, `BUILD_EXAMPLES`, `BUILD_APP`
- Enabled: `BUILD_TOOLS` (server only), `BUILD_SERVER`, `BUILD_UI`
- Removed: `LLAMA_TOOLS_INSTALL`, `LLAMA_TESTS_INSTALL`, `LLAMA_LLGUIDANCE`

**`ggml/src/CMakeLists.txt`** (-348 lines):
- Kept only **CPU** and **Vulkan** backend compilation
- Removed: CUDA, Metal, HIP, SYCL, BLAS, RPC, CANN, kanata, vulkan-shader lib embed
- Removed: LTO/IPO, ccache/sccache integration, sanitizer flags, Apple linker check

**`src/CMakeLists.txt`:**
- Replaced `file(GLOB models/*.cpp)` with explicit list:
  - `models/delta-net-base.cpp`
  - `models/qwen35.cpp`

**`tools/CMakeLists.txt`:**
- Only builds: `mtmd` (server dependency), `ui`, `server`
- Removed: batched-bench, gguf-split, imatrix, llama-bench, completion,
  perplexity, quantize, tokenize, parser, tts, rpc, cvector-generator,
  export-lora, fit-param

**`tools/server/CMakeLists.txt`:**
- Removed `install(TARGETS ...)` block

### 2. Model architecture

**`src/llama-model.cpp`** (-258 lines):
- Reduced `llama_model_mapping()` switch from ~100 arch cases to only `QWEN35`
- All other architectures fall through to an error

### 3. Server defaults (`common/common.h`)

| Parameter | Before | After | Reason |
|-----------|--------|-------|--------|
| `n_ctx` | 0 (auto) | 266144 | 256K context for Qwen3.6-27B |
| `cache_type_k` | F16 | Q4_0 | Fit 256K KV cache in 24GB VRAM |
| `cache_type_v` | F16 | Q4_0 | Same |
| `flash_attn_type` | AUTO | ENABLED | Qwen3.6 uses flash attention |

### 4. Server routes (`tools/server/server.cpp`)

- Disabled router mode (`is_router_server = false`)
- Removed router models manager and all proxy handlers
- Kept only 4 routes:
  - `GET  /health`, `/v1/health` — health check
  - `GET  /models`, `/v1/models` — model info
  - `POST /v1/chat/completions` — OpenAI-compatible chat
  - `GET/POST /props` — server properties

---

## Remaining work

### Phase 2: Architecture enum cleanup — DONE

**Completed:**
- `src/llama-arch.cpp`: Name mapping reduced to QWEN35 + UNKNOWN only (-133 lines)
- `src/llama-arch.cpp`: Helper functions simplified (`llm_arch_is_hybrid`, `llm_arch_is_recurrent`,
  `llm_arch_is_diffusion`, `llm_arch_supports_rs_rollback`, `llm_arch_supports_sm_tensor`)
- `src/llama-graph.cpp`: Removed STEP35 swiglu clamp, MODERN_BERT pooling, QWEN3/QWEN3VL reranker paths
- `src/llama-kv-cache.cpp`: Removed DEEPSEEK32 indexer, STEP35 shift check
- `src/llama-model-saver.cpp`: Simplified `supports_arch` to only QWEN35

**Note:** `llama-arch.h` enum values kept (compile-time constants, zero runtime cost).
`llama-model.cpp` switch cases for other arches kept (unreachable — model mapping throws first).

### Phase 3: Vulkan shader simplification

- Hardcode shader target to `RDNA3` (`gfx1100`)
- Enable `coopmat` extension by default (7900 XTX supports it)
- Remove runtime shader target detection

### Phase 4: Quantization simplification

- Keep only Q4_0, Q4_1, Q5_0, Q5_1, Q8_0 (formats used by Qwen3.6 GGUF)
- Remove MXFP4, IQ2, IQ3, IQ4, K-quants if not needed

### Phase 5: Common library cleanup

- Remove unused sampling strategies
- Remove speculative decoding, control vectors, multimodal from common
- Simplify `common_params` struct

### Phase 6: Build verification

- Configure with `-DGGML_VULKAN=ON` and verify full build
- Test `llama-server` loads Qwen3.6-27B GGUF correctly
- Verify Vulkan backend is active (not CPU fallback)

---

## Build instructions

### CPU-only build (development/testing)

```bash
mkdir build-cpu && cd build-cpu
cmake .. -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=OFF \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
cmake --build . --config Release -j$(nproc)
```

### Vulkan build (target deployment)

Requires: `vulkan-sdk` or `vulkan-tools` + `libvulkan-dev`

```bash
mkdir build-vulkan && cd build-vulkan
cmake .. -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=ON \
  -DCMAKE_C_COMPILER_LAUNCHER=ccache \
  -DCMAKE_CXX_COMPILER_LAUNCHER=ccache
cmake --build . --config Release -j$(nproc)
```

### Run

```bash
./bin/llama-server --model /path/to/Qwen3.6-27B-Q4_K_M.gguf -ngl 99
```

---

## Summary

- **1704 files deleted**, 13 files modified (231 insertions, 1781 deletions)
- Removed: all non-QWEN35 models, all non-server tools, tests, examples, CI, docs, conversion scripts
- Kept: src/ (QWEN35 only), ggml/ (CPU + Vulkan), common/, tools/server/ + mtmd/ + ui/, vendor/ (all needed)
- All remaining source files are verified compiled in the build
- CPU build: **verified passing**
- Vulkan build: **verified passing** (glslc + VK 1.3.275, coopmat supported, ccache enabled)
- Phase 2 (arch cleanup): **completed**

---

## References

- [DrBearJew/RoxxY](https://github.com/DrBearJew/RoxxY) — Related project for RX 7900 XTX llama.cpp optimization
