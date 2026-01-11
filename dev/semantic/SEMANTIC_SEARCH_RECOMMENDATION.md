# Semantic Search Tool - Quick Reference

## Overview

Provide AI agents with **semantic search capability** to discover relevant notes based on meaning, not just keywords.

**Purpose:** Agent-facing MCP tool (not a user UI feature)

## What It Does

Enables agents to search the vault using natural language queries:
- "project planning" → finds "Roadmap 2024", "Sprint Goals" 
- "time management" → finds "GTD System", "Pomodoro Technique"
- "machine learning" → finds "Neural Networks", "Deep Learning Tutorial"

## Technology Stack

- **Embedding Model:** Xenova all-MiniLM-L6-v2 (23MB, local ONNX)
- **Vector Storage:** Vectra (in-memory, JSON-backed)
- **Interface:** MCP tool for agent invocation
- **Auto-sync:** Keeps index current with vault changes

## Key Benefits

✅ **100% Local** - No API dependencies  
✅ **Agent-facing** - Tool for AI, not UI feature  
✅ **Simple** - ~600 lines of code, 2 dependencies  
✅ **Fast** - <100ms queries  
✅ **Auto-updating** - Syncs with vault changes  
✅ **Pure semantic** - No hybrid complexity  

## Implementation Phases

**Phase 1:** Core semantic search service  
**Phase 2:** Auto-sync with vault changes  
**Phase 3:** MCP tool integration  
**Phase 4:** Settings & optimization  

**Total effort:** 4-6 AI implementation sessions

## Documentation

📖 **Complete technical guide:** [`RECOMMENDATION.md`](./RECOMMENDATION.md)

Includes:
- Architecture diagrams
- Full TypeScript implementations
- Performance benchmarks
- Testing strategy
- Alternative approaches analysis

## Quick Start

```bash
# Install dependencies
npm install @xenova/transformers vectra

# Implement Phase 1 first
# See RECOMMENDATION.md for complete code examples
```

## Dependencies

- `@xenova/transformers` (^2.17.0) - Local embeddings
- `vectra` (^0.9.0) - Vector storage

**Bundle impact:** +150KB  
**Model download:** ~23MB (one-time, cached)
