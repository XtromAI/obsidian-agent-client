# Semantic Search Implementation Recommendation

This document provides a comprehensive recommendation for implementing local semantic search functionality in the Agent Client plugin. The goal is to enhance note mentions with semantic understanding while maintaining simplicity, privacy, and maintainability.

## Problem Statement

Currently, the plugin uses fuzzy string matching (via Obsidian's `prepareFuzzySearch`) to search notes when users type `@` to mention a note. This approach:

- Only matches based on literal character sequences in filenames, paths, and aliases
- Doesn't understand semantic similarity or meaning
- May miss relevant notes that use different terminology or phrasing

**Example:** A user searching for `@productivity` might not find notes titled "Getting Things Done" or "Time Management Tips" even though they're semantically related.

## Goals

1. **Local-only processing**: No external API calls or cloud dependencies
2. **Privacy-first**: All embeddings and search happen on the user's device
3. **Simple & maintainable**: Minimal dependencies, clear code structure
4. **Performance**: Fast enough for real-time search suggestions (<100ms)
5. **Compatibility**: Works on Windows, macOS, and Linux

## Recommended Solution

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    User Types "@query"                  │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│           NoteMentionService (Enhanced)                 │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 1. Fuzzy search (existing - fast initial filter) │  │
│  └───────────────────┬───────────────────────────────┘  │
│                      │                                   │
│  ┌───────────────────▼───────────────────────────────┐  │
│  │ 2. Semantic search (new - rerank by meaning)     │  │
│  │    - Generate query embedding                      │  │
│  │    - Compare with cached note embeddings          │  │
│  │    - Merge & sort results by relevance            │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│              Vectra (Vector Database)                   │
│  - Stores note embeddings locally                       │
│  - Fast cosine similarity search                        │
│  - Syncs on vault changes                               │
└─────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Embedding Model: Xenova all-MiniLM-L6-v2

**Why this model?**
- **Small & Fast**: Only 23MB, generates 384-dimensional embeddings
- **Well-tested**: Popular in production, excellent balance of accuracy vs. speed
- **Local execution**: Runs in Node.js via ONNX (no GPU required)
- **Zero dependencies on external APIs**: Fully offline after initial model download
- **Battle-tested**: Used in many semantic search applications

**Installation:**
```bash
npm install @xenova/transformers
```

**Usage Example:**
```typescript
import { pipeline } from '@xenova/transformers';

// Initialize once at startup
const embedder = await pipeline(
  'feature-extraction', 
  'Xenova/all-MiniLM-L6-v2'
);

// Generate embeddings
const embedding = await embedder('Your text here', {
  pooling: 'mean',
  normalize: true
});
```

#### 2. Vector Storage: Vectra

**Why Vectra?**
- **Local-first**: Stores all data as JSON files in the vault
- **In-memory performance**: Sub-2ms queries for thousands of notes
- **Simple API**: MongoDB-like query syntax, easy to integrate
- **Lightweight**: Minimal dependencies, ~100KB
- **Perfect fit**: Designed exactly for this use case (small-to-medium datasets)

**Installation:**
```bash
npm install vectra
```

**Storage Structure:**
```
.obsidian/
└── plugins/
    └── agent-client/
        └── embeddings/
            ├── index.json          # Vector index + metadata
            └── [note-id].meta.json # Large metadata (optional)
```

## Implementation Plan

### Phase 1: Core Infrastructure (Week 1)

#### 1.1 Create Semantic Search Service

**File:** `src/adapters/obsidian/semantic-search.service.ts`

```typescript
import { pipeline } from '@xenova/transformers';
import { LocalIndex } from 'vectra';
import { TFile } from 'obsidian';
import type AgentClientPlugin from '../../plugin';
import { Logger } from '../../shared/logger';

interface NoteEmbeddingMetadata {
  path: string;
  basename: string;
  mtime: number; // For cache invalidation
  preview: string; // First 200 chars
}

export class SemanticSearchService {
  private plugin: AgentClientPlugin;
  private logger: Logger;
  private index: LocalIndex | null = null;
  private embedder: any = null; // transformers.js pipeline
  private isInitializing = false;
  private isReady = false;

  constructor(plugin: AgentClientPlugin) {
    this.plugin = plugin;
    this.logger = new Logger(plugin);
  }

  async initialize(): Promise<void> {
    if (this.isInitializing || this.isReady) return;
    
    this.isInitializing = true;
    try {
      // Initialize embedder
      this.embedder = await pipeline(
        'feature-extraction',
        'Xenova/all-MiniLM-L6-v2'
      );

      // Initialize vector index
      const indexPath = this.plugin.app.vault.configDir + 
                        '/plugins/agent-client/embeddings';
      this.index = new LocalIndex(indexPath);

      // Check if index exists, create if not
      if (!(await this.index.isIndexCreated())) {
        await this.index.createIndex();
        this.logger.log('[SemanticSearch] Created new vector index');
      } else {
        this.logger.log('[SemanticSearch] Loaded existing vector index');
      }

      this.isReady = true;
    } catch (error) {
      this.logger.log('[SemanticSearch] Initialization failed:', error);
      throw error;
    } finally {
      this.isInitializing = false;
    }
  }

  async indexNote(file: TFile): Promise<void> {
    if (!this.isReady) return;

    try {
      const content = await this.plugin.app.vault.cachedRead(file);
      
      // Create searchable text (title + content preview)
      const searchText = `${file.basename}\n${content.slice(0, 1000)}`;
      
      // Generate embedding
      const embedding = await this.embedder(searchText, {
        pooling: 'mean',
        normalize: true
      });

      // Store in vector database
      const metadata: NoteEmbeddingMetadata = {
        path: file.path,
        basename: file.basename,
        mtime: file.stat.mtime,
        preview: content.slice(0, 200)
      };

      await this.index.upsertItem({
        id: file.path,
        vector: Array.from(embedding.data),
        metadata
      });

      this.logger.log(`[SemanticSearch] Indexed: ${file.basename}`);
    } catch (error) {
      this.logger.log(`[SemanticSearch] Failed to index ${file.path}:`, error);
    }
  }

  async search(query: string, limit = 20): Promise<TFile[]> {
    if (!this.isReady || !query.trim()) return [];

    try {
      // Generate query embedding
      const queryEmbedding = await this.embedder(query, {
        pooling: 'mean',
        normalize: true
      });

      // Search vector database
      const results = await this.index.queryItems(
        Array.from(queryEmbedding.data),
        limit
      );

      // Convert results to TFiles
      const files: TFile[] = [];
      for (const result of results) {
        const file = this.plugin.app.vault.getAbstractFileByPath(
          result.item.metadata.path
        );
        if (file instanceof TFile) {
          files.push(file);
        }
      }

      return files;
    } catch (error) {
      this.logger.log('[SemanticSearch] Search failed:', error);
      return [];
    }
  }

  async rebuildIndex(files: TFile[]): Promise<void> {
    if (!this.isReady) return;

    this.logger.log('[SemanticSearch] Rebuilding index...');
    
    // Clear existing index
    await this.index.deleteIndex();
    await this.index.createIndex();

    // Index all files
    for (const file of files) {
      await this.indexNote(file);
    }

    this.logger.log(`[SemanticSearch] Indexed ${files.length} notes`);
  }

  destroy(): void {
    this.isReady = false;
    this.embedder = null;
    this.index = null;
  }
}
```

#### 1.2 Enhance NoteMentionService

**File:** `src/adapters/obsidian/mention-service.ts`

Add hybrid search combining fuzzy + semantic:

```typescript
import { SemanticSearchService } from './semantic-search.service';

export class NoteMentionService {
  private semanticSearch: SemanticSearchService;
  
  constructor(plugin: AgentClientPlugin) {
    // ... existing code ...
    this.semanticSearch = new SemanticSearchService(plugin);
    
    // Initialize semantic search asynchronously (don't block)
    this.semanticSearch.initialize().catch(err => {
      this.logger.log('[NoteMentionService] Semantic search unavailable:', err);
    });
  }

  async searchNotes(query: string): Promise<TFile[]> {
    // Existing fuzzy search (fast, reliable)
    const fuzzyResults = this.searchNotesFuzzy(query);
    
    // Try semantic search (slower, but more intelligent)
    let semanticResults: TFile[] = [];
    try {
      semanticResults = await this.semanticSearch.search(query, 10);
    } catch (error) {
      // Fallback to fuzzy-only if semantic fails
      this.logger.log('[NoteMentionService] Semantic search failed:', error);
    }

    // Merge results: prioritize fuzzy matches, add semantic matches
    return this.mergeResults(fuzzyResults, semanticResults);
  }

  private searchNotesFuzzy(query: string): TFile[] {
    // Existing implementation (unchanged)
    // ...
  }

  private mergeResults(fuzzy: TFile[], semantic: TFile[]): TFile[] {
    const seen = new Set<string>();
    const merged: TFile[] = [];

    // Add fuzzy results first (they match user's exact query)
    for (const file of fuzzy) {
      seen.add(file.path);
      merged.push(file);
    }

    // Add semantic results that aren't already included
    for (const file of semantic) {
      if (!seen.has(file.path)) {
        merged.push(file);
        if (merged.length >= 20) break;
      }
    }

    return merged.slice(0, 20);
  }
}
```

### Phase 2: Incremental Indexing (Week 2)

Add event listeners to keep embeddings up-to-date:

```typescript
constructor(plugin: AgentClientPlugin) {
  // ... existing code ...
  
  // Listen for vault changes
  this.eventRefs.push(
    this.plugin.app.vault.on('create', (file) => {
      if (file instanceof TFile && file.extension === 'md') {
        this.semanticSearch.indexNote(file);
      }
    })
  );

  this.eventRefs.push(
    this.plugin.app.vault.on('modify', (file) => {
      if (file instanceof TFile && file.extension === 'md') {
        this.semanticSearch.indexNote(file);
      }
    })
  );

  this.eventRefs.push(
    this.plugin.app.vault.on('delete', (file) => {
      if (file instanceof TFile) {
        this.semanticSearch.deleteNote(file.path);
      }
    })
  );
}
```

### Phase 3: Settings & Optimization (Week 3)

#### 3.1 Add Settings UI

**File:** `src/components/settings/AgentClientSettingTab.ts`

```typescript
// Add to plugin settings interface
interface AgentClientSettings {
  // ... existing settings ...
  semanticSearchEnabled: boolean;
  semanticSearchRebuildOnStartup: boolean;
}

// Add to settings tab
containerEl.createEl('h3', { text: 'Semantic Search' });

new Setting(containerEl)
  .setName('Enable semantic search')
  .setDesc('Use AI embeddings for intelligent note suggestions. Requires ~50MB storage.')
  .addToggle(toggle => toggle
    .setValue(this.plugin.settings.semanticSearchEnabled)
    .onChange(async (value) => {
      this.plugin.settings.semanticSearchEnabled = value;
      await this.plugin.saveSettings();
    })
  );

new Setting(containerEl)
  .setName('Rebuild index on startup')
  .setDesc('Re-index all notes when Obsidian starts (slower startup, ensures accuracy)')
  .addToggle(toggle => toggle
    .setValue(this.plugin.settings.semanticSearchRebuildOnStartup)
    .onChange(async (value) => {
      this.plugin.settings.semanticSearchRebuildOnStartup = value;
      await this.plugin.saveSettings();
    })
  );

new Setting(containerEl)
  .setName('Rebuild semantic index now')
  .setDesc('Manually rebuild the entire embedding index')
  .addButton(button => button
    .setButtonText('Rebuild')
    .onClick(async () => {
      new Notice('Rebuilding semantic search index...');
      const files = this.plugin.app.vault.getMarkdownFiles();
      await this.plugin.noteMentionService.semanticSearch.rebuildIndex(files);
      new Notice('Semantic search index rebuilt!');
    })
  );
```

#### 3.2 Performance Optimizations

1. **Lazy initialization**: Load embedder only when first needed
2. **Batch processing**: Index multiple notes in parallel
3. **Caching**: Skip re-indexing if file hasn't changed (check mtime)
4. **Debouncing**: Wait 2s after file modification before re-indexing

```typescript
private modifyDebounce = new Map<string, NodeJS.Timeout>();

onFileModify(file: TFile): void {
  // Clear existing timeout
  const existing = this.modifyDebounce.get(file.path);
  if (existing) clearTimeout(existing);

  // Set new timeout
  const timeout = setTimeout(() => {
    this.semanticSearch.indexNote(file);
    this.modifyDebounce.delete(file.path);
  }, 2000);

  this.modifyDebounce.set(file.path, timeout);
}
```

## Performance Characteristics

Based on benchmarks with similar setups:

| Metric | Expected Performance |
|--------|---------------------|
| **Initial model load** | 2-3 seconds (one-time) |
| **Embedding generation** | 50-100ms per note |
| **Index 1000 notes** | ~2-3 minutes (one-time) |
| **Search query** | <50ms (sub-10ms typical) |
| **Storage overhead** | ~50KB per 1000 notes |
| **Memory usage** | ~100MB additional RAM |

### Vault Size Guidelines

- **Small vaults (< 500 notes)**: Excellent experience, instant results
- **Medium vaults (500-2000 notes)**: Great experience, 1-2 min initial indexing
- **Large vaults (2000-5000 notes)**: Good experience, 5-10 min initial indexing
- **Very large vaults (> 5000 notes)**: Consider selective indexing or lazy loading

## Alternative Approaches Considered

### Option 2: LanceDB (Not Recommended)

**Pros:**
- Scales to petabyte-level datasets
- Supports multimodal embeddings
- Hybrid search built-in

**Cons:**
- Overkill for typical Obsidian vaults (< 10,000 notes)
- More complex API and setup
- Larger binary size (~10MB vs Vectra's ~100KB)

**Verdict:** Too heavy for this use case. Consider only if targeting power users with 10K+ notes.

### Option 3: Ollama Embeddings (Not Recommended)

**Pros:**
- Best-in-class embedding quality
- Supports many models

**Cons:**
- Requires separate Ollama service installation
- Added complexity for users
- Not truly "built-in" to the plugin

**Verdict:** Violates the "simple & maintainable" goal. Could be offered as an advanced option later.

### Option 4: BM25 (Considered but Insufficient)

**Pros:**
- Pure JavaScript, no ML model needed
- Extremely fast

**Cons:**
- Still keyword-based (not semantic)
- Only marginally better than fuzzy search

**Verdict:** Doesn't achieve the semantic understanding goal.

## Testing Strategy

### Unit Tests

```typescript
describe('SemanticSearchService', () => {
  it('should find semantically similar notes', async () => {
    await service.indexNote(createMockFile('productivity.md', 'GTD system'));
    const results = await service.search('time management');
    expect(results).toContain('productivity.md');
  });

  it('should handle empty queries', async () => {
    const results = await service.search('');
    expect(results).toEqual([]);
  });
});
```

### Integration Tests

1. Test with real vault (100-1000 notes)
2. Measure search latency (should be <100ms)
3. Test incremental updates (add/modify/delete notes)
4. Verify cache invalidation

### User Acceptance Testing

1. **Semantic understanding**: Search for "meetings" and verify "Standup Notes" appears
2. **Fallback behavior**: Disable semantic search, verify fuzzy search still works
3. **Performance**: Measure time from typing `@` to seeing suggestions
4. **Storage**: Verify `.obsidian/plugins/agent-client/embeddings/` size is reasonable

## Migration Path

### For Existing Users

1. **v1.0 (Initial Release)**:
   - Semantic search opt-in (disabled by default)
   - Notice on first enable: "Building semantic search index... This may take a few minutes."
   - Progress indicator during indexing

2. **v1.1 (Refinement)**:
   - Enable by default for vaults < 1000 notes
   - Add "smart indexing" (prioritize recently modified notes)

3. **v2.0 (Future)**:
   - Index note content (not just title + preview)
   - Support for images (multimodal embeddings)
   - Cross-vault search

### Rollback Plan

If semantic search causes issues:
1. Add setting to disable entirely
2. Gracefully degrade to fuzzy-only search
3. Add command to clear embeddings folder

## Estimated Development Time

| Phase | Tasks | Time |
|-------|-------|------|
| **Phase 1** | Core infrastructure, basic semantic search | 3-4 days |
| **Phase 2** | Incremental indexing, vault event handling | 2 days |
| **Phase 3** | Settings UI, optimization, testing | 2-3 days |
| **Total** | End-to-end implementation | **1-2 weeks** |

Additional time for:
- Documentation updates: 1 day
- User testing and feedback: 1 week
- Bug fixes and refinement: Ongoing

## Dependencies Added

```json
{
  "dependencies": {
    "@xenova/transformers": "^2.17.0",
    "vectra": "^0.9.0"
  }
}
```

**Total bundle size impact:** ~150KB (minified + gzipped)

**Model download:** ~23MB (one-time, cached by transformers.js)

## Security & Privacy Considerations

✅ **All computation happens locally** - No data sent to external servers  
✅ **Model downloaded from official Hugging Face** - Trusted source  
✅ **Embeddings stored in `.obsidian` folder** - Same security as other plugin data  
✅ **No telemetry or analytics** - Zero network requests after model download  
✅ **User controls indexing** - Can disable or delete index anytime  

## Maintenance Considerations

### Long-term Sustainability

1. **Dependencies**: Both `@xenova/transformers` and `vectra` are actively maintained
2. **Model updates**: MiniLM-L6-v2 is stable, unlikely to change
3. **Breaking changes**: Minimal risk (ONNX and vector search APIs are stable)

### Future Enhancements

1. **Better ranking**: Combine semantic + fuzzy scores with learned weights
2. **Query expansion**: Use embeddings to suggest related search terms
3. **Clustering**: Group related notes visually
4. **Contextual search**: Bias results based on current note's topic

## Conclusion

This recommendation provides a **pragmatic, production-ready approach** to adding semantic search to the Agent Client plugin. The combination of:

- **Xenova transformers.js** (proven, local embedding model)
- **Vectra** (simple, performant vector storage)
- **Hybrid search** (fuzzy fallback for reliability)

...delivers intelligent note suggestions while maintaining the plugin's core values of simplicity, privacy, and maintainability.

**Key Benefits:**
- ✅ 100% local, no API dependencies
- ✅ Simple implementation (~500 lines of code)
- ✅ Fast enough for real-time use (<50ms queries)
- ✅ Graceful degradation (falls back to fuzzy search)
- ✅ Minimal storage overhead (~50KB per 1000 notes)

**Recommended Next Steps:**
1. Prototype Phase 1 in a feature branch
2. Test with various vault sizes (100, 1K, 5K notes)
3. Gather feedback from beta users
4. Iterate based on performance metrics and user feedback

## References

- [Xenova transformers.js Documentation](https://huggingface.co/docs/transformers.js)
- [all-MiniLM-L6-v2 Model Card](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2)
- [Vectra GitHub Repository](https://github.com/Stevenic/vectra)
- [Vector Search Best Practices](https://www.pinecone.io/learn/vector-search/)
- [Semantic Search Guide](https://www.sbert.net/examples/applications/semantic-search/README.html)
