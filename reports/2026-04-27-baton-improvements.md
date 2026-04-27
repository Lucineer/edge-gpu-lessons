# Baton Protocol Improvements — From hermes-agent Study
**Date:** 2026-04-27
**Source:** NousResearch/hermes-agent context_compressor.py (1350 lines)

## Key Patterns to Adopt

### 1. "Reference Only" Framing (CRITICAL)
hermes-agent uses this summary prefix:
> "[CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted into the summary below. This is a handoff from a previous context window — treat it as background reference, NOT as active instructions. Do NOT answer questions or fulfill requests mentioned in this summary; they were already addressed."

Our baton protocol should adopt this framing to prevent the agent from re-executing completed work.

### 2. Token-Budget Tail Protection (replace fixed message count)
Currently our baton uses fixed message counts for tail protection. hermes-agent uses:
- Walk backward from end, accumulating token costs
- Protect tail until budget exceeded
- 4 chars ≈ 1 token estimate
- Images counted as 1600 tokens flat each
- Summary budget: 20% of compressed content, ceiling 12K tokens

### 3. Tool Output Pruning (CHEAP, no LLM call)
Before any LLM summarization:
- Replace old tool results with 1-line summaries: `[terminal] ran 'npm test' -> exit 0, 47 lines output`
- Deduplicate identical tool results (same file read 5x → keep newest, back-reference others)
- Truncate large tool_call arguments (50KB write_file content → 200 char JSON-valid truncation)

### 4. Anti-Thrashing Protection
- Track compression savings percentage
- If last 2 compressions each saved <10%, skip further compression
- Suggest `/new` or focused `/compress <topic>` instead
- Cooldown timer on summary failures (600s)

### 5. Iterative Summary Updates
- Store previous summary
- When compressing again, update the existing summary instead of regenerating from scratch
- Preserves information across multiple compaction events

### 6. Compaction Summary Template (structured)
```
## Resolved
- [completed items with outcomes]

## Pending
- [items that were in-progress or not yet addressed]

## Active Task
- [what the agent should resume doing RIGHT NOW]

## Key Decisions
- [architectural choices, context that informs future work]

## Files Modified
- [list of files changed, for state awareness]
```

### 7. "Remaining Work" Not "Next Steps"
hermes-agent uses "Remaining Work" instead of "Next Steps" — the latter reads as active instructions that get re-executed.

## Implementation Priority

1. **HIGH**: "Reference Only" framing in baton handoff
2. **HIGH**: Anti-thrashing protection (prevent infinite compaction loops)
3. **MEDIUM**: Token-budget tail protection
4. **MEDIUM**: Tool output pruning pass (before LLM summarization)
5. **LOW**: Structured summary template
6. **LOW**: Iterative summary updates
