# Baton Protocol Improvements — From hermes-agent Study
**Date:** 2026-04-27
**Source:** NousResearch/hermes-agent context_compressor.py (1350 lines)
**Status:** IMPLEMENTED — see ~/.openclaw/skills/baton-compaction/SKILL.md v3

## Key Patterns Adopted

### 1. "Reference Only" Framing (CRITICAL) — ✅ IMPLEMENTED
ONBOARDING.md now includes mandatory handoff framing that prevents next-gen
from re-executing completed work. Header text copied from hermes-agent's
SUMMARY_PREFIX pattern.

### 2. Anti-Thrashing Protection — ✅ IMPLEMENTED
lineage.json now tracks `anti_thrash` object with savings_pct and
ineffective_handoff_count. Two consecutive <10% savings triggers a
recommendation to `/new` instead of continuing.

### 3. Tool Output Pruning — ✅ IMPLEMENTED
STATE.md "Completed Actions" section now requires structured one-liner
format: `[tool] action -> outcome`. No raw output allowed.

### 4. Token-Budget Tail Protection — ✅ IMPLEMENTED
ONBOARDING.md capped at 200 words with explicit trimming priority.
STATE.md unlimited. "Active Task" field identified as the single most
important piece of context.

### 5. Iterative Summary Updates — ✅ IMPLEMENTED
Protocol now requires reading previous generation's STATE.md before
writing new one. Completed actions continue numbering, resolved questions
migrate, active task updates.

### 6. Structured Summary Template — ✅ IMPLEMENTED
STATE.md now has 13 sections including "Resolved Questions" (prevents
re-answering), "Pending User Asks" (prevents dropping requests), and
"Remaining Work" (not "Next Steps" — avoids re-execution).

### 7. "Remaining Work" Not "Next Steps" — ✅ IMPLEMENTED
Framed as context, not instructions.

## What Changed in the Skill File
- SKILL.md grew from ~3.5KB to ~8KB
- Added 7 numbered patterns with implementation details
- Verification checklist expanded from 5 to 6 items
- Secret verification pattern added (grep for sk-, key-, token, etc.)
- baton.yaml now includes environment fields (hostname, os, gpu, cuda_version)
