# Edge Toolkit Architecture Review — JC1 Self-Audit
## Date: 2026-04-29

---

## 1. ARCHITECTURE ASSESSMENT

### Current Split (gateway/chat/rag/monitor) — MOSTLY RIGHT

**Correct decisions:**
- `edge-gateway.py` as the unified API surface (single port, OpenAI-compatible) ✅
- `edge-rag.py` as standalone CLI + server (can run independently) ✅
- `jetson-monitor.py` as CLI-only (no HTTP needed) ✅
- `gpu-bench.py` as one-shot benchmark runner ✅

**Problems:**
1. **Massive code duplication** — `ollama_request()` and `cosine_sim()` are copy-pasted across gateway, rag, chat. Need a shared `ollama_client.py`.
2. **edge-chat.py duplicates edge-gateway.py** — chat.py has its own HTTP server, Ollama client, and HTML. It should be a thin frontend that calls the gateway API.
3. **Missing: shared config** — OLLAMA_URL, DEFAULT_MODEL, paths are hardcoded in every file.
4. **Missing: graceful shutdown** — HTTPServer doesn't handle SIGTERM properly. `server.shutdown()` in except block may deadlock.
5. **Missing: request timeout enforcement** — Ollama requests have 600s timeout but the HTTP server itself has no timeout. A hung Ollama request blocks the single-threaded server forever.
6. **Missing: threading** — HTTPServer is single-threaded. One slow inference blocks all other requests. Need ThreadingHTTPServer.

### Recommended Architecture
```
edge/
  __init__.py
  config.py          # Shared config (OLLAMA_URL, defaults, paths)
  ollama_client.py   # Shared Ollama API client
  embeddings.py      # Embedding utilities
  similarity.py      # Cosine similarity, vector ops
  monitoring.py      # Thermal, CMA, RAM reading
edge-gateway.py      # HTTP server (imports from edge/)
edge-chat.py         # Static HTML served by gateway or nginx
edge-rag.py          # CLI tool (imports from edge/)
jetson-monitor.py    # CLI tool (imports from edge/)
gpu-bench.py         # CLI tool (imports from edge/)
```

---

## 2. BUGS & ISSUES (with fixes)

### BUG: jetson-monitor.py — UnboundLocalError in read_thermal_zones (line 98)
```python
# BROKEN: temp_raw may be undefined if f.read() returns None
with open(os.path.join(tz_path, 'temp'), 'rb') as f:
    raw = f.read()
if raw:
    temp_raw = raw.decode().strip()
if temp_raw and temp_raw != '0':  # NameError if raw was None
```
**FIX:** Initialize `temp_raw = ""` before the try block.

### BUG: edge-rag.py — Re-indexing appends duplicates
If you run `edge-rag.py index` twice on the same path, chunks get appended to the index without deduplication. The index grows forever.
**FIX:** Check file hash before adding. If hash matches existing, skip.

### BUG: edge-gateway.py — Streaming SSE format issues
The streaming chat sends `data: [DONE]` as the final event, but OpenAI's actual SSE format sends `data: [DONE]` as a bare line (no JSON wrapping). Our implementation is correct here but the chunk IDs all use `int(time.time()*1000)` which can duplicate for rapid chunks.
**FIX:** Use a counter for chunk IDs.

### BUG: edge-gateway.py — No request body size limit
A malicious client can send an arbitrarily large JSON body. On 8GB RAM this is dangerous.
**FIX:** Cap Content-Length at 1MB.

### BUG: gpu-bench.py — get_thermal() uses open() without 'rb' fallback
```python
with open(path) as f:  # text mode — fails on some sysfs nodes
```
**FIX:** Use 'rb' mode consistently (like jetson-monitor.py does).

### BUG: edge-gateway.py — rag_search loads entire index into memory on every request
For large indices this will be slow and memory-intensive.
**FIX:** Cache the index in memory with a file modification time check.

### SECURITY: edge-gateway.py — API key comparison is timing-vulnerable
```python
if auth != f"Bearer {API_KEY}":
```
**FIX:** Use `hmac.compare_digest()` for constant-time comparison.

### SECURITY: All tools — Binding to 0.0.0.0
Exposes the server on all interfaces. Fine for development, dangerous in production.
**FIX:** Default to 127.0.0.1, require --host flag for 0.0.0.0.

---

## 3. PERFORMANCE ANALYSIS

### CUDA 3.17 GFLOPS (0.08% of theoretical 40 TOPS)

**Root cause:** The CUDA matmul kernel uses naive O(N³) without:
1. Shared memory tiling (critical for memory-bound operations)
2. Proper thread block sizing for Orin's 1024-core Ampere SM
3. CUDA graphs to reduce kernel launch overhead

**Theoretical analysis:**
- Orin Nano: 1024 CUDA cores, ~1.0 GHz GPU clock (idle clock, can go higher with jetson_clocks)
- FP32 peak: 1024 cores × 2 FMA/cycle × 1.0 GHz = ~2 TFLOPS FP32
- 512×512 matmul = 2 × 512³ = 268M FLOPs
- At 3.17 GFLOPS: 268M / 3.17G = 84.5ms per matmul → matches our benchmark (1.7ms × 50 iters = 85ms)
- Expected at peak: 268M / 2T = 0.134ms → **should be 630× faster**

**Fixes:**
1. **Use TensorRT** for everything — trtexec already gets 17,600 QPS. The ONNX→TRT compiler handles kernel fusion, tiling, and layout optimization.
2. **Use cuBLAS directly** — `cublasSgemm` is heavily optimized for Ampere.
3. **CUDA shared memory tiling** — For custom kernels, use 16×16 or 32×32 tiles with shared memory.
4. **The 0.5 GB/s bandwidth** is also terrible — Orin has ~68 GB/s memory bandwidth. Likely the GPU is not clocked up (idle clock = ~300 MHz vs max ~1.3 GHz). `jetson_clocks` (needs sudo) would fix this.

### TensorRT Python API crashes

**Root cause:** Myelin (TensorRT's Jetson-specific kernel optimizer) requires a proper CUDA stream, not the default stream. Python's tensorrt bindings don't expose stream creation easily.

**Fix:** Use `trtexec` CLI for all TensorRT work. It handles streams correctly. Wrap with subprocess.

### CMA Workarounds (without sudo)

1. **Kill idle GPU processes** — Other services may be holding CMA. Check with `fuser /dev/nvhost-gpu`.
2. **Ollama --num-gpu-layers** — Reduce GPU layers to fit in available CMA.
3. **Smaller quantization** — Use Q2_K or Q3_K GGUF quants instead of Q4_K_M.
4. **llama.cpp directly** — More memory control than Ollama's wrapper.
5. **Set CUDA_VISIBLE_DEVICES=""** for CPU-only inference of small models.
6. **Pre-clear CMA** — Unload all models before loading a new one: `ollama stop <all>`.

---

## 4. PRODUCT VIABILITY FOR COCAPN.AI

### What's missing for production:

1. **Authentication** — API key is a start but needs:
   - Per-user API keys stored in a config file or SQLite
   - Rate limiting per key
   - Usage quotas

2. **Model management** — Users need to:
   - Pull/update models without CLI
   - See which models fit their hardware
   - Auto-detect GPU memory and recommend models

3. **Persistence** — Conversations are in-memory. Need:
   - SQLite or JSON file storage
   - Conversation history API (list/search/delete)

4. **Multi-device** — Gateway should support:
   - Multiple Jetson devices behind a load balancer
   - Device health reporting
   - Automatic model routing based on device capability

5. **Monitoring dashboard** — The stats endpoint exists but needs:
   - Web UI with charts (GPU temp over time, request latency, token throughput)
   - Alerting on thermal throttling or OOM
   - Historical data (currently no persistence)

6. **Error handling** — Current errors are bare dicts. Need:
   - OpenAI-compatible error format
   - Retry logic for transient Ollama failures
   - Circuit breaker for Ollama health checks

7. **Install story** — Need:
   - pip installable package (setup.py/pyproject.toml)
   - systemd service files
   - Auto-detection of Jetson capabilities
   - First-run setup wizard

### What makes it useful to customers:

1. **Privacy** — Data never leaves the device
2. **No API costs** — Perpetual inference after hardware purchase
3. **Low latency** — No network round-trip
4. **Offline capable** — Works without internet
5. **OpenAI compatible** — Drop-in replacement for existing integrations
6. **RAG included** — Document search + generation in one API call

### Pricing suggestion:
- Sell the Jetson Orin Nano pre-configured (hardware + software bundle)
- Or sell the edge-gateway software as a product ($49-99/device)
- Or include it in cocapn.ai Pro tier as a hybrid option

---

## 5. PRIORITY FIXES TO IMPLEMENT NOW

1. Create `edge/config.py` — shared config
2. Create `edge/ollama_client.py` — shared Ollama client
3. Fix jetson-monitor.py thermal zone bug
4. Add ThreadingHTTPServer to gateway
5. Add request body size limit
6. Fix API key timing attack
7. Add index caching to RAG search
8. Fix edge-rag.py duplicate indexing
