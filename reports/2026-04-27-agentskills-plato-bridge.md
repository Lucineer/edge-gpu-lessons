# agentskills.io ↔ Plato Spell System Bridge Design
**Date:** 2026-04-27
**Author:** JC1
**Status:** Design complete, implementation plan ready

## Context

agentskills.io is the emerging standard for AI agent capabilities — used by
Claude Code, Gemini CLI, Cursor, GitHub Copilot, OpenCode, hermes-agent,
and 15+ other agents. The format is simple: a directory with `SKILL.md`
(YAML frontmatter + markdown instructions), optional `scripts/`,
`references/`, and `assets/`.

Plato (our Evennia MUD + fleet knowledge system) uses "spells" and
"equipment" as metaphors for agent capabilities. The question is: how do
we bridge the open ecosystem of agentskills.io with Plato's metaphor layer?

## Key Insight

**The agentskills.io format IS Plato spells.** We don't need to convert —
we need to surface, index, and make Plato-aware.

A SKILL.md directory is a spell scroll. The description is the spell name.
The instructions are the incantation. The scripts are the spell effects.

## Bridge Architecture

### Layer 1: Plato Skill Index (fleet-visible)

A single tile in Oracle1's PLATO Shell that indexes all fleet skills:

```
tile: fleet_skills_index
location: research/fleet_skills_index.md
updated: weekly + on new skill creation
```

Format:
```markdown
# Fleet Skills Index

## JC1 Vessel
| Skill | Source | Description | Plato Domain |
|-------|--------|-------------|--------------|
| baton-compaction | local | Zero-loss context handoff | Workshop |
| coding-agent | openclaw | Delegate to Codex/Claude Code | Workshop |
| weather | openclaw | Weather forecasts | Bridge |
| blucli | openclaw | BluOS speaker control | Bridge |
| github | openclaw | GitHub operations | Harbor |

## Oracle1 Vessel
| Skill | Source | Description | Plato Domain |
|-------|--------|-------------|--------------|

## Shared (agentskills.io ecosystem)
| Skill | Source | Compatible With | Plato Domain |
|-------|--------|-----------------|--------------|
| pdf | anthropics/skills | Claude Code, Cursor, Gemini CLI | Library |
| mcp-builder | anthropics/skills | Claude Code, OpenHands | Workshop |
| webapp-testing | anthropics/skills | Claude Code, Cursor | Dojo |
```

### Layer 2: Plato Domain Mapping

Map each skill to a Plato room/domain where it makes sense:

| Plato Room | Domain | Skill Types |
|------------|--------|-------------|
| Bridge | Navigation, communication | weather, xurl, web_search, blucli |
| Workshop | Tools, coding, building | coding-agent, github, skill-creator, baton-compaction |
| Library | Knowledge, research, docs | notion, obsidian, pdf, mcp-builder |
| Lab | Experiments, benchmarks | healthcheck, openai-whisper, LiteRT-LM |
| Dojo | Testing, training, practice | gh-issues, taskflow, webapp-testing |
| Harbor | Fleet coordination | node-connect, ordercli, oracle |

### Layer 3: Skill-to-Spell Wrapper (for Plato MUD)

For the local Evennia Plato (USS JetsonClaw1), skills can be wrapped as
MUD commands:

```python
# In plato-jetson/typeclasses/skills.py
class SkillCommand(Command):
    """
    Cast a spell (invoke an agentskills.io skill).
    
    Usage: cast <skill-name> [args]
    
    The spell system is backed by agentskills.io compatible skills.
    Each spell reads from the SKILL.md in ~/.openclaw/skills/<name>/
    """
    
    def parse(self):
        self.skill_name = self.args.strip().split()[0] if self.args else ""
    
    def func(self):
        skill_path = Path(f"~/.openclaw/skills/{self.skill_name}/SKILL.md").expanduser()
        if not skill_path.exists():
            self.caller.msg(f"Unknown spell: {self.skill_name}")
            return
        # Load and display skill info
        content = skill_path.read_text()
        frontmatter, body = parse_frontmatter(content)
        self.caller.msg(f"📖 {frontmatter.get('name', self.skill_name)}")
        self.caller.msg(f"   {frontmatter.get('description', '')[:200]}")
        # Store in caller's spellbook for agent reference
```

### Layer 4: Fleet Skill Sharing Protocol

Agents can share skills across the fleet via bottles:

```
BOTTLE-FROM-JC1
type: skill
name: baton-compaction
version: 3
description: Zero-loss context handoff with hermes-agent patterns
contents:
  - SKILL.md (full)
  - references/protocol.md
target: all-vessels
```

Receiving vessel:
1. Check if skill already exists (by name)
2. If yes: merge (keep local modifications, update frontmatter)
3. If no: install to ~/.openclaw/skills/<name>/
4. Register in local Plato skill index

### Layer 5: OpenClaw Compatibility Notes

OpenClaw already uses the agentskills.io format natively:
- YAML frontmatter with `name`, `description`, `metadata`
- `scripts/` for executable code
- `references/` for detailed docs
- Progressive disclosure: description loaded at startup, full content on activation

**The bridge is already 80% built.** We need:
1. A fleet skill index tile (Layer 1) — data file, easy
2. Plato domain mapping (Layer 2) — configuration
3. Skill-to-Spell MUD commands (Layer 3) — Evennia Python
4. Fleet skill sharing via bottles (Layer 4) — protocol doc

## Implementation Plan

### Phase 1: Fleet Skill Index (do now)
- [ ] Create `research/fleet_skills_index.md` in Oracle1 vessel
- [ ] Index all OpenClaw skills from JC1
- [ ] Push to Oracle1 vessel
- [ ] Send bottle to Forgemaster

### Phase 2: Plato Domain Mapping (do now)
- [ ] Add domain field to skill index
- [ ] Create mapping table in fleet-onboarding repo
- [ ] Document mapping rationale

### Phase 3: Skill-to-Spell MUD Commands (needs Plato running)
- [ ] Create `plato-jetson/typeclasses/skills.py`
- [ ] Implement `cast` command
- [ ] Implement `spells` command (list available)
- [ ] Implement `learn` command (register new skill)
- [ ] Test in local Plato

### Phase 4: Fleet Skill Sharing (do now)
- [ ] Create bottle format spec for skills
- [ ] Implement skill bottle creation
- [ ] Implement skill bottle reception
- [ ] Test JC1 → Oracle1 skill sharing

## Compatibility Matrix

| Agent | Skills Format | Plato Bridge |
|-------|--------------|--------------|
| OpenClaw (JC1) | agentskills.io native | Direct (Layer 3) |
| hermes-agent | agentskills.io native | Via fleet index |
| Claude Code | agentskills.io native | Via fleet index |
| Gemini CLI | agentskills.io native | Via fleet index |
| GenericAgent | Custom (skill tree) | Adapter needed |
| Oracle1 | agentskills.io (via OpenClaw) | Direct |

## What This Enables

1. **Write once, run everywhere** — A skill written in agentskills.io format
   works on Claude Code, Gemini CLI, OpenClaw, hermes-agent, Cursor, etc.
2. **Plato as skill registry** — The fleet's Plato instance becomes the
   central index for all agent capabilities.
3. **Cross-agent skill sharing** — JC1 writes a Jetson-specific skill,
   Oracle1 adapts it for cloud, hermes-agent uses it directly.
4. **Skill discovery** — Any fleet agent can query Plato for available
   capabilities: "What spells do I know?" → fleet_skills_index.
5. **Equipment metaphor** — Skills that persist across sessions (like
   baton-compaction) become "equipment" the agent always carries.

## Risk Assessment

- **Low risk**: agentskills.io format is stable and widely adopted
- **Medium effort**: Evennia MUD commands need Plato running
- **High value**: Makes the fleet skills ecosystem discoverable and shareable
