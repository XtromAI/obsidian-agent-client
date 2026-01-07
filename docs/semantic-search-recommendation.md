# Semantic Search Implementation Recommendation

## Executive Summary

This document provides a comprehensive recommendation for implementing local semantic search in the **obsidian-agent-client** plugin. The proposed system combines vector-based semantic search with lexical (keyword-based) search to provide powerful note retrieval capabilities for AI agents, enabling context-aware conversations without requiring external API calls.

**Key Design Principles:**
- **100% Local Operation**: All embeddings and search operations run locally (no API dependencies)
- **Architectural Integration**: Follows the existing React Hooks Architecture pattern
- **Memory Efficient**: Chunked storage with configurable memory budgets
- **Incremental Updates**: Only re-index changed notes
- **Agent-Focused**: Optimized for providing relevant context to AI agents during conversations

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Package Dependencies](#package-dependencies)
3. [Data Storage Strategy](#data-storage-strategy)
4. [Chunking Strategy](#chunking-strategy)
5. [Embedding Models (Local)](#embedding-models-local)
6. [Vector Search Implementation](#vector-search-implementation)
7. [Lexical Search Implementation](#lexical-search-implementation)
8. [Integration with Existing Architecture](#integration-with-existing-architecture)
9. [Implementation Roadmap](#implementation-roadmap)
10. [Performance Considerations](#performance-considerations)

---

## Architecture Overview

The semantic search system will operate in two complementary modes:

### Hybrid Search Architecture

```
User Query (from Chat or Agent)
    ↓
┌────────────────────────────────────┐
│   Query Processing                 │
│   (Normalize, expand variants)     │
└────────────────────────────────────┘
    ↓
┌────────────────────────────────────┐
│   Parallel Retrieval               │
│   ├─ Vector Search (Embeddings)    │
│   │   • Ollama/LM Studio           │
│   │   • In-memory vector store     │
│   └─ Lexical Search (Keywords)     │
│       • FlexSearch engine          │
│       • Tag/path boosting          │
└────────────────────────────────────┘
    ↓
┌────────────────────────────────────┐
│   Result Merging & Ranking         │
│   (Weighted score combination)     │
└────────────────────────────────────┘
    ↓
    Top K Documents → Agent Context
```

### System Components

1. **Embedding Manager**: Manages local embedding models (Ollama/LM Studio)
2. **Vector Store**: In-memory storage with disk persistence
3. **Lexical Engine**: FlexSearch-based full-text search
4. **Chunk Manager**: Deterministic chunking with caching
5. **Index Manager**: Incremental indexing, garbage collection
6. **Search Coordinator**: Merges semantic + lexical results

---

## Package Dependencies

### Core Search Libraries

```json
{
  "@orama/orama": "^3.0.0",              // Vector database with HNSW
  "flexsearch": "^0.8.205",               // Full-text search engine
  "fuzzysort": "^3.1.0"                   // Fuzzy matching for mentions
}
```

### LangChain Ecosystem (Optional - for advanced features)

```json
{
  "langchain": "^0.3.12",                 // Core LangChain library
  "@langchain/core": "^0.3.25",           // Core abstractions
  "@langchain/textsplitters": "^0.1.0"    // Text chunking utilities
}
```

### Local Embedding Providers

```json
{
  "@langchain/ollama": "^0.1.5",          // Ollama integration (PRIMARY)
  "ollama": "^0.5.15"                     // Native Ollama SDK (alternative)
}
```

### Utilities

```json
{
  "crypto-js": "^4.2.0",                  // Hashing for document IDs
  "async-mutex": "^0.5.0",                // Concurrency control
  "p-queue": "^8.1.0"                     // Rate limiting
}
```

**Rationale:**
- **Ollama only**: Focus on Ollama as the primary local embedding provider (simplest, most reliable)
- **No API dependencies**: All packages work offline
- **Small bundle size**: Keep plugin lightweight (~500KB additional)

---

## Data Storage Strategy

### Storage Location

**Vault-based Storage** (recommended for obsidian-agent-client):

```
.obsidian/plugins/agent-client/
  ├── search-index/
  │   ├── metadata.json              // Index metadata
  │   ├── chunks-0.jsonl             // Chunked documents (JSONL format)
  │   ├── chunks-1.jsonl
  │   ├── chunks-N.jsonl
  │   └── embeddings/                // Separate embeddings directory
  │       ├── vectors-0.bin          // Binary embedding storage
  │       ├── vectors-1.bin
  │       └── vectors-N.bin
```

### Metadata File Structure

```typescript
// metadata.json
{
  "version": "1.0.0",
  "embeddingModel": "nomic-embed-text",
  "vectorLength": 768,
  "numPartitions": 4,
  "totalChunks": 1523,
  "lastIndexed": 1704632400000,
  "chunkSize": 6000,
  "schema": {
    "id": "string",
    "notePath": "string",
    "title": "string",
    "content": "string",
    "embedding": "vector[768]",
    "chunkIndex": "number",
    "heading": "string",
    "contentHash": "string",
    "mtime": "number",
    "tags": "string[]"
  }
}
```

### Chunk File Format (JSONL)

```jsonl
{"id":"note.md#0","notePath":"Projects/AI.md","title":"AI Projects","content":"NOTE TITLE: [[AI Projects]]\n\nNOTE BLOCK CONTENT:\n\n## Introduction\nThis document...","contentHash":"abc123","heading":"Introduction","chunkIndex":0,"mtime":1704632400000,"tags":["#ai","#projects"]}
{"id":"note.md#1","notePath":"Projects/AI.md","title":"AI Projects","content":"NOTE TITLE: [[AI Projects]]\n\nNOTE BLOCK CONTENT:\n\n## Methods\nWe use...","contentHash":"def456","heading":"Methods","chunkIndex":1,"mtime":1704632400000,"tags":["#ai","#methods"]}
```

**Benefits:**
- **JSONL format**: One JSON object per line, easy to stream and append
- **Partitioned**: Split large indexes across multiple files for memory efficiency
- **Binary embeddings**: Store vectors separately in efficient binary format
- **Incremental updates**: Only rewrite changed partitions

### Embedding Storage Format

```typescript
// Binary format for vectors (Float32Array)
// Format: [vector_count: uint32][dimension: uint32][vector1][vector2]...[vectorN]
// Each vector: [float32, float32, ..., float32] (dimension times)

interface EmbeddingMetadata {
  id: string;          // Chunk ID
  offset: number;      // Byte offset in binary file
  partition: number;   // Which binary file (0-N)
}
```

---

## Chunking Strategy

### Configuration

```typescript
const CHUNK_CONFIG = {
  size: 6000,           // Characters per chunk
  overlap: 0,           // No overlap (deterministic)
  separators: [
    "\n\n",             // Paragraphs first
    "\n",               // Lines
    ". ",               // Sentences
    " ",                // Words
    ""                  // Characters (fallback)
  ]
};
```

### Heading-First Chunking Algorithm

The system uses a sophisticated **heading-first** approach to maintain semantic coherence:

#### Step 1: Parse Document Structure

```typescript
// Extract headings using Obsidian metadata cache
const cache = app.metadataCache.getFileCache(file);
const headings = cache?.headings || [];

// Build section hierarchy
const sections = headings.map((heading, index) => ({
  level: heading.level,
  title: heading.heading,
  startLine: heading.position.start.line,
  endLine: headings[index + 1]?.position.start.line || fileLineCount
}));
```

#### Step 2: Section-Based Chunking

```
Document: "AI Research.md"
  ├─ # Introduction (lines 1-50)
  │   └─ Content ≤ 6000 chars → Single chunk
  ├─ # Methods (lines 51-200)
  │   └─ Content > 6000 chars → Split into 2 chunks
  │       ├─ Chunk 0: Methods (Part 1)
  │       └─ Chunk 1: Methods (Part 2)
  └─ # Results (lines 201-250)
      └─ Content ≤ 6000 chars → Single chunk
```

**Logic:**
1. If document has no headings → treat as single section
2. For each section:
   - Extract content from heading to next heading (or EOF)
   - If section ≤ 6000 chars → single chunk
   - If section > 6000 chars → split by paragraphs using recursive splitter

#### Step 3: Contextual Headers

Each chunk includes contextual information:

```markdown
NOTE TITLE: [[note_title]]

NOTE BLOCK CONTENT:

[actual chunk content with heading preserved]
```

This pattern helps maintain context when chunks are retrieved separately.

### Chunk ID Generation

**Format**: `{note_path}#{chunk_index}`

**Examples:**
- `"Projects/AI/notes.md#0"`
- `"Projects/AI/notes.md#1"`
- `"Daily Notes/2026-01-07.md#0"`

**Properties:**
- Deterministic (same note always produces same IDs)
- Sortable (chunks appear in document order)
- No padding (unlimited chunks per note)

### Implementation Example

```typescript
interface Chunk {
  id: string;              // "note_path#chunk_index"
  notePath: string;        // Original note path
  chunkIndex: number;      // 0-based chunk position
  content: string;         // Chunk text with headers
  contentHash: string;     // MD5 hash for change detection
  title: string;           // Note title (basename)
  heading: string;         // Section heading (if any)
  mtime: number;           // Note modification time
  tags: string[];          // Extracted tags
}

class ChunkManager {
  private cache: Map<string, Chunk[]> = new Map();
  
  async getChunksForNote(file: TFile): Promise<Chunk[]> {
    // Check cache first
    if (this.cache.has(file.path)) {
      const cached = this.cache.get(file.path)!;
      const currentMtime = file.stat.mtime;
      
      if (cached[0]?.mtime === currentMtime) {
        return cached; // Cache hit, no changes
      }
    }
    
    // Generate chunks
    const content = await app.vault.read(file);
    const chunks = await this.chunkDocument(file, content);
    
    // Update cache
    this.cache.set(file.path, chunks);
    
    return chunks;
  }
  
  private async chunkDocument(file: TFile, content: string): Promise<Chunk[]> {
    const cache = app.metadataCache.getFileCache(file);
    const headings = cache?.headings || [];
    const tags = cache?.tags?.map(t => t.tag) || [];
    
    if (headings.length === 0) {
      // No headings: single chunk
      return [{
        id: `${file.path}#0`,
        notePath: file.path,
        chunkIndex: 0,
        content: this.addContext(file.basename, content),
        contentHash: MD5(content).toString(),
        title: file.basename,
        heading: "",
        mtime: file.stat.mtime,
        tags
      }];
    }
    
    // Chunk by sections
    const chunks: Chunk[] = [];
    
    for (let i = 0; i < headings.length; i++) {
      const heading = headings[i];
      const nextHeading = headings[i + 1];
      
      // Extract section content
      const sectionContent = this.extractSection(
        content, 
        heading.position.start.offset,
        nextHeading?.position.start.offset || content.length
      );
      
      // Split if too large
      const sectionChunks = this.splitLargeSection(sectionContent);
      
      for (let j = 0; j < sectionChunks.length; j++) {
        chunks.push({
          id: `${file.path}#${chunks.length}`,
          notePath: file.path,
          chunkIndex: chunks.length,
          content: this.addContext(file.basename, sectionChunks[j]),
          contentHash: MD5(sectionChunks[j]).toString(),
          title: file.basename,
          heading: heading.heading,
          mtime: file.stat.mtime,
          tags
        });
      }
    }
    
    return chunks;
  }
  
  private addContext(title: string, content: string): string {
    return `NOTE TITLE: [[${title}]]\n\nNOTE BLOCK CONTENT:\n\n${content}`;
  }
  
  private splitLargeSection(content: string): string[] {
    if (content.length <= 6000) {
      return [content];
    }
    
    // Use recursive splitter
    const splitter = new RecursiveCharacterTextSplitter({
      chunkSize: 6000,
      chunkOverlap: 0,
      separators: ["\n\n", "\n", ". ", " ", ""]
    });
    
    return splitter.splitText(content);
  }
}
```

---

## Embedding Models (Local)

### Recommended: Ollama

**Why Ollama:**
- Easy installation and management
- Automatic model updates
- OpenAI-compatible API
- Excellent model selection
- Built-in model quantization

### Installation & Setup

```bash
# Install Ollama (one-time)
curl -fsSL https://ollama.com/install.sh | sh

# Pull recommended embedding model
ollama pull nomic-embed-text

# Verify installation
ollama list
```

### Recommended Models

| Model | Parameters | Dimensions | Speed | Quality | Use Case |
|-------|-----------|------------|-------|---------|----------|
| **nomic-embed-text** | 137M | 768 | Fast | High | **Recommended** - Best balance |
| mxbai-embed-large | 335M | 1024 | Medium | Highest | High accuracy needs |
| all-minilm | 33M | 384 | Very Fast | Good | Resource-constrained |
| snowflake-arctic-embed | 137M | 768 | Fast | High | Multilingual support |

**Default**: `nomic-embed-text` (best balance of speed, quality, and size)

### Integration Code

```typescript
import { Ollama } from 'ollama';

class LocalEmbeddingManager {
  private ollama: Ollama;
  private model: string;
  
  constructor(model: string = 'nomic-embed-text') {
    this.ollama = new Ollama({
      host: 'http://localhost:11434'
    });
    this.model = model;
  }
  
  async isAvailable(): Promise<boolean> {
    try {
      await this.ollama.list();
      return true;
    } catch (error) {
      return false;
    }
  }
  
  async embedQuery(text: string): Promise<number[]> {
    const response = await this.ollama.embeddings({
      model: this.model,
      prompt: text
    });
    
    return response.embedding;
  }
  
  async embedDocuments(texts: string[]): Promise<number[][]> {
    // Process in batches to avoid memory issues
    const batchSize = 32;
    const embeddings: number[][] = [];
    
    for (let i = 0; i < texts.length; i += batchSize) {
      const batch = texts.slice(i, i + batchSize);
      
      const batchEmbeddings = await Promise.all(
        batch.map(text => this.embedQuery(text))
      );
      
      embeddings.push(...batchEmbeddings);
      
      // Rate limiting (optional)
      await this.sleep(100);
    }
    
    return embeddings;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### Fallback: LM Studio

If Ollama is not available, fall back to LM Studio:

```typescript
class LMStudioEmbeddings {
  private baseUrl: string;
  
  constructor(baseUrl: string = 'http://localhost:1234/v1') {
    this.baseUrl = baseUrl;
  }
  
  async embedQuery(text: string): Promise<number[]> {
    const response = await fetch(`${this.baseUrl}/embeddings`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'text-embedding-ada-002', // Model name in LM Studio
        input: text
      })
    });
    
    const data = await response.json();
    return data.data[0].embedding;
  }
}
```

### Settings UI

Add to plugin settings:

```typescript
interface SemanticSearchSettings {
  enabled: boolean;
  embeddingProvider: 'ollama' | 'lmstudio';
  ollamaHost: string;          // Default: http://localhost:11434
  ollamaModel: string;         // Default: nomic-embed-text
  lmstudioHost: string;        // Default: http://localhost:1234/v1
  lmstudioModel: string;       // Default: text-embedding-ada-002
  chunkSize: number;           // Default: 6000
  maxResults: number;          // Default: 10
  minSimilarity: number;       // Default: 0.1
}
```

---

## Vector Search Implementation

### Orama Vector Database

**Why Orama:**
- Pure TypeScript (no native dependencies)
- HNSW algorithm (fast approximate search)
- Built-in vector support
- Small bundle size (~100KB)
- Works in browser (Obsidian desktop app)

### Database Schema

```typescript
import { create, insert, search } from '@orama/orama';

interface VectorDocument {
  id: string;              // Chunk ID
  notePath: string;        // Note path
  title: string;           // Note title
  content: string;         // Chunk content
  heading: string;         // Section heading
  embedding: number[];     // Vector embedding
  chunkIndex: number;      // Chunk position
  mtime: number;           // Modification time
  tags: string[];          // Note tags
}

const db = await create({
  schema: {
    id: 'string',
    notePath: 'string',
    title: 'string',
    content: 'string',
    heading: 'string',
    embedding: 'vector[768]',     // Dimension matches model
    chunkIndex: 'number',
    mtime: 'number',
    tags: 'string[]'
  } as const
});
```

### Indexing Pipeline

```typescript
class VectorIndexManager {
  private db: any;
  private embeddingManager: LocalEmbeddingManager;
  private chunkManager: ChunkManager;
  
  async indexVault(files: TFile[]): Promise<void> {
    // Filter eligible files
    const eligibleFiles = files.filter(f => 
      f.extension === 'md' && !this.shouldExclude(f.path)
    );
    
    console.log(`Indexing ${eligibleFiles.length} files...`);
    
    for (const file of eligibleFiles) {
      await this.indexFile(file);
    }
    
    console.log('Indexing complete!');
  }
  
  async indexFile(file: TFile): Promise<void> {
    // Get or generate chunks
    const chunks = await this.chunkManager.getChunksForNote(file);
    
    // Check if already indexed with current mtime
    const existing = await this.getExistingChunks(file.path);
    if (existing.length > 0 && existing[0].mtime === file.stat.mtime) {
      console.log(`Skipping ${file.path} (no changes)`);
      return;
    }
    
    // Remove old chunks
    await this.removeChunks(file.path);
    
    // Generate embeddings (batched)
    const contents = chunks.map(c => c.content);
    const embeddings = await this.embeddingManager.embedDocuments(contents);
    
    // Insert into vector database
    for (let i = 0; i < chunks.length; i++) {
      await insert(this.db, {
        id: chunks[i].id,
        notePath: chunks[i].notePath,
        title: chunks[i].title,
        content: chunks[i].content,
        heading: chunks[i].heading,
        embedding: embeddings[i],
        chunkIndex: chunks[i].chunkIndex,
        mtime: chunks[i].mtime,
        tags: chunks[i].tags
      });
    }
  }
  
  async removeChunks(notePath: string): Promise<void> {
    // Remove all chunks for a note path
    // (Orama API details depend on version)
  }
  
  async incrementalIndex(): Promise<void> {
    // Get all markdown files
    const files = this.app.vault.getMarkdownFiles();
    
    // Find files that need reindexing
    const toReindex: TFile[] = [];
    
    for (const file of files) {
      const existing = await this.getExistingChunks(file.path);
      
      if (existing.length === 0) {
        // New file
        toReindex.push(file);
      } else if (existing[0].mtime !== file.stat.mtime) {
        // Modified file
        toReindex.push(file);
      }
    }
    
    console.log(`Reindexing ${toReindex.length} files...`);
    await this.indexVault(toReindex);
  }
  
  async garbageCollect(): Promise<void> {
    // Remove chunks for deleted files
    const allChunks = await this.getAllChunks();
    const existingPaths = new Set(
      this.app.vault.getMarkdownFiles().map(f => f.path)
    );
    
    for (const chunk of allChunks) {
      if (!existingPaths.has(chunk.notePath)) {
        await this.removeChunks(chunk.notePath);
      }
    }
  }
  
  private shouldExclude(path: string): boolean {
    // Exclude patterns (from settings)
    const excludePatterns = [
      '.obsidian/',
      'node_modules/',
      '.trash/'
    ];
    
    return excludePatterns.some(pattern => path.startsWith(pattern));
  }
}
```

### Vector Search

```typescript
class VectorSearcher {
  private db: any;
  private embeddingManager: LocalEmbeddingManager;
  
  async search(
    query: string,
    options: {
      limit?: number;
      minSimilarity?: number;
      filterTags?: string[];
      filterPaths?: string[];
    } = {}
  ): Promise<SearchResult[]> {
    const {
      limit = 10,
      minSimilarity = 0.1,
      filterTags = [],
      filterPaths = []
    } = options;
    
    // Generate query embedding
    const queryEmbedding = await this.embeddingManager.embedQuery(query);
    
    // Vector search
    const results = await search(this.db, {
      mode: 'vector',
      vector: {
        value: queryEmbedding,
        property: 'embedding'
      },
      limit: limit * 2,  // Over-fetch for filtering
      similarity: minSimilarity
    });
    
    // Apply filters
    let filteredResults = results.hits;
    
    if (filterTags.length > 0) {
      filteredResults = filteredResults.filter(hit =>
        hit.document.tags.some(tag => filterTags.includes(tag))
      );
    }
    
    if (filterPaths.length > 0) {
      filteredResults = filteredResults.filter(hit =>
        filterPaths.some(path => hit.document.notePath.startsWith(path))
      );
    }
    
    // Limit results
    return filteredResults.slice(0, limit).map(hit => ({
      id: hit.document.id,
      notePath: hit.document.notePath,
      title: hit.document.title,
      content: hit.document.content,
      heading: hit.document.heading,
      score: hit.score,
      chunkIndex: hit.document.chunkIndex
    }));
  }
}

interface SearchResult {
  id: string;
  notePath: string;
  title: string;
  content: string;
  heading: string;
  score: number;
  chunkIndex: number;
}
```

### Similarity Metrics

Orama uses **cosine similarity** by default:

```
similarity(A, B) = (A · B) / (||A|| × ||B||)
```

**Range**: [-1, 1], where:
- 1.0 = identical vectors
- 0.0 = orthogonal (no similarity)
- -1.0 = opposite vectors

**Thresholds:**
- 0.7+ = High similarity (very relevant)
- 0.4-0.7 = Medium similarity (relevant)
- 0.1-0.4 = Low similarity (potentially relevant)
- < 0.1 = Not relevant (filter out)

---

## Lexical Search Implementation

### FlexSearch Engine

**Why FlexSearch:**
- Pure JavaScript
- Fast (~50x faster than Lunr.js)
- Memory efficient
- Multilingual support (CJK)
- Highly configurable

### Configuration

```typescript
import FlexSearch from 'flexsearch';

const lexicalIndex = new FlexSearch.Document({
  id: 'id',
  index: [
    { field: 'title', weight: 3 },      // Titles most important
    { field: 'heading', weight: 2.5 },  // Headings very important
    { field: 'tags', weight: 4 },       // Tags highly weighted
    { field: 'path', weight: 2 },       // Paths important
    { field: 'content', weight: 1 }     // Content baseline
  ],
  store: ['id', 'notePath', 'title', 'heading', 'chunkIndex'],
  tokenize: 'forward',                  // Forward matching
  resolution: 9,                        // Granularity (1-9)
  context: {
    depth: 2,                           // Context matching depth
    resolution: 3
  }
});
```

### Multilingual Tokenization

```typescript
class MixedTokenizer {
  tokenize(text: string): string[] {
    const tokens = new Set<string>();
    const lowered = text.toLowerCase();
    
    // ASCII words (standard tokenization)
    const asciiWords = lowered.match(/[a-z0-9]+/g) || [];
    asciiWords.forEach(word => tokens.add(word));
    
    // CJK bigrams (for Chinese, Japanese, Korean)
    const cjkChars = lowered.match(/[\u3040-\u309F\u30A0-\u30FF\u4E00-\u9FFF\uAC00-\uD7AF]+/g) || [];
    cjkChars.forEach(chars => {
      // Generate bigrams
      for (let i = 0; i < chars.length - 1; i++) {
        tokens.add(chars.substring(i, i + 2));
      }
      
      // Also add individual characters
      for (let i = 0; i < chars.length; i++) {
        tokens.add(chars[i]);
      }
    });
    
    return Array.from(tokens);
  }
}
```

### Lexical Search Implementation

```typescript
class LexicalSearchEngine {
  private index: FlexSearch.Document;
  private tokenizer: MixedTokenizer;
  
  constructor() {
    this.tokenizer = new MixedTokenizer();
    // Initialize FlexSearch index (see configuration above)
  }
  
  async indexChunks(chunks: Chunk[]): Promise<void> {
    for (const chunk of chunks) {
      await this.index.addAsync({
        id: chunk.id,
        notePath: chunk.notePath,
        title: chunk.title,
        heading: chunk.heading,
        tags: chunk.tags.join(' '),
        path: chunk.notePath,
        content: chunk.content
      });
    }
  }
  
  async search(
    query: string,
    options: {
      limit?: number;
      fields?: string[];
    } = {}
  ): Promise<SearchResult[]> {
    const { limit = 10, fields } = options;
    
    // Tokenize query
    const tokens = this.tokenizer.tokenize(query);
    
    // Search with all tokens
    const results = await this.index.searchAsync(
      tokens.join(' '),
      {
        limit,
        index: fields,  // Search specific fields if provided
        enrich: true,   // Include document data
        bool: 'or'      // Match any token
      }
    );
    
    // Flatten and score results
    const allResults: SearchResult[] = [];
    
    for (const fieldResults of results) {
      for (const result of fieldResults.result) {
        allResults.push({
          id: result.id,
          notePath: result.doc.notePath,
          title: result.doc.title,
          heading: result.doc.heading,
          content: '', // Load from chunk manager if needed
          score: this.calculateScore(result, tokens),
          chunkIndex: result.doc.chunkIndex
        });
      }
    }
    
    // Sort by score and deduplicate
    return this.deduplicateAndSort(allResults, limit);
  }
  
  private calculateScore(result: any, tokens: string[]): number {
    // Base score from FlexSearch
    let score = 1.0;
    
    // Boost based on:
    // 1. Exact matches
    // 2. Title/heading matches
    // 3. Tag matches
    // 4. Path matches
    
    return score;
  }
  
  private deduplicateAndSort(
    results: SearchResult[],
    limit: number
  ): SearchResult[] {
    const seen = new Set<string>();
    const unique: SearchResult[] = [];
    
    for (const result of results) {
      if (!seen.has(result.id)) {
        seen.add(result.id);
        unique.push(result);
      }
    }
    
    return unique
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }
}
```

### Graph-Based Boosting

Use Obsidian's link graph to boost related notes:

```typescript
class GraphBooster {
  constructor(private app: App) {}
  
  calculateBoost(notePath: string, queryPaths: Set<string>): number {
    let boost = 1.0;
    
    // Get resolved links for this note
    const cache = this.app.metadataCache.getCache(notePath);
    if (!cache) return boost;
    
    const resolvedLinks = this.app.metadataCache.resolvedLinks[notePath] || {};
    
    // Boost if this note links to query results
    for (const linkedPath in resolvedLinks) {
      if (queryPaths.has(linkedPath)) {
        boost += 0.1 * resolvedLinks[linkedPath]; // Weight by link count
      }
    }
    
    // Boost if query results link to this note
    for (const queryPath of queryPaths) {
      const queryLinks = this.app.metadataCache.resolvedLinks[queryPath] || {};
      if (queryLinks[notePath]) {
        boost += 0.15 * queryLinks[notePath];
      }
    }
    
    return Math.min(boost, 2.0); // Cap at 2x boost
  }
}
```

---

## Integration with Existing Architecture

### New Directories and Files

```
src/
├── domain/
│   ├── models/
│   │   ├── search-result.ts          // NEW: Search result model
│   │   └── search-settings.ts        // NEW: Search settings model
│   └── ports/
│       └── semantic-search.port.ts   // NEW: Search interface
│
├── hooks/
│   └── useSemanticSearch.ts          // NEW: Search hook
│
├── adapters/
│   └── search/                       // NEW: Search adapters
│       ├── embedding.adapter.ts      // Ollama/LM Studio
│       ├── vector-store.adapter.ts   // Orama integration
│       ├── lexical.adapter.ts        // FlexSearch integration
│       └── hybrid-search.adapter.ts  // Merges semantic + lexical
│
├── shared/
│   ├── chunk-manager.ts              // NEW: Chunking logic
│   └── search-utils.ts               // NEW: Search utilities
│
└── components/
    └── chat/
        └── SearchResultsView.tsx     // NEW: Display search results
```

### Domain Layer

```typescript
// src/domain/models/search-result.ts
export interface SearchResult {
  id: string;
  notePath: string;
  title: string;
  content: string;
  heading: string;
  score: number;
  chunkIndex: number;
  source: 'semantic' | 'lexical' | 'hybrid';
}

// src/domain/models/search-settings.ts
export interface SemanticSearchSettings {
  enabled: boolean;
  embeddingProvider: 'ollama' | 'lmstudio';
  ollamaHost: string;
  ollamaModel: string;
  lmstudioHost: string;
  lmstudioModel: string;
  chunkSize: number;
  maxResults: number;
  minSimilarity: number;
  useHybridSearch: boolean;
  semanticWeight: number;  // 0.0-1.0
  lexicalWeight: number;   // 0.0-1.0
}

// src/domain/ports/semantic-search.port.ts
export interface ISemanticSearch {
  search(query: string, options?: SearchOptions): Promise<SearchResult[]>;
  indexVault(): Promise<void>;
  incrementalIndex(): Promise<void>;
  isReady(): Promise<boolean>;
}

export interface SearchOptions {
  limit?: number;
  minSimilarity?: number;
  filterTags?: string[];
  filterPaths?: string[];
  mode?: 'semantic' | 'lexical' | 'hybrid';
}
```

### Hook Layer

```typescript
// src/hooks/useSemanticSearch.ts
import { useState, useCallback, useEffect } from 'react';
import { ISemanticSearch } from '../domain/ports/semantic-search.port';
import { SearchResult } from '../domain/models/search-result';

export function useSemanticSearch(searchAdapter: ISemanticSearch | null) {
  const [results, setResults] = useState<SearchResult[]>([]);
  const [isSearching, setIsSearching] = useState(false);
  const [isReady, setIsReady] = useState(false);
  
  useEffect(() => {
    if (searchAdapter) {
      searchAdapter.isReady().then(setIsReady);
    }
  }, [searchAdapter]);
  
  const search = useCallback(async (query: string, options?: SearchOptions) => {
    if (!searchAdapter || !isReady) {
      console.log('Search adapter not ready');
      return [];
    }
    
    setIsSearching(true);
    
    try {
      const results = await searchAdapter.search(query, options);
      setResults(results);
      return results;
    } catch (error) {
      console.error('Search error:', error);
      return [];
    } finally {
      setIsSearching(false);
    }
  }, [searchAdapter, isReady]);
  
  const indexVault = useCallback(async () => {
    if (!searchAdapter) return;
    await searchAdapter.indexVault();
  }, [searchAdapter]);
  
  const incrementalIndex = useCallback(async () => {
    if (!searchAdapter) return;
    await searchAdapter.incrementalIndex();
  }, [searchAdapter]);
  
  return {
    results,
    isSearching,
    isReady,
    search,
    indexVault,
    incrementalIndex
  };
}
```

### Adapter Layer

```typescript
// src/adapters/search/hybrid-search.adapter.ts
import { ISemanticSearch, SearchOptions } from '../../domain/ports/semantic-search.port';
import { SearchResult } from '../../domain/models/search-result';

export class HybridSearchAdapter implements ISemanticSearch {
  constructor(
    private vectorSearch: VectorSearchAdapter,
    private lexicalSearch: LexicalSearchAdapter,
    private semanticWeight: number = 0.6,
    private lexicalWeight: number = 0.4
  ) {}
  
  async search(query: string, options?: SearchOptions): Promise<SearchResult[]> {
    const mode = options?.mode || 'hybrid';
    
    if (mode === 'semantic') {
      return this.vectorSearch.search(query, options);
    }
    
    if (mode === 'lexical') {
      return this.lexicalSearch.search(query, options);
    }
    
    // Hybrid mode: run both in parallel
    const [semanticResults, lexicalResults] = await Promise.all([
      this.vectorSearch.search(query, { ...options, limit: options?.limit || 10 }),
      this.lexicalSearch.search(query, { ...options, limit: options?.limit || 10 })
    ]);
    
    // Merge results with weighted scoring
    return this.mergeResults(semanticResults, lexicalResults, options?.limit || 10);
  }
  
  private mergeResults(
    semanticResults: SearchResult[],
    lexicalResults: SearchResult[],
    limit: number
  ): SearchResult[] {
    const merged = new Map<string, SearchResult>();
    
    // Add semantic results
    for (const result of semanticResults) {
      merged.set(result.id, {
        ...result,
        score: result.score * this.semanticWeight,
        source: 'semantic'
      });
    }
    
    // Add/merge lexical results
    for (const result of lexicalResults) {
      const existing = merged.get(result.id);
      
      if (existing) {
        // Found by both: combine scores
        existing.score += result.score * this.lexicalWeight;
        existing.source = 'hybrid';
      } else {
        // Found only by lexical
        merged.set(result.id, {
          ...result,
          score: result.score * this.lexicalWeight,
          source: 'lexical'
        });
      }
    }
    
    // Sort by combined score
    return Array.from(merged.values())
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }
  
  async indexVault(): Promise<void> {
    await Promise.all([
      this.vectorSearch.indexVault(),
      this.lexicalSearch.indexVault()
    ]);
  }
  
  async incrementalIndex(): Promise<void> {
    await Promise.all([
      this.vectorSearch.incrementalIndex(),
      this.lexicalSearch.incrementalIndex()
    ]);
  }
  
  async isReady(): Promise<boolean> {
    const [vectorReady, lexicalReady] = await Promise.all([
      this.vectorSearch.isReady(),
      this.lexicalSearch.isReady()
    ]);
    
    return vectorReady && lexicalReady;
  }
}
```

### Component Integration

```typescript
// In src/components/chat/ChatView.tsx

// Add search adapter initialization
const searchAdapter = useMemo(() => {
  if (!settings.semanticSearch.enabled) return null;
  
  const embeddingAdapter = new EmbeddingAdapter(
    settings.semanticSearch.embeddingProvider,
    {
      ollamaHost: settings.semanticSearch.ollamaHost,
      ollamaModel: settings.semanticSearch.ollamaModel,
      lmstudioHost: settings.semanticSearch.lmstudioHost,
      lmstudioModel: settings.semanticSearch.lmstudioModel
    }
  );
  
  const vectorAdapter = new VectorSearchAdapter(embeddingAdapter, app);
  const lexicalAdapter = new LexicalSearchAdapter(app);
  
  return new HybridSearchAdapter(
    vectorAdapter,
    lexicalAdapter,
    settings.semanticSearch.semanticWeight,
    settings.semanticSearch.lexicalWeight
  );
}, [settings.semanticSearch, app]);

// Add search hook
const {
  results: searchResults,
  isSearching,
  isReady: isSearchReady,
  search,
  indexVault,
  incrementalIndex
} = useSemanticSearch(searchAdapter);

// Use in message preparation
const handleSendMessage = useCallback(async (content: string) => {
  // Check if agent requested context
  if (shouldProvideContext(content)) {
    // Extract query from message
    const query = extractSearchQuery(content);
    
    // Search for relevant context
    const contextResults = await search(query, {
      limit: 5,
      mode: 'hybrid'
    });
    
    // Add context to message
    const contextText = formatSearchResults(contextResults);
    const enhancedContent = `${content}\n\n## Relevant Context\n\n${contextText}`;
    
    // Send enhanced message
    await sendMessage(enhancedContent);
  } else {
    await sendMessage(content);
  }
}, [search, sendMessage]);
```

---

## Implementation Roadmap

### Phase 1: Foundation (Week 1-2)

**Goals:**
- Set up package dependencies
- Implement core chunking logic
- Create domain models and ports

**Tasks:**
1. Install packages: `@orama/orama`, `flexsearch`, `ollama`, `crypto-js`, `async-mutex`
2. Create `src/shared/chunk-manager.ts` with heading-first chunking
3. Create domain models:
   - `src/domain/models/search-result.ts`
   - `src/domain/models/search-settings.ts`
4. Create port interface:
   - `src/domain/ports/semantic-search.port.ts`
5. Add unit tests for chunking logic

**Deliverables:**
- Chunking works correctly for various note structures
- Domain layer complete with types and interfaces
- ~500 lines of code

### Phase 2: Local Embeddings (Week 3)

**Goals:**
- Integrate Ollama for embeddings
- Test embedding generation
- Implement settings UI

**Tasks:**
1. Create `src/adapters/search/embedding.adapter.ts`
2. Implement Ollama integration:
   - Connection check
   - Model validation
   - Batch embedding generation
3. Add fallback to LM Studio
4. Update settings UI:
   - Enable/disable semantic search
   - Choose embedding provider
   - Configure Ollama/LM Studio settings
5. Add status indicator in UI

**Deliverables:**
- Embedding generation works locally
- Settings UI complete
- Status indicator shows readiness
- ~300 lines of code

### Phase 3: Vector Search (Week 4)

**Goals:**
- Implement Orama vector database
- Create indexing pipeline
- Test vector search

**Tasks:**
1. Create `src/adapters/search/vector-store.adapter.ts`
2. Implement Orama integration:
   - Schema definition
   - Document insertion
   - Vector search
3. Create `src/shared/index-manager.ts`:
   - Full vault indexing
   - Incremental indexing
   - Garbage collection
4. Add persistence layer (JSONL + binary)
5. Test with real vault data

**Deliverables:**
- Vector search returns relevant results
- Indexing handles large vaults (1000+ notes)
- Persistence works correctly
- ~600 lines of code

### Phase 4: Lexical Search (Week 5)

**Goals:**
- Implement FlexSearch engine
- Add multilingual support
- Test keyword search

**Tasks:**
1. Create `src/adapters/search/lexical.adapter.ts`
2. Implement FlexSearch integration:
   - Index configuration
   - Document indexing
   - Search with field weighting
3. Add multilingual tokenization (CJK support)
4. Implement graph-based boosting
5. Test with various queries

**Deliverables:**
- Lexical search works for English and CJK
- Graph boosting improves results
- ~400 lines of code

### Phase 5: Hybrid Search (Week 6)

**Goals:**
- Merge semantic and lexical results
- Tune scoring weights
- Optimize performance

**Tasks:**
1. Create `src/adapters/search/hybrid-search.adapter.ts`
2. Implement result merging:
   - Weighted score combination
   - Deduplication
   - Result ranking
3. Add `src/hooks/useSemanticSearch.ts`
4. Integrate into ChatView
5. Benchmark performance
6. Tune scoring weights

**Deliverables:**
- Hybrid search provides best results
- Performance is acceptable (< 500ms)
- ~300 lines of code

### Phase 6: Agent Integration (Week 7)

**Goals:**
- Automatically provide context to agents
- Add slash commands for search
- Implement context formatting

**Tasks:**
1. Add `/search` slash command
2. Implement automatic context provision:
   - Detect when agent needs context
   - Search relevant notes
   - Format results for agent
3. Add UI for search results
4. Update message preparation logic
5. Add examples to documentation

**Deliverables:**
- Agents receive relevant context automatically
- Users can manually trigger search
- Documentation complete
- ~400 lines of code

### Phase 7: Polish & Optimization (Week 8)

**Goals:**
- Optimize performance
- Add advanced features
- Complete documentation

**Tasks:**
1. Performance optimization:
   - Lazy loading
   - Index compression
   - Query caching
2. Advanced features:
   - Tag filtering
   - Path filtering
   - Date range filtering
3. Settings refinement:
   - Preset configurations
   - Advanced options
4. Documentation:
   - User guide
   - Developer guide
   - Troubleshooting

**Deliverables:**
- Performance < 200ms for most queries
- All features documented
- Ready for release

---

## Performance Considerations

### Memory Usage

**Estimated Memory:**
- Embeddings: `numChunks * vectorDim * 4 bytes`
  - Example: 5000 chunks × 768 dims × 4 bytes = ~15MB
- FlexSearch index: `~5-10MB` per 1000 notes
- Chunk cache: `~10-50MB` (configurable)

**Total**: ~30-75MB for typical vault (1000 notes)

**Optimization Strategies:**
1. **Partitioned Storage**: Split index across multiple files
2. **Lazy Loading**: Load partitions on-demand
3. **Binary Format**: Store vectors in Float32Array
4. **Chunk Cache**: LRU cache with configurable size
5. **Compression**: GZIP partition files

### Indexing Speed

**Benchmarks** (estimated):
- Chunking: ~1000 notes/second
- Embedding generation: ~10-50 notes/second (depends on model)
- Vector insertion: ~500 notes/second
- Lexical indexing: ~2000 notes/second

**Total Indexing Time**:
- 100 notes: ~5-10 seconds
- 1000 notes: ~30-60 seconds
- 5000 notes: ~3-5 minutes

**Optimization Strategies:**
1. **Incremental Indexing**: Only reindex changed files
2. **Batch Processing**: Embed multiple documents at once
3. **Background Indexing**: Index during idle time
4. **Skip Large Files**: Warn user about files > 100KB
5. **Progress Indicator**: Show indexing progress

### Search Speed

**Target Performance:**
- Vector search: < 100ms
- Lexical search: < 50ms
- Hybrid search: < 200ms

**Optimization Strategies:**
1. **HNSW Algorithm**: Approximate nearest neighbor (Orama default)
2. **Index Warmup**: Keep index in memory
3. **Query Caching**: Cache recent queries (5-minute TTL)
4. **Result Limiting**: Cap at 50 results
5. **Lazy Content Loading**: Load chunk content on-demand

### Disk Usage

**Estimated Storage:**
- Chunks (JSONL): `~1KB per chunk`
  - Example: 5000 chunks = ~5MB
- Embeddings (binary): `~3KB per chunk`
  - Example: 5000 chunks × 768 dims × 4 bytes = ~15MB
- FlexSearch index: `~2-5MB per 1000 notes`

**Total**: ~25-50MB for typical vault (1000 notes)

**Optimization Strategies:**
1. **Compression**: GZIP all storage files
2. **Incremental Updates**: Only rewrite changed partitions
3. **Garbage Collection**: Remove stale data
4. **Configurable Retention**: Option to clear old embeddings

---

## Conclusion

This semantic search implementation provides a powerful, **100% local** solution for context-aware AI agent conversations in Obsidian. By combining:

1. **Vector search** (semantic understanding via Ollama embeddings)
2. **Lexical search** (keyword matching via FlexSearch)
3. **Graph boosting** (leveraging Obsidian's link structure)
4. **Intelligent chunking** (heading-first strategy)

The system can automatically provide relevant context to AI agents, significantly improving the quality of responses and reducing the need for manual note mentions.

**Key Benefits:**
- ✅ **Privacy**: All processing happens locally (no external APIs)
- ✅ **Speed**: Optimized for quick search (< 200ms)
- ✅ **Memory Efficient**: Chunked storage with configurable limits
- ✅ **Architectural Fit**: Follows React Hooks Architecture pattern
- ✅ **Incremental Updates**: Only reindex changed notes
- ✅ **Obsidian-Native**: Uses Obsidian's metadata cache and link graph

**Estimated Effort:**
- Total development time: **6-8 weeks** (one developer)
- Total lines of code: **~2,500 lines**
- Additional dependencies: **~500KB** bundle size

**Next Steps:**
1. Review this recommendation with the team
2. Prioritize features (semantic-only vs. hybrid)
3. Set up development environment with Ollama
4. Begin Phase 1 implementation (foundation)

---

## References

- **Obsidian Copilot Implementation**: https://github.com/logancyang/obsidian-copilot
- **Ollama Documentation**: https://ollama.com/
- **Orama Vector Database**: https://docs.orama.com/
- **FlexSearch**: https://github.com/nextapps-de/flexsearch
- **LangChain Text Splitters**: https://js.langchain.com/docs/modules/data_connection/document_transformers/
- **Agent Client Protocol**: https://github.com/zed-industries/agent-client-protocol

---

## Appendix A: Example Search Queries

### Query 1: Technical Concept

**User Query**: "How do I implement authentication in my project?"

**System Actions**:
1. Generate embedding for query
2. Search vector store (semantic)
3. Search FlexSearch (lexical: "authentication", "implement", "project")
4. Merge results (hybrid)
5. Apply graph boosting (notes linked to "authentication" patterns)

**Expected Results**:
- Notes about authentication implementation
- Security best practices
- Code examples from past projects
- Related notes about auth libraries

### Query 2: Multilingual

**User Query**: "機械学習アルゴリズム" (Machine learning algorithms in Japanese)

**System Actions**:
1. Generate embedding (works with any language)
2. Tokenize as CJK bigrams: ["機械", "械学", "学習", "習ア", ...]
3. Search both semantic and lexical
4. Return relevant notes regardless of language

**Expected Results**:
- Notes about machine learning (English or Japanese)
- Algorithm implementations
- Related research papers

### Query 3: Contextual

**User Query**: "What did I write about React hooks last month?"

**System Actions**:
1. Parse temporal constraint ("last month")
2. Search for "React hooks"
3. Filter by date (mtime)
4. Sort by relevance + recency

**Expected Results**:
- Recent notes about React hooks
- Meeting notes from last month
- Code snippets and examples

---

## Appendix B: Settings Configuration Example

```typescript
// Default settings
const DEFAULT_SEMANTIC_SEARCH_SETTINGS: SemanticSearchSettings = {
  enabled: false,                              // Opt-in
  embeddingProvider: 'ollama',                 // Default to Ollama
  ollamaHost: 'http://localhost:11434',
  ollamaModel: 'nomic-embed-text',            // Best balance
  lmstudioHost: 'http://localhost:1234/v1',
  lmstudioModel: 'text-embedding-ada-002',
  chunkSize: 6000,                            // 6KB chunks
  maxResults: 10,                             // Top 10 results
  minSimilarity: 0.1,                         // Filter threshold
  useHybridSearch: true,                      // Semantic + lexical
  semanticWeight: 0.6,                        // 60% semantic
  lexicalWeight: 0.4,                         // 40% lexical
  autoIndex: true,                            // Index on startup
  indexInterval: 300000,                      // Reindex every 5 minutes
  excludePatterns: [                          // Exclude paths
    '.obsidian/',
    '.trash/',
    'node_modules/'
  ],
  maxIndexSize: 100 * 1024 * 1024,           // 100MB limit
  enableGraphBoost: true,                     // Use link graph
  graphBoostWeight: 0.2                       // 20% boost for linked notes
};
```

---

*Document Version: 1.0*  
*Created: 2026-01-07*  
*Author: AI Architecture Team*
