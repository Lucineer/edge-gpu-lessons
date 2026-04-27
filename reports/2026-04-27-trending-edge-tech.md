# Edge & Agent Tech Research Report — April 2026
**Author:** JC1 (JetsonClaw1)  
**Date:** 2026-04-27  
**Scope:** Trending repos from April 2026, filtered for edge/fleet relevance

## Executive Summary

Five projects stand out for immediate integration into the Cocapn fleet:
1. **rtk** — Token compression proxy (INSTALLED on Jetson, 87% savings on find)
2. **hermes-agent** — Self-improving agent with learning loop, agentskills.io compatible
3. **GenericAgent** — 3K-line self-evolving agent, skill crystallization
4. **LiteRT-LM** — Google's edge LLM inference (benchmark in progress)
5. **claude-mem** — Session memory compression (reference for baton protocol)

---

## 1. rtk — CLI Token Compression Proxy

**Repo:** rtk-ai/rtk (37K★, +22K/month)  
**Language:** Rust (single binary, 25K lines)  
**Status:** ✅ COMPILED AND RUNNING on Jetson Orin Nano

### What It Does
Wraps CLI commands (ls, find, git, cargo, etc.) and compresses output before it reaches LLM context. 60-90% token reduction on verbose commands.

### Jetson Benchmarks
| Command | Raw (bytes) | rtk (bytes) | Savings |
|---------|------------|-------------|---------|
| ls -la memory/ | 1,340 | 454 | 66% |
| find workspace/ | 7,557 | 956 | 87% |
| git log --oneline -20 | 1,524 | 1,468 | 4% |

### Integration Notes
- Compiled from source (prebuilt ARM64 binary requires GLIBC 2.39, Jetson has 2.35)
- Installed at `~/.local/bin/rtk`, global hook registered
- Best savings on verbose outputs (find, tree, large file reads)
- Minimal savings on already-compact git output
- **Recommendation:** Wire into OpenClaw exec wrapper for automatic compression

---

## 2. hermes-agent — Self-Improving Agent

**Repo:** NousResearch/hermes-agent (120K★, +105K/month)  
**Language:** Python  
**Status:** 📖 Architecture studied

### Key Innovations

**Context Compressor** (our baton protocol equivalent):
- Uses auxiliary (cheap) model to summarize middle turns
- Protects head (system prompt) and tail (recent messages) 
- Structured summary with Resolved/Pending question tracking
- "Handoff framing" — tells model "this is from a different session, don't re-execute"
- Token-budget tail protection (not fixed message count)
- Tool output pruning before LLM summarization
- Iterative summary updates across multiple compactions
- **KEY INSIGHT:** "Do NOT answer questions mentioned in summary — they were already addressed"
- Summary budget: 20% of compressed content, ceiling 12K tokens

**Memory Manager:**
- BuiltinMemoryProvider always registered
- External plugin memory (max 1 at a time)
- System prompt integration
- Pre-turn prefetch, post-turn sync
- Context fencing: strips injected context blocks from provider output

**Skill System:**
- agentskills.io compatible (open standard!)
- Autonomous skill creation after complex tasks
- Skills self-improve during use
- FTS5 session search with LLM summarization for cross-session recall
- Cron scheduler for scheduled automations

**Multi-model Support:**
- z.ai/GLM, Kimi/Moonshot, OpenRouter, NVIDIA NIM, HuggingFace
- Switch with `hermes model` — no code changes

### Plato Integration Opportunities
1. **agentskills.io standard** — Our Plato spells/equipment could implement this
2. **Context compressor patterns** — Adopt their summary template for baton protocol
3. **Learning loop** — Their "nudge" system for persisting knowledge maps to Plato knowledge tiles
4. **Multi-model** — They already support our stack (z.ai, Moonshot)

### What We Should Adopt
- Their `SUMMARY_PREFIX` framing for compaction (reference-only, not instructions)
- Token-budget tail protection instead of our fixed message count
- Tool output pruning before summarization
- agentskills.io compatibility for Plato spell system

---

## 3. GenericAgent — Self-Evolving Minimal Agent

**Repo:** lsdefine/GenericAgent (7.7K★, +6.5K/month)  
**Language:** Python (~3K lines core, 100-line agent loop)  
**Status:** 📖 Architecture studied

### Core Design
- **9 atomic tools** + ~100-line agent loop
- System-level control: browser (real, with sessions), terminal, filesystem, keyboard/mouse, screen vision, ADB (mobile)
- **Self-evolution:** Crystallizes each task into a reusable skill
- Skill tree grows with use — personal, unique to each instance
- **Token efficient:** <30K context window (vs 200K-1M for other agents)
- Reset tool descriptions every 10 turns to avoid context bloat
- "No preload skills — evolve them" philosophy

### Architecture (agent_loop.py)
```
AgentRunnerLoop:
  - messages = [system, user]
  - while turn < max_turns:
      - LLM call with tools_schema
      - dispatch tool calls via handler.do_{tool_name}
      - StepOutcome: {data, next_prompt, should_exit}
      - handler.turn_end_callback()
      - messages = [new user message] (history kept in Session, not context)
```

### Key Insight: History in Session, Not Context
GenericAgent keeps conversation history in a Session object, NOT in the LLM context. Only the latest message goes to the model. This is the core of their token efficiency.

### Plato Integration
- Their skill crystallization maps directly to Plato spells
- "The longer you use it, the more skills accumulate" = Plato equipment
- Their tool reset pattern (every 10 turns) could help our context management
- Session-based history (not context-based) is what Casey described: "context surrounding you in Plato"

---

## 4. LiteRT-LM — Google Edge LLM Inference

**Repo:** google-ai-edge/LiteRT-LM (4.4K★, +3.4K/month)  
**Status:** 🔄 Benchmarking in progress (DeepSeek-R1-Distill-Qwen-1.5B)

### What It Does
Production-ready LLM inference framework for edge devices. Powers Chrome, Chromebook Plus, Pixel Watch.

### Key Features
- Cross-platform: Android, iOS, Web, Desktop, IoT (Raspberry Pi)
- GPU and NPU acceleration
- Multi-modality (vision, audio)
- Tool use / function calling for agentic workflows
- Supports Gemma, Llama, Phi-4, Qwen

### Jetson Testing Plan
- Model: DeepSeek-R1-Distill-Qwen-1.5B (q8, ~1.5GB)
- Backend: GPU (Jetson Orin Nano CUDA cores)
- Metrics: prefill speed (tok/s), decode speed (tok/s), memory usage
- Comparison: vs raw CUDA, vs TensorRT

### Hurdle
- Most LiteRT models are gated on HuggingFace (need auth token)
- Ungated models available but limited (DeepSeek-R1-Distill-Qwen-1.5B found)
- .tflite format NOT supported for LLM inference — need .litertlm files
- The only ungated .litertlm file (DeepSeek-R1-Distill-Qwen-1.5B) is ~1.5GB
- Without HF_TOKEN, download rate-limited to crawl (got 587MB in 30min, killed)
- **BLOCKED**: Need Casey to provide HuggingFace token for gated model access
- Tool itself installs and runs fine on Jetson via `uv tool install litert-lm`
- Python 3.13 (via uv) works — no compatibility issues

---

## 5. claude-mem — Session Memory Compression

**Repo:** thedotmack/claude-mem (68K★, +27K/month)  
**Language:** TypeScript (Claude Code plugin)

### What It Does
Automatically captures everything Claude does during coding sessions, compresses with AI, injects relevant context into future sessions.

### Relevance
- Similar problem to our baton compaction protocol
- Session-to-session continuity
- AI-powered compression (not just truncation)
- **Reference:** Compare their compression approach with our baton protocol

---

## Integration Roadmap

### Phase 1 (Immediate — Today)
- [x] Install rtk on Jetson ✅
- [ ] Complete LiteRT-LM benchmark
- [ ] Update TOOLS.md with rtk installation notes

### Phase 2 (This Week)
- [ ] Adopt hermes-agent context compressor patterns for baton protocol
- [ ] Implement agentskills.io compatibility in Plato spell system
- [ ] Write GenericAgent skill crystallization analysis for Plato integration
- [ ] Push edge research report to edge-gpu-lessons repo

### Phase 3 (Near Term)
- [ ] Benchmark LiteRT-LM vs TensorRT vs raw CUDA on Jetson
- [ ] Evaluate hermes-agent as a fleet agent framework
- [ ] Design Plato spell ↔ agentskills.io bridge
- [ ] Consider GenericAgent's "history in session, not context" pattern for fleet

---

## Technical Notes

### Jetson Constraints
- GLIBC 2.35 (rtk prebuilt needs 2.39 — must compile from source)
- 8GB unified RAM limits model size
- ARM64 — not all prebuilt binaries available
- Python 3.10 (some tools need 3.12+)
- OOM risk with large git clone operations (use --depth 1)
- Disk: 2TB NVME — plenty of space for models

### API Key Status
- xAI search: key invalid (web_search broken)
- DeepSeek: credits expired 2026-04-24
- Anthropic: credits at zero
- Working: Groq, DeepInfra, SiliconFlow, Moonshot, z.ai
