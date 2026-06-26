
# MCP Server Implementation Guide

A comprehensive guide for AI agents on how to implement a production-quality MCP (Model Context Protocol) server, based on the excellent patterns used in the team-docs-mcp project.

## Overview

This guide demonstrates how to build a well-architected MCP server with TypeScript, focusing on clean architecture, type safety, modularity, and production-ready practices. The team-docs-mcp server serves as the reference implementation, showcasing semantic search over team documentation with local embeddings.

## Core Architecture Principles

### 1. Separation of Concerns

The implementation separates distinct responsibilities into focused modules:

- **Entry Point (`server.ts`)** - Wiring, startup sequence, and MCP server initialization
- **Configuration (`config.ts`)** - Centralized environment variable management with defaults
- **Tools (`tools/*.ts`)** - Individual tool implementations extending a base class
- **Core Logic (`file-indexer.ts`, `document-loader.ts`)** - Business logic separate from MCP protocol
- **Utilities (`logger.ts`, `tool.ts`)** - Reusable infrastructure components

**Key Insight**: Business logic (indexing, search, file operations) is independent of the MCP protocol. This makes the code testable, maintainable, and reusable.

### 2. Type Safety First

- **Strict TypeScript configuration** with `strict: true`
- **Zod schemas** for runtime validation of tool inputs
- **Explicit type definitions** for all data structures
- **No `any` types** except where absolutely necessary for SDK bridge code

### 3. Modular Tool Architecture

Each tool is a self-contained class that:
- Extends the abstract `Tool<TInput>` base class
- Defines its schema, description, and metadata
- Implements the `execute()` method
- Registers itself on the MCP server

## Project Structure

```
server.ts                              # Entry point - wiring & startup
src/
  config.ts                            # Centralized configuration
  tool.ts                              # Abstract Tool<TInput> base class
  logger.ts                            # Logging utility (stderr for MCP)
  file-indexer.ts                      # Core business logic: chunking, embedding, search
  document-loader.ts                   # File discovery and reading
  tools/
    search-documentation.ts            # Tool: semantic search
    list-indexed-files.ts              # Tool: list indexed files
    get-document-content.ts            # Tool: read file content
    get-documentation-guidelines.ts    # Tool: get documentation guidelines
    save-documentation.ts              # Tool: save and index new docs
assets/
  documentation-guidelines.md          # Static resources
docs/                                  # User-provided documentation
  .cache/                              # Generated embeddings cache
package.json                           # Dependencies and scripts
tsconfig.json                          # TypeScript configuration
```

**Key Insight**: A shallow, organized structure makes navigation easy. Related files are grouped logically without excessive nesting.

## Step-by-Step Implementation

### Step 1: Project Setup

#### package.json

```json
{
  "name": "your-mcp-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "start": "npx tsx server.ts",
    "inspect": "npx @modelcontextprotocol/inspector npx tsx server.ts"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.27.1",
    "chalk": "^5.6.2",
    "ts-node": "^10.9.2",
    "typescript": "^6.0.2",
    "zod": "^3.25.76"
  },
  "devDependencies": {
    "@types/node": "^25.5.0",
    "tsx": "^4.21.0"
  }
}
```

**Critical Points**:
- **`"type": "module"`** - Use ES modules (required for modern MCP SDK)
- **`tsx`** - Fast TypeScript execution without compilation
- **`chalk`** - Terminal colors for logging
- **`zod`** - Runtime schema validation

#### tsconfig.json

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "target": "esnext",
    "types": ["node"],
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "skipLibCheck": true
  }
}
```

**Critical Points**:
- **`module: "nodenext"`** - Proper ES module resolution for Node.js
- **`strict: true`** - Enable all strict type checking
- **`noUncheckedIndexedAccess: true`** - Prevents undefined access bugs
- **`verbatimModuleSyntax: true`** - Explicit import/export semantics

### Step 2: Configuration Management

Create `src/config.ts` for centralized configuration:

```typescript
import path from 'node:path';

// ── Helpers ──────────────────────────────────────────────────────────

function envInt(key: string, fallback: number): number {
    const raw = process.env[key];
    if (raw === undefined || raw === '') return fallback;
    const parsed = Number(raw);
    return Number.isFinite(parsed) ? parsed : fallback;
}

function envString(key: string, fallback: string): string {
    const raw = process.env[key];
    return raw !== undefined && raw !== '' ? raw : fallback;
}

function envSet(key: string, fallback: Set<string>): Set<string> {
    const raw = process.env[key];
    if (raw === undefined || raw === '') return fallback;
    return new Set(raw.split(',').map(s => s.trim()).filter(Boolean));
}

// ── Default values ───────────────────────────────────────────────────

const DEFAULT_DOCS_DIR = path.resolve(process.cwd(), 'docs');
const DEFAULT_ALLOWED_EXTENSIONS = new Set(['.md', '.txt', '.json']);
const DEFAULT_MAX_FILE_SIZE_BYTES = 20 * 1024 * 1024; // 20 MB

// ── Exported config ──────────────────────────────────────────────────

const PREFIX = 'YOUR_MCP_';

export const DOCS_DIR = envString(`${PREFIX}DOCS_DIR`, DEFAULT_DOCS_DIR);
export const ALLOWED_EXTENSIONS = envSet(`${PREFIX}ALLOWED_EXTENSIONS`, DEFAULT_ALLOWED_EXTENSIONS);
export const MAX_FILE_SIZE_BYTES = envInt(`${PREFIX}MAX_FILE_SIZE_BYTES`, DEFAULT_MAX_FILE_SIZE_BYTES);
```

**Best Practices**:
- **Use a unique prefix** for all environment variables to avoid collisions
- **Provide sensible defaults** so the server works out-of-the-box
- **Helper functions** ensure type-safe parsing with fallbacks
- **Export constants** (not functions) for easy consumption

### Step 3: Logging System

Create `src/logger.ts` for rich, structured logging:

```typescript
import chalk from 'chalk';

type LogLevel = 'info' | 'warn' | 'error';

export class Logger {
    private scope: string;

    constructor(scope: string) {
        this.scope = scope;
    }

    private log(level: LogLevel, message: string, extra?: Record<string, unknown>): void {
        const timestamp = new Date().toLocaleTimeString('en-US', { hour12: false });
        const prefix = `${chalk.dim(timestamp)} ${chalk.magenta(`[${this.scope}]`)}`;
        
        // MCP servers write logs to stderr to keep stdout clean for protocol
        console.error(`${prefix} ${message}`);
        
        if (extra) {
            for (const [key, value] of Object.entries(extra)) {
                console.error(`  ${chalk.dim(key + ':')} ${String(value)}`);
            }
        }
    }

    info(message: string, extra?: Record<string, unknown>): void {
        this.log('info', chalk.cyan(message), extra);
    }

    warn(message: string, extra?: Record<string, unknown>): void {
        this.log('warn', chalk.yellow(message), extra);
    }

    error(message: string, extra?: Record<string, unknown>): void {
        this.log('error', chalk.red(message), extra);
    }

    header(title: string): void {
        console.error('');
        console.error(`  ${chalk.cyan('━'.repeat(50))}`);
        console.error(`  ${chalk.bold.white(title)}`);
        console.error(`  ${chalk.cyan('━'.repeat(50))}`);
        console.error('');
    }

    success(message: string): void {
        console.error(`${chalk.green('✔')} ${chalk.green.bold(message)}`);
    }

    toolStart(toolName: string, params?: Record<string, unknown>): void {
        const paramStr = params
            ? Object.entries(params)
                .filter(([, v]) => v !== undefined)
                .map(([k, v]) => `${k}=${v}`)
                .join(' ')
            : '';
        console.error(`${chalk.green('▶')} ${chalk.bgGreen.black(' TOOL ')} ${chalk.green.bold(toolName)} ${paramStr}`);
    }

    toolEnd(toolName: string, durationMs: number, summary?: string): void {
        const dur = durationMs < 1000 ? `${Math.round(durationMs)}ms` : `${(durationMs / 1000).toFixed(2)}s`;
        console.error(`${chalk.green('✔')} ${chalk.green.bold(toolName)} ${chalk.dim(`(${dur})`)} ${summary || ''}`);
    }
}
```

**Critical Points**:
- **Always log to stderr** - MCP protocol uses stdout for JSON-RPC messages
- **Structured logs** with timestamps and scopes for debugging
- **Color coding** makes logs scannable during development
- **Tool lifecycle methods** (`toolStart`, `toolEnd`) for observability

### Step 4: Abstract Tool Base Class

Create `src/tool.ts` for the tool abstraction:

```typescript
import type { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import type { CallToolResult } from '@modelcontextprotocol/sdk/types.js';
import type { AnySchema, ZodRawShapeCompat } from '@modelcontextprotocol/sdk/server/zod-compat.js';
import type { ToolAnnotations } from '@modelcontextprotocol/sdk/types.js';
import { Logger } from './logger.js';

export type ToolConfig<InputArgs extends ZodRawShapeCompat | AnySchema | undefined> = {
    title?: string;
    description?: string;
    inputSchema?: InputArgs;
    outputSchema?: ZodRawShapeCompat | AnySchema;
    annotations?: ToolAnnotations;
    _meta?: Record<string, unknown>;
};

export abstract class Tool<TInput extends ZodRawShapeCompat | AnySchema | undefined = undefined> {
    protected readonly log = new Logger('tool');

    abstract readonly name: string;
    abstract readonly definition: ToolConfig<TInput>;
    abstract execute(args: TInput extends undefined ? undefined : Record<string, unknown>, extra: unknown): Promise<CallToolResult>;

    register(server: McpServer): void {
        // eslint-disable-next-line @typescript-eslint/no-explicit-any
        server.registerTool(this.name, this.definition as any, (async (...callArgs: any[]) => {
            const args = callArgs.length > 1 ? callArgs[0] : undefined;
            const extra = callArgs.length > 1 ? callArgs[1] : callArgs[0];
            return this.execute(args, extra);
        }) as any);
    }
}
```

**Key Benefits**:
- **Consistent interface** for all tools
- **Type-safe** with generic `TInput` parameter
- **Self-registering** via `register()` method
- **Built-in logging** through protected `log` property
- **Clear separation** of definition (metadata) and implementation (execute)

### Step 5: Implement Individual Tools

Each tool extends the `Tool` base class. Here's a complete example:

```typescript
import { z } from 'zod';
import type { CallToolResult } from '@modelcontextprotocol/sdk/types.js';
import { Tool, type ToolConfig } from '../tool.js';

// Define input schema with Zod
const inputSchema = z.object({
    query: z.string().describe('Natural language search query'),
    topK: z.number().optional().default(5).describe('Max results to return'),
    minScore: z.number().optional().default(0.2).describe('Minimum similarity threshold'),
});

type InputSchema = typeof inputSchema;

export class SearchDocumentationTool extends Tool<InputSchema> {
    readonly name = 'search-documentation';

    readonly definition: ToolConfig<InputSchema> = {
        title: 'Search Documentation',
        description: 'Semantic similarity search over team documentation',
        inputSchema,
    };

    private readonly indexer: FileIndexer;

    constructor(indexer: FileIndexer) {
        super();
        this.indexer = indexer;
    }

    async execute(args: z.infer<InputSchema>): Promise<CallToolResult> {
        this.log.toolStart(this.name, { query: args.query, topK: args.topK });
        const start = Date.now();

        try {
            const results = await this.indexer.search(args.query, args.topK, args.minScore);
            
            this.log.toolEnd(this.name, Date.now() - start, `${results.length} results`);

            return {
                content: [{
                    type: 'text',
                    text: JSON.stringify({
                        query: args.query,
                        resultCount: results.length,
                        results,
                    }, null, 2),
                }],
            };
        } catch (err) {
            this.log.error('Search failed', { error: String(err) });
            throw err;
        }
    }
}
```

**Tool Implementation Checklist**:
- ✅ Define Zod schema for input validation
- ✅ Extend `Tool<InputSchema>`
- ✅ Set unique `name` (kebab-case)
- ✅ Provide clear `title` and `description`
- ✅ Accept dependencies via constructor
- ✅ Log tool start and end
- ✅ Return JSON-formatted results
- ✅ Handle errors gracefully

### Step 6: Server Entry Point

Create `server.ts` to wire everything together:

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { Logger } from './src/logger.js';
import { FileIndexer } from './src/file-indexer.js';
import { DocumentLoader } from './src/document-loader.js';
import { SearchDocumentationTool } from './src/tools/search-documentation.js';
import { DOCS_DIR, ALLOWED_EXTENSIONS, MAX_FILE_SIZE_BYTES } from './src/config.js';

// ── Instantiate core components ─────────────────────────────────────

const indexer = new FileIndexer({
    embeddingModel: 'Xenova/all-MiniLM-L6-v2',
    cacheDir: './docs/.cache',
});

const loader = new DocumentLoader({
    docsDir: DOCS_DIR,
    allowedExtensions: ALLOWED_EXTENSIONS,
    maxFileSizeBytes: MAX_FILE_SIZE_BYTES,
});

// ── Create MCP server ────────────────────────────────────────────────

const server = new McpServer({
    name: 'your-mcp-server',
    version: '1.0.0',
    description: 'Your MCP server description',
});

// ── Register tools ───────────────────────────────────────────────────

const tools = [
    new SearchDocumentationTool(indexer),
    // Add more tools here
];

for (const tool of tools) {
    tool.register(server);
}

// ── Main startup sequence ────────────────────────────────────────────

async function main() {
    const mainLog = new Logger('server');
    mainLog.header('Your MCP Server v1.0.0');
    mainLog.info('Starting up...');

    // Perform any initialization (load data, build indexes, etc.)
    const documents = await loader.loadDocuments();
    await indexer.indexDocuments(documents);

    // Connect to transport (stdio for MCP)
    const transport = new StdioServerTransport();
    await server.connect(transport);
    
    mainLog.success('Server connected and ready for requests');
}

main().catch((error) => {
    const errorLog = new Logger('server');
    errorLog.error('Fatal error in main()', { error: String(error) });
    process.exit(1);
});
```

**Startup Sequence Best Practices**:
1. **Instantiate dependencies** before creating the server
2. **Create the MCP server** with metadata
3. **Register all tools** in a loop
4. **Perform initialization** (load data, build indexes)
5. **Connect transport** last (server is now live)
6. **Log success** for debugging
7. **Catch and log fatal errors** with proper exit codes

### Step 7: Core Business Logic

Keep business logic separate from MCP protocol details. Example structure for a document indexer:

```typescript
export class FileIndexer {
    private chunks: IndexedChunk[] = [];
    private embedderPromise: Promise<EmbedderFunction> | null = null;

    constructor(options: FileIndexerOptions) {
        // Store configuration
    }

    // Public API
    async indexDocuments(documents: DocumentInput[]): Promise<void> {
        // Load from cache or compute embeddings
    }

    async search(query: string, topK: number, minScore: number): Promise<SearchResult[]> {
        // Embed query and rank by cosine similarity
    }

    getIndexedFiles(): string[] {
        // Return list of indexed files
    }

    // Private methods
    private async getEmbedder() { /* ... */ }
    private async embedText(text: string): Promise<number[]> { /* ... */ }
    private chunkText(text: string): Chunk[] { /* ... */ }
    private async saveCachedChunks(): Promise<void> { /* ... */ }
    private async loadCachedChunks(): Promise<Chunk[] | null> { /* ... */ }
}
```

**Design Principles**:
- **Public API** methods are tool-facing
- **Private methods** handle implementation details
- **Async initialization** (lazy-load heavy resources like embedding models)
- **Caching strategy** to avoid redundant computation
- **Type-safe data structures** for all inputs/outputs

## Advanced Patterns

### Pattern 1: Intelligent Caching

```typescript
private getCacheFilePath(relativePath: string): string {
    const baseName = relativePath.replace(/\//g, '__');
    return path.join(this.cacheDir, `${baseName}.json`);
}

private async loadCachedChunks(relativePath: string, sourceModifiedMs: number): Promise<IndexedChunk[] | null> {
    const cachePath = this.getCacheFilePath(relativePath);
    try {
        const cacheStat = await fs.stat(cachePath);
        if (cacheStat.mtimeMs < sourceModifiedMs) {
            return null; // cache is stale
        }
        const raw = await fs.readFile(cachePath, 'utf-8');
        return JSON.parse(raw) as IndexedChunk[];
    } catch {
        return null; // cache doesn't exist or is invalid
    }
}
```

**Key Points**:
- Compare `sourceModifiedMs` vs `cacheStat.mtimeMs` for invalidation
- Return `null` on cache miss or staleness
- Graceful fallback on errors

### Pattern 2: Lazy Initialization

```typescript
private embedderPromise: Promise<EmbedderFunction> | null = null;

private async getEmbedder() {
    if (!this.embedderPromise) {
        this.embedderPromise = pipeline('feature-extraction', this.embeddingModel);
    }
    return this.embedderPromise;
}
```

**Benefits**:
- Expensive resources loaded only once
- No blocking during server startup
- First request triggers initialization

### Pattern 3: Progressive Logging

```typescript
for (const [fileIdx, doc] of documents.entries()) {
    const cached = await this.loadCachedChunks(doc.relativePath, doc.sourceModifiedMs);
    
    if (cached) {
        nextChunks.push(...cached);
        log.step(fileIdx + 1, documents.length, `Loaded ${doc.relativePath} from cache`);
        continue;
    }

    // Index from scratch
    const chunks = await this.indexDocument(doc);
    log.step(fileIdx + 1, documents.length, `Indexed ${doc.relativePath} → ${chunks.length} chunks`);
}
```

**Benefits**:
- User sees progress for long operations
- Easy to debug which files are cached vs recomputed
- Step numbers make logs scannable

### Pattern 4: Security - Path Validation

```typescript
async execute(args: { filePath: string }): Promise<CallToolResult> {
    // Normalize and prevent directory traversal
    const normalizedPath = path.normalize(args.filePath).replace(/^(\.\.[\/\\])+/, '');
    const fullPath = path.join(this.docsDir, normalizedPath);

    // Security check: ensure resolved path is within docsDir
    const resolvedPath = path.resolve(fullPath);
    const resolvedDocsDir = path.resolve(this.docsDir);

    if (!resolvedPath.startsWith(resolvedDocsDir)) {
        throw new Error('Invalid file path: cannot access files outside docs directory');
    }

    // Now safe to read the file
    const content = await fs.readFile(fullPath, 'utf-8');
    // ...
}
```

**Security Checklist**:
- ✅ Normalize paths to remove `../` sequences
- ✅ Resolve absolute paths for comparison
- ✅ Validate resolved path is within allowed directory
- ✅ Use `path.join()` instead of string concatenation

### Pattern 5: Error Handling in Tools

```typescript
async execute(args: z.infer<InputSchema>): Promise<CallToolResult> {
    this.log.toolStart(this.name, args);
    const start = Date.now();

    try {
        // Tool implementation
        const result = await this.performOperation(args);
        
        this.log.toolEnd(this.name, Date.now() - start, 'Success');
        
        return {
            content: [{ type: 'text', text: JSON.stringify(result, null, 2) }],
        };
    } catch (error) {
        const errorMsg = error instanceof Error ? error.message : String(error);
        this.log.error(`${this.name} failed`, { error: errorMsg });
        
        return {
            content: [{ 
                type: 'text', 
                text: JSON.stringify({ error: errorMsg }, null, 2) 
            }],
            isError: true,
        };
    }
}
```

**Error Handling Best Practices**:
- ✅ Try-catch around all tool operations
- ✅ Extract error message safely (check `instanceof Error`)
- ✅ Log errors for debugging
- ✅ Return JSON error response (don't throw)
- ✅ Set `isError: true` flag
- ✅ Always log duration even on error

## Testing and Debugging

### Using MCP Inspector

The MCP Inspector is essential for testing tools interactively:

```bash
npm run inspect
```

Then navigate to `http://localhost:6274` in your browser to:
- See all registered tools
- Inspect schemas and descriptions
- Execute tools with custom inputs
- View responses and debug issues

### Logging Strategy

**Development**: Rich, colorful logs with full context

```typescript
log.header('Indexing Documents');
log.step(1, 10, 'Processing file 1/10');
log.stat('Total chunks', 150);
log.success('Indexing complete');
```

**Production**: Structured logs that can be parsed

```typescript
console.error(JSON.stringify({
    level: 'info',
    timestamp: new Date().toISOString(),
    scope: 'indexer',
    message: 'Indexing complete',
    chunks: 150,
}));
```

## Performance Optimization

### 1. Chunking Strategy

```typescript
private chunkText(text: string): Chunk[] {
    const words = normalizeWhitespace(text).split(' ');
    const chunks: Chunk[] = [];
    
    const step = this.chunkSize - this.overlap;
    
    for (let start = 0; start < words.length; start += step) {
        const end = Math.min(words.length, start + this.chunkSize);
        const chunkWords = words.slice(start, end);
        
        if (chunkWords.length < 15) continue; // Skip tiny chunks
        
        chunks.push({
            text: chunkWords.join(' '),
            startWord: start,
            endWord: end,
        });
        
        if (end >= words.length) break;
    }
    
    return chunks;
}
```

**Optimization Tips**:
- Skip very small chunks (< 15 words)
- Use word-based chunking for semantic coherence
- Overlap chunks for context continuity
- Normalize whitespace first

### 2. Embedding Batching

For many documents, batch embeddings:

```typescript
async embedBatch(texts: string[]): Promise<number[][]> {
    const embedder = await this.getEmbedder();
    const results = await Promise.all(
        texts.map(text => embedder(text, { pooling: 'mean', normalize: true }))
    );
    return results.map(r => Array.from(r.data));
}
```

### 3. Search Optimization

```typescript
async search(query: string, topK: number, minScore: number): Promise<SearchResult[]> {
    const queryEmbedding = await this.embedText(query);
    
    const scored = this.chunks
        .map(chunk => ({
            ...chunk,
            score: cosineSimilarity(queryEmbedding, chunk.embedding),
        }))
        .filter(item => item.score >= minScore)
        .sort((a, b) => b.score - a.score)
        .slice(0, topK);
    
    return scored.map(item => ({
        score: item.score,
        filePath: item.filePath,
        chunkIndex: item.chunkIndex,
        text: item.text,
    }));
}
```

**Key Optimizations**:
- Filter by `minScore` before sorting (reduce work)
- Use `.slice(0, topK)` after sorting (limit results)
- Map to clean result structure (hide internal details)

## Common Pitfalls and Solutions

### Pitfall 1: Logging to stdout

**Problem**: Logging to `console.log()` breaks MCP protocol

**Solution**: Always use `console.error()` for logs

```typescript
// ❌ BAD
console.log('Processing document...');

// ✅ GOOD
console.error('Processing document...');
```

### Pitfall 2: Missing Input Validation

**Problem**: Unvalidated inputs cause crashes

**Solution**: Use Zod schemas with descriptive errors

```typescript
const inputSchema = z.object({
    query: z.string().min(1).describe('Non-empty search query'),
    topK: z.number().int().positive().max(100).optional().default(5),
});
```

### Pitfall 3: Synchronous File Operations

**Problem**: Blocking operations freeze the server

**Solution**: Always use async/await with promises

```typescript
// ❌ BAD
const content = fs.readFileSync(filePath, 'utf-8');

// ✅ GOOD
const content = await fs.readFile(filePath, 'utf-8');
```

### Pitfall 4: Not Handling Tool Errors

**Problem**: Throwing errors breaks the MCP connection

**Solution**: Return error results instead of throwing

```typescript
try {
    // operation
} catch (error) {
    return {
        content: [{ type: 'text', text: JSON.stringify({ error: String(error) }) }],
        isError: true,
    };
}
```

### Pitfall 5: Weak Type Safety

**Problem**: Using `any` leads to runtime errors

**Solution**: Define explicit types for everything

```typescript
// ❌ BAD
function processData(data: any): any { /* ... */ }

// ✅ GOOD
interface DocumentInput {
    relativePath: string;
    text: string;
    sourceModifiedMs: number;
}

interface ProcessedDocument {
    id: string;
    chunks: Chunk[];
    embedding: number[];
}

function processData(data: DocumentInput): ProcessedDocument { /* ... */ }
```

## MCP Client Configuration

### Claude Desktop

**macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`

**Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "your-mcp-server": {
      "command": "npx",
      "args": ["-y", "tsx", "/absolute/path/to/your-mcp-server/server.ts"],
      "env": {
        "YOUR_MCP_DOCS_DIR": "/path/to/docs",
        "YOUR_MCP_ALLOWED_EXTENSIONS": ".md,.txt,.json"
      }
    }
  }
}
```

**Configuration Tips**:
- Use absolute paths (avoid `~` and relative paths)
- Pass environment variables via `env` object
- Use unique server name (kebab-case)
- Test with MCP Inspector first before adding to Claude

## Deployment Checklist

Before deploying your MCP server:

- [ ] All TypeScript errors resolved (`npm run build`)
- [ ] Tools tested with MCP Inspector
- [ ] Error handling for all edge cases
- [ ] Input validation with Zod schemas
- [ ] Security checks (path traversal, size limits)
- [ ] Logging to stderr (not stdout)
- [ ] Environment variables documented
- [ ] README with setup instructions
- [ ] Example configuration for MCP clients
- [ ] License file (if publishing)

## Summary

Building a production-quality MCP server requires:

1. **Clean Architecture** - Separate concerns, modular design
2. **Type Safety** - Strict TypeScript, Zod validation
3. **Tool Pattern** - Abstract base class for consistency
4. **Configuration** - Environment variables with defaults
5. **Logging** - Structured logs to stderr
6. **Error Handling** - Graceful failures, never crash
7. **Security** - Validate all inputs, prevent path traversal
8. **Performance** - Caching, lazy loading, batching
9. **Testing** - MCP Inspector for interactive debugging
10. **Documentation** - Clear README and examples

The team-docs-mcp server demonstrates all these patterns in a real-world semantic search application. Use it as a reference for your own MCP server implementations.

## Related Documentation

- [coding-standards.md](../coding-standards.md) - TypeScript coding standards
- [development-setup.md](../development-setup.md) - Development environment setup
- [api-conventions.md](../api-conventions.md) - API design conventions

## References

- [Model Context Protocol Specification](https://modelcontextprotocol.io/)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Zod Documentation](https://zod.dev/)
- [Transformers.js](https://huggingface.co/docs/transformers.js/)

---
