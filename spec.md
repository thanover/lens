# LENS Specification

**Lightweight Encoded Navigation Schema**

*Version 0.1.0 (Draft)*

---

## 1. Overview

### 1.1 Purpose

LENS is a specification for representing knowledge in a format optimized for AI agent consumption. It enables agents to efficiently navigate and understand codebases, documentation, and other knowledge artifacts without loading and analyzing raw content from scratch.

### 1.2 Design Principles

| Principle | Rationale |
|-----------|-----------|
| **Pre-computed** | Work is done at build time, not query time |
| **Incremental** | Changes update only affected portions |
| **Layered** | Agents can load minimal context first, drill deeper as needed |
| **Format-agnostic** | Works for code, prose, or mixed content |
| **Human-readable** | Developers can inspect and debug the output |
| **Tool-independent** | Any agent or tool can consume the spec |

### 1.3 Core Concepts

| Term | Definition | Examples |
|------|------------|----------|
| **Ecosystem** | A collection of related Knowledge Sources | All repos and docs for a platform |
| **Knowledge Source (KS)** | A distinct container of knowledge with a defined boundary | A git repo, a documentation site, a folder of markdown files |
| **Document** | A single artifact within a KS (implicit, derived from node paths) | A source file, a markdown doc, a config file |
| **Node** | A discrete, addressable unit within a Document | A function, a class, a paragraph, a section, a type definition |
| **Edge** | A relationship between two Nodes | calls, imports, references, depends_on |
| **Layer** | A level of detail (L0 = high-level overview, L1 = module-level, L2 = implementation) | L0 summary describes entire KS, L2 describes a single function |

### 1.4 Conceptual Hierarchy

```
Ecosystem
└── Knowledge Source (KS)
    └── Document (implicit, derived from node paths)
        └── Node
            └── Edge (connects nodes)
```

Documents are not explicit entities in the graph. They are derived from node `source_path` values. This allows LENS to organize nodes semantically rather than mirroring potentially arbitrary file structures.

---

## 2. Directory Structure

```
.lens/
├── manifest.json           # Entry point, metadata, schema version
├── graph.json              # Nodes and edges
├── hashes.json             # File hashes for incremental updates
├── summaries/
│   ├── L0.md               # Ecosystem/KS-level summary
│   ├── L1/                 # Module/section-level summaries
│   │   ├── auth.md
│   │   └── api.md
│   └── L2/                 # Detailed summaries (optional)
│       └── auth/
│           └── validateToken.md
├── entities.json           # Named entities index (people, systems, concepts)
├── claims.json             # Assertions and evidence (for prose content)
├── protocol.md             # Human-readable consumption guide
├── protocol.json           # Machine-readable agent instructions
└── embeddings/             # Optional: vector embeddings for semantic search
    └── nodes.bin
```

---

## 3. Manifest Schema

`manifest.json` is the entry point. An agent reads this first.

```json
{
  "lens_version": "0.1.0",
  "generated_at": "2025-02-01T14:30:00Z",
  "generator": {
    "tool": "lens-cli",
    "version": "0.1.0"
  },
  "source": {
    "type": "repository",
    "name": "my-service",
    "root": "/",
    "languages": ["typescript"],
    "content_types": ["code", "documentation"]
  },
  "freshness": {
    "generated_at": "2025-02-01T14:30:00Z",
    "source_ref": {
      "type": "git_commit",
      "value": "abc123def456"
    },
    "expires_hint": "P7D"
  },
  "coverage": {
    "files_analyzed": 185,
    "files_skipped": 15,
    "skip_reasons": {
      "excluded_pattern": 10,
      "parse_error": 3,
      "unsupported_type": 2
    },
    "skipped_paths": ["generated/", "dist/", "src/legacy/broken.js"]
  },
  "build": {
    "mode": "incremental",
    "last_full_build": "2025-01-01T00:00:00Z",
    "last_incremental": "2025-02-01T14:30:00Z",
    "files_changed_since_full": 12
  },
  "stats": {
    "total_nodes": 142,
    "total_edges": 287,
    "layers": ["L0", "L1", "L2"]
  },
  "entry_points": {
    "graph": "./graph.json",
    "hashes": "./hashes.json",
    "summaries": "./summaries/",
    "entities": "./entities.json",
    "embeddings": "./embeddings/",
    "protocol": "./protocol.json"
  },
  "related_lens": [
    {
      "name": "shared-utils",
      "location": "https://github.com/org/shared-utils/.lens/"
    }
  ],
  "embeddings": {
    "format": "sqlite",
    "path": "./embeddings/vectors.db",
    "model": "text-embedding-3-small",
    "dimensions": 1536
  }
}
```

### 3.1 Field Definitions

| Field | Required | Description |
|-------|----------|-------------|
| `lens_version` | Yes | Spec version for compatibility |
| `generated_at` | Yes | ISO 8601 timestamp of last build |
| `generator` | Yes | Tool and version that produced this LENS |
| `source.type` | Yes | `repository`, `documentation`, `mixed` |
| `source.content_types` | Yes | Array: `code`, `documentation`, `data` |
| `freshness` | Yes | Information about build freshness and validity |
| `coverage` | Yes | What was and wasn't indexed |
| `build` | Yes | Build mode and history |
| `stats` | Yes | Quick overview for agents to gauge scope |
| `entry_points` | Yes | Paths to other LENS files |
| `related_lens` | No | Cross-references to other LENS-enabled sources |
| `embeddings` | No | Embedding storage configuration |

---

## 4. Graph Schema

`graph.json` contains the knowledge graph: nodes and edges.

```json
{
  "nodes": [
    {
      "id": "fn:auth:validateToken",
      "type": "function",
      "name": "validateToken",
      "layer": "L2",
      "source_path": "src/auth/validate.ts",
      "line_start": 24,
      "line_end": 58,
      "content_hash": "sha256:abc123...",
      "signature_hash": "sha256:def456...",
      "summary_input_hash": "sha256:ghi789...",
      "summary_ref": "./summaries/L2/auth/validateToken.md",
      "build_dependencies": {
        "files": ["src/crypto/verify.ts"],
        "nodes": ["fn:crypto:verifySignature"]
      },
      "metadata": {
        "visibility": "exported",
        "async": true,
        "parameters": ["token: string", "options?: ValidateOptions"],
        "return_type": "Promise<TokenPayload | null>"
      }
    },
    {
      "id": "mod:auth",
      "type": "module",
      "name": "auth",
      "layer": "L1",
      "source_path": "src/auth/",
      "summary_ref": "./summaries/L1/auth.md",
      "children": ["fn:auth:validateToken", "fn:auth:refreshToken", "fn:auth:revokeToken"]
    },
    {
      "id": "doc:adr:003",
      "type": "decision",
      "name": "ADR-003: JWT vs Session Tokens",
      "layer": "L1",
      "source_path": "docs/decisions/003-jwt-vs-sessions.md",
      "summary_ref": "./summaries/L1/decisions/adr-003.md"
    }
  ],
  "edges": [
    {
      "from": "fn:auth:validateToken",
      "to": "fn:crypto:verifySignature",
      "type": "calls"
    },
    {
      "from": "mod:auth",
      "to": "mod:crypto",
      "type": "depends_on"
    },
    {
      "from": "fn:auth:validateToken",
      "to": "doc:adr:003",
      "type": "implements_decision"
    }
  ]
}
```

### 4.1 Node Types

| Type | Description | Typical Layer |
|------|-------------|---------------|
| `knowledge_source` | Top-level container (the KS itself) | L0 |
| `module` | Directory, package, or logical grouping | L1 |
| `function` | Function or method | L2 |
| `class` | Class definition | L2 |
| `type` | Type/interface definition | L2 |
| `document` | Prose document | L0-L1 |
| `section` | Section within a document | L1-L2 |
| `decision` | ADR or design decision | L1 |
| `concept` | Abstract concept (for prose) | L1 |
| `entity` | Named entity (person, system, org) | L1 |

### 4.2 Edge Types

| Type | Description |
|------|-------------|
| `calls` | Function invokes another function |
| `imports` | Module imports another module |
| `depends_on` | General dependency relationship |
| `extends` | Class inheritance |
| `implements` | Implements an interface |
| `references` | Document references another document |
| `implements_decision` | Code implements a design decision |
| `defines` | Document defines a concept |
| `related_to` | General semantic relationship |

### 4.3 Layers

| Layer | Scope | Use Case |
|-------|-------|----------|
| **L0** | Entire KS | "What is this project?" |
| **L1** | Modules, major sections | "What are the main components?" |
| **L2** | Functions, detailed sections | "How does this specific thing work?" |

Agents load L0 first, then selectively load L1/L2 as needed.

### 4.4 Node Field Definitions

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Unique identifier for the node |
| `type` | Yes | Node type (see 4.1) |
| `name` | Yes | Human-readable name |
| `layer` | Yes | L0, L1, or L2 |
| `source_path` | Yes | Path to the source file or directory |
| `line_start` | No | Starting line number (for code nodes) |
| `line_end` | No | Ending line number (for code nodes) |
| `content_hash` | No | Hash of the source content |
| `signature_hash` | No | Hash of the interface/signature (for incremental builds) |
| `summary_input_hash` | No | Hash of input sent to LLM for summary generation |
| `summary_ref` | No | Path to the summary file |
| `build_dependencies` | No | Files and nodes this node depends on |
| `children` | No | Child node IDs (for container nodes) |
| `metadata` | No | Type-specific additional information |

---

## 5. Hashes Schema

`hashes.json` stores file hashes for incremental updates.

```json
{
  "schema_version": "0.1.0",
  "algorithm": "sha256",
  "generated_at": "2025-02-01T14:30:00Z",
  "files": {
    "src/auth/validate.ts": {
      "content_hash": "sha256:a1b2c3d4...",
      "size": 2847,
      "produces_nodes": ["fn:auth:validateToken", "fn:auth:checkExpiry"]
    },
    "src/auth/refresh.ts": {
      "content_hash": "sha256:e5f6g7h8...",
      "size": 1923,
      "produces_nodes": ["fn:auth:refreshToken"]
    },
    "docs/architecture.md": {
      "content_hash": "sha256:i9j0k1l2...",
      "size": 8472,
      "produces_nodes": ["doc:architecture", "section:architecture:overview"]
    }
  }
}
```

### 5.1 Incremental Update Algorithm

```
function incrementalBuild():
    
    currentHashes = hashAllSourceFiles()
    previousHashes = loadHashesJson()
    
    changedFiles = []
    for file in currentHashes:
        if file not in previousHashes:
            changedFiles.add(file)  // New file
        elif currentHashes[file] != previousHashes[file]:
            changedFiles.add(file)  // Modified file
    
    deletedFiles = previousHashes.keys() - currentHashes.keys()
    
    // Phase 1: Determine directly affected nodes
    directlyAffected = []
    for file in changedFiles:
        directlyAffected.addAll(previousHashes[file].produces_nodes)
    
    for file in deletedFiles:
        directlyAffected.addAll(previousHashes[file].produces_nodes)
        markForDeletion(previousHashes[file].produces_nodes)
    
    // Phase 2: Determine indirectly affected nodes
    indirectlyAffected = []
    for node in directlyAffected:
        dependents = findNodesThatDependOn(node)
        indirectlyAffected.addAll(dependents)
    
    // Phase 3: Regenerate
    allAffected = directlyAffected + indirectlyAffected
    
    for node in allAffected:
        if node.markedForDeletion:
            removeFromGraph(node)
        else:
            regenerateNode(node)      // Re-parse, extract info
            regenerateSummary(node)   // Re-run LLM summarization
            updateEdges(node)         // Recompute relationships
    
    // Phase 4: Update hashes
    saveHashesJson(currentHashes)
    updateManifest()
```

### 5.2 Signature-Aware Optimization

To minimize unnecessary propagation, track interface vs implementation changes:

- If only `content_hash` changed (implementation detail): Regenerate this node's summary, don't propagate to dependents
- If `signature_hash` changed (interface changed): Regenerate this node and propagate to all dependents

### 5.3 When to Force Full Rebuild

- Schema version changes
- Build tool version changes
- Too many incremental builds (e.g., every 50 builds)
- User explicitly requests it
- Hashes file is missing or corrupted

---

## 6. Summaries

Summaries are natural language descriptions stored as Markdown. They exist at each layer.

### 6.1 L0 Summary (KS-level)

`summaries/L0.md`

```markdown
# my-service

A TypeScript microservice handling user authentication and session management for the platform.

## Primary Responsibilities
- JWT token validation and refresh
- Session lifecycle management
- Integration with the central identity provider

## Key Dependencies
- `shared-utils` for cryptographic operations
- `platform-events` for publishing auth events

## Entry Points
- `src/index.ts` — Service initialization
- `src/handlers/` — HTTP request handlers
```

### 6.2 L1 Summary (Module-level)

`summaries/L1/auth.md`

```markdown
# auth module

Handles all token operations including validation, refresh, and revocation.

## Key Functions
- `validateToken` — Verifies JWT signature and expiration
- `refreshToken` — Issues new token from valid refresh token
- `revokeToken` — Invalidates a token family

## Design Notes
- Implements decision ADR-003 (JWT over sessions)
- Tokens use RS256 signing (keys in Vault)
```

### 6.3 L2 Summary (Function-level)

`summaries/L2/auth/validateToken.md`

```markdown
# validateToken

Validates a JWT access token and returns the decoded payload if valid.

## Behavior
- Checks signature using public key from key cache
- Validates `exp`, `iat`, and `iss` claims
- Returns `null` for invalid/expired tokens (does not throw)

## Usage Context
Called by auth middleware on every protected route.
```

---

## 7. Entities Index

`entities.json` captures named entities for quick lookup.

```json
{
  "entities": [
    {
      "id": "entity:vault",
      "name": "HashiCorp Vault",
      "type": "system",
      "mentions": [
        {"node": "fn:auth:validateToken", "context": "retrieves signing keys"},
        {"node": "doc:adr:003", "context": "chosen for secrets management"}
      ]
    },
    {
      "id": "entity:identity-provider",
      "name": "Central Identity Provider",
      "type": "system",
      "aliases": ["IdP", "CIP"],
      "mentions": [
        {"node": "mod:auth", "context": "integration point"}
      ]
    }
  ]
}
```

---

## 8. Claims Index

`claims.json` captures assertions and their supporting evidence (primarily for prose content).

```json
{
  "claims": [
    {
      "id": "claim:001",
      "statement": "JWT tokens are preferred over server-side sessions for this service",
      "source_node": "doc:adr:003",
      "evidence": [
        "Stateless validation reduces database load",
        "Aligns with platform-wide token strategy"
      ],
      "related_claims": ["claim:002"]
    }
  ]
}
```

---

## 9. Embeddings

Embeddings are optional vector representations for semantic search.

### 9.1 Purpose

Embeddings enable semantic search—finding nodes by meaning rather than exact keyword match. An agent can ask "how does login work?" and find relevant nodes even if they don't contain the word "login."

### 9.2 Storage Formats

| Format | Description | Use Case |
|--------|-------------|----------|
| `binary` | Raw binary file with vectors | Self-contained, offline |
| `sqlite` | SQLite database with vector extension | Queryable, self-contained |
| `external` | Reference to external vector DB | Large scale, cloud-native |
| `none` | No embeddings | Simpler, keyword-only search |

### 9.3 Configuration in Manifest

```json
{
  "embeddings": {
    "format": "sqlite",
    "path": "./embeddings/vectors.db",
    "model": "text-embedding-3-small",
    "dimensions": 1536
  }
}
```

### 9.4 Relationship to Other Components

Embeddings are one tool in the toolkit, not a replacement for graph or summaries:

| Component | What it answers |
|-----------|-----------------|
| **Graph** | "What is structurally related to X?" (explicit relationships) |
| **Summaries** | "What does X do?" (natural language descriptions) |
| **Embeddings** | "What is semantically similar to this query?" (fuzzy matching) |

---

## 10. Agent Consumption Protocol

### 10.1 Discovery

```
1. Check for .lens/manifest.json in KS root
2. If not found, fall back to raw content analysis
3. If found, load manifest to understand scope and available components
```

### 10.2 Consumption Workflow

```
┌─────────────────────────────────────────────────────────────┐
│                     Agent receives task                     │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  1. ORIENT                                                  │
│     Load L0 summary                                         │
│     → Understand overall scope of the KS                    │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  2. DISCOVER                                                │
│     Use embeddings (if present) for semantic search         │
│     → Returns ranked list of candidate node IDs             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  3. TRAVERSE                                                │
│     Follow graph edges from candidate nodes                 │
│     → Understand relationships and expand context           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  4. SUMMARIZE                                               │
│     Load L1 summaries for relevant modules                  │
│     Load L2 summaries if more detail needed                 │
│     → Build understanding without reading raw source        │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│  5. SOURCE (only if necessary)                              │
│     Access raw source files for implementation details      │
│     → Use source_path and line numbers from nodes           │
└─────────────────────────────────────────────────────────────┘
```

### 10.3 Efficiency Rules

- Never load raw source files if a summary answers the question
- Prefer graph traversal over scanning all nodes
- Use embeddings when terminology might not match exactly
- Load L2 only when L1 is insufficient
- Check `coverage.skipped_paths` before assuming LENS has everything

### 10.4 Handling Freshness

- Compare `freshness.source_ref` to current state if possible
- Treat LENS older than `expires_hint` with lower confidence
- For critical modifications, verify against raw source

### 10.5 Recommended Tool Definitions

Agents implementing LENS support SHOULD expose these tools:

**lens_discover**
```json
{
  "name": "lens_discover",
  "description": "Check if a knowledge source has LENS indexing available.",
  "parameters": {
    "path": "Root path of the knowledge source"
  }
}
```

**lens_search**
```json
{
  "name": "lens_search",
  "description": "Semantic search across a LENS-indexed knowledge source.",
  "parameters": {
    "query": "Natural language description of what you're looking for",
    "limit": "Maximum results to return"
  }
}
```

**lens_get_summary**
```json
{
  "name": "lens_get_summary",
  "description": "Get the summary for a specific node.",
  "parameters": {
    "node_id": "The node ID from graph or search results",
    "layer": "L0, L1, or L2"
  }
}
```

**lens_traverse**
```json
{
  "name": "lens_traverse",
  "description": "Get nodes related to a given node via graph edges.",
  "parameters": {
    "node_id": "Starting node",
    "edge_types": "Filter by relationship type",
    "direction": "outgoing, incoming, or both"
  }
}
```

**lens_get_source**
```json
{
  "name": "lens_get_source",
  "description": "Get raw source content for a node. Use only when summaries are insufficient.",
  "parameters": {
    "node_id": "The node ID"
  }
}
```

### 10.6 Protocol Files

Every LENS instance SHOULD include:

- `protocol.md` — Human-readable consumption guide
- `protocol.json` — Machine-readable agent instructions (optional)

---

## 11. LENS vs RAG

LENS and RAG (Retrieval-Augmented Generation) are complementary approaches.

### 11.1 Comparison

| Aspect | RAG | LENS |
|--------|-----|------|
| **Primary mechanism** | Vector similarity search | Multi-modal: graph + summaries + embeddings |
| **Knowledge representation** | Flat chunks in vector DB | Hierarchical graph with typed nodes and edges |
| **Relationships** | Implicit (similar vectors might be related) | Explicit (A calls B, X depends on Y) |
| **Context assembly** | Top-k similar chunks, concatenated | Progressive loading based on task needs |
| **Granularity control** | Chunk size fixed at index time | Layers (L0/L1/L2) allow dynamic depth |
| **Understanding structure** | Lost during chunking | Preserved in graph |

### 11.2 How They Relate

LENS is not "better RAG"—it's a different layer:

- RAG is about **retrieval**
- LENS is about **representation**

Embeddings (the RAG mechanism) are one component within LENS. LENS adds structure, relationships, and summaries on top.

```
┌─────────────────────────────────────────────┐
│                   LENS                      │
│  ┌───────────────────────────────────────┐  │
│  │  Graph (structure & relationships)    │  │
│  ├───────────────────────────────────────┤  │
│  │  Summaries (pre-computed descriptions)│  │
│  ├───────────────────────────────────────┤  │
│  │  Embeddings (semantic search) ← RAG   │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### 11.3 When to Use Which

| Situation | Recommendation |
|-----------|----------------|
| Chat over a pile of PDFs | RAG is probably sufficient |
| AI agent working in a codebase | LENS adds significant value |
| Documentation with clear structure | LENS preserves and exposes that structure |
| One-off question answering | RAG is simpler |
| Repeated access by multiple agents | LENS amortizes the understanding cost |
| Cross-KS navigation | LENS graph relationships shine |

---

## 12. Generation Requirements

Tools that generate LENS must:

| Requirement | Rationale |
|-------------|-----------|
| Parse AST for code (not regex) | Accurate structure extraction |
| Extract prose structure semantically | Headers, sections, claims |
| Generate summaries via LLM or templates | Human-readable descriptions |
| Compute relationships statically | Calls, imports, references |
| Support incremental updates | Only regenerate changed nodes |
| Validate output against schema | Ensure spec compliance |
| Track file hashes | Enable efficient incremental builds |

---

## 13. Cross-KS Relationships

LENS supports linking nodes across Knowledge Sources within an Ecosystem.

### 13.1 Ecosystem Links in Manifest

```json
{
  "related_lens": [
    {
      "name": "shared-utils",
      "location": "https://github.com/org/shared-utils/.lens/"
    },
    {
      "name": "platform-docs",
      "location": "../platform-docs/.lens/"
    }
  ]
}
```

### 13.2 Cross-KS Edge References

```json
{
  "edges": [
    {
      "from": "fn:auth:validateToken",
      "to": "shared-utils:fn:crypto:verifySignature",
      "type": "calls"
    }
  ]
}
```

---

## 14. Open Questions

### Spec Design

- **Schema formalization** — JSON Schema for validation? How strict?
- **Minimum viable LENS** — What's required vs optional?
- **Embedding format default** — Should there be a "standard" portable format?

### Cross-KS

- **Version mismatches** — KS A references KS B, but B updated—how to handle?
- **Central registry** — Should there be a way to discover LENS-enabled sources?

### Generation

- **Tooling scope** — Build a reference `lens` CLI? Or just spec the format?
- **LLM dependency** — How to handle cost/consistency for summary generation?
- **Language support** — Start with TypeScript? How to extend?

### Adoption

- **Standardization path** — Keep proprietary? Open spec? Community input?
- **Competing approaches** — Interoperability with similar efforts?

---

## Appendix A: Example Directory Structure

```
my-service/
├── .lens/
│   ├── manifest.json
│   ├── graph.json
│   ├── hashes.json
│   ├── entities.json
│   ├── claims.json
│   ├── protocol.md
│   ├── protocol.json
│   ├── summaries/
│   │   ├── L0.md
│   │   ├── L1/
│   │   │   ├── auth.md
│   │   │   ├── api.md
│   │   │   └── decisions/
│   │   │       └── adr-003.md
│   │   └── L2/
│   │       └── auth/
│   │           ├── validateToken.md
│   │           └── refreshToken.md
│   └── embeddings/
│       └── vectors.db
├── src/
│   ├── auth/
│   │   ├── validate.ts
│   │   └── refresh.ts
│   └── api/
│       └── handlers.ts
└── docs/
    └── decisions/
        └── 003-jwt-vs-sessions.md
```

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **AST** | Abstract Syntax Tree — a tree representation of source code structure |
| **Ecosystem** | A collection of related Knowledge Sources |
| **Edge** | A relationship between two nodes |
| **Embedding** | A vector representation of text for semantic similarity |
| **Knowledge Source (KS)** | A distinct container of knowledge (repo, docs, etc.) |
| **Layer** | A level of detail (L0, L1, L2) |
| **LENS** | Lightweight Encoded Navigation Schema |
| **Node** | A discrete unit of knowledge in the graph |
| **RAG** | Retrieval-Augmented Generation |

---

*End of Specification*
