# Semantic Search Tool - Implementation Summary

## What This Is

A recommendation for implementing a **semantic search MCP tool** that AI agents can use to discover relevant notes in the Obsidian vault based on conceptual meaning rather than exact keyword matches.

**Important:** This is a tool for AI agents (Claude Code, Gemini CLI, etc.), not a user-facing feature.

## The Problem

Currently, agents can only find notes through:
- Exact file paths (must know the exact path)
- Keyword search (only matches literal text)
- User mentions (user must manually provide context)

**Missing:** Ability to discover conceptually related notes. Example:
- Agent discusses "project planning" but can't find "Roadmap 2024" or "Sprint Goals"
- Agent helps with "time management" but misses "GTD System" or "Pomodoro Technique"

## The Solution

Provide agents with a `semantic_search` MCP tool that:
1. Accepts natural language queries
2. Returns semantically similar notes ranked by relevance
3. Runs 100% locally (no APIs)
4. Auto-updates as vault changes

### Example Usage

```
Agent invokes: semantic_search("project planning strategies")

Returns:
- "Work/2024 Roadmap.md" (score: 0.91)
- "Planning/Product Strategy.md" (score: 0.87)
- "Projects/Sprint Goals.md" (score: 0.82)

Agent: "I found your 2024 Roadmap and Product Strategy notes. 
       Would you like me to review them for your planning?"
```

## Architecture

```
AI Agent → MCP Tool → SemanticSearchService → Vectra Index
                                                   ↓
                                            Embeddings stored
                                            in .obsidian/
```

**Core Technologies:**
- **Xenova all-MiniLM-L6-v2:** Local embedding model (23MB)
- **Vectra:** Lightweight vector database (~100KB)
- **Auto-sync:** Vault event listeners keep index current

## Implementation Phases

### Phase 1: Core Service (4-6 hours AI effort)
- Create `SemanticSearchService` class
- Initialize embedding model
- Implement vector indexing
- Basic search functionality

### Phase 2: Auto-Sync (2-3 hours AI effort)  
- Hook into vault events (create/modify/delete/rename)
- Debounced re-indexing for edits
- Startup sync check for stale entries

### Phase 3: MCP Tool Integration (4-6 hours AI effort)
- Define tool schema
- Implement tool handler
- Register with ACP adapter
- Handle agent invocations

### Phase 4: Settings & Polish (2-3 hours AI effort)
- Settings UI (enable/disable, rebuild)
- Error handling & graceful degradation
- Performance optimizations

**Total:** 4-6 AI implementation sessions

## Files to Create

```
src/domain/ports/semantic-search.port.ts
src/adapters/obsidian/semantic-search.service.ts
src/adapters/obsidian/semantic-index-sync.manager.ts
```

## Performance Expectations

| Metric | Value |
|--------|-------|
| Initial model load | 2-3s (once per session) |
| Index 1000 notes | 2-4 minutes (one-time) |
| Query latency | 10-100ms |
| Storage | ~50KB per 1000 notes |
| Memory | ~150MB additional |

## Key Changes from Original Proposal

✅ **Focus:** Agent tool, not user UI feature  
✅ **Interface:** MCP tool, not @mention enhancement  
✅ **Timeline:** Phases (AI sessions), not weeks  
✅ **Approach:** Pure semantic, not hybrid search  
✅ **Auto-sync:** Index stays current with vault changes  

## Documentation Structure

- **This file:** High-level summary
- **[SEMANTIC_SEARCH_RECOMMENDATION.md](SEMANTIC_SEARCH_RECOMMENDATION.md):** Quick reference
- **[RECOMMENDATION.md](RECOMMENDATION.md):** Complete technical guide with full code examples

## Next Steps

1. Review **[RECOMMENDATION.md](RECOMMENDATION.md)** for complete implementation details
2. Install dependencies: `npm install @xenova/transformers vectra`
3. Begin Phase 1: Implement `SemanticSearchService`
4. Test with small vault before proceeding to Phase 2
5. Iterate based on performance metrics

## Dependencies

```json
{
  "dependencies": {
    "@xenova/transformers": "^2.17.0",
    "vectra": "^0.9.0"
  }
}
```

**Bundle impact:** +150KB  
**Model download:** ~23MB (one-time, cached)

## Questions?

Refer to **[RECOMMENDATION.md](RECOMMENDATION.md)** for:
- Complete architecture diagrams
- Full TypeScript code examples
- Performance benchmarks
- Testing strategy
- Security analysis
- Alternative approaches considered
