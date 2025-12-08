# MCP Marketplace Launch Strategy

**Document Version:** 1.0
**Last Updated:** 2025-12-07
**Status:** URGENT - Launch ASAP

---

## 🎯 Quick Win: MCP Marketplace Launch

**Goal:** Get FLTR in front of developers using Claude Desktop/MCP within 1 week

**Why This Works:**
- MCP marketplace has built-in distribution
- Developers are early adopters
- Showcases FLTR's core capability
- Zero marketing cost
- Viral potential (devs share tools)

---

## 🚀 Minimum Viable MCP Server

**Core Functionality:**
1. Upload file (PDF, TXT, MD, DOCX)
2. RAG processing (chunk → embed → store)
3. Query with natural language
4. Return answers with citations

**That's it.** No complex features needed for v1.

---

## 📝 Implementation Plan (3-5 Days)

### Day 1: Core MCP Server Setup

**File:** `/mcp-server/src/index.ts`

```typescript
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

// Initialize MCP server
const server = new Server(
  {
    name: "fltr-rag",
    version: "1.0.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

// Define tools
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: [
    {
      name: "upload_and_index",
      description: "Upload a file and index it for semantic search. Supports PDF, TXT, MD, DOCX.",
      inputSchema: {
        type: "object",
        properties: {
          file_path: {
            type: "string",
            description: "Absolute path to the file to upload and index",
          },
          document_id: {
            type: "string",
            description: "Unique ID for this document (optional, auto-generated if not provided)",
          },
        },
        required: ["file_path"],
      },
    },
    {
      name: "query_documents",
      description: "Search indexed documents using natural language. Returns relevant passages with citations.",
      inputSchema: {
        type: "object",
        properties: {
          query: {
            type: "string",
            description: "Natural language question or search query",
          },
          limit: {
            type: "number",
            description: "Maximum number of results to return (default: 5)",
          },
          document_ids: {
            type: "array",
            items: { type: "string" },
            description: "Optional: Limit search to specific documents",
          },
        },
        required: ["query"],
      },
    },
    {
      name: "list_documents",
      description: "List all indexed documents with metadata",
      inputSchema: {
        type: "object",
        properties: {},
      },
    },
    {
      name: "delete_document",
      description: "Remove a document from the index",
      inputSchema: {
        type: "object",
        properties: {
          document_id: {
            type: "string",
            description: "ID of document to delete",
          },
        },
        required: ["document_id"],
      },
    },
  ],
}));

// Implement tool handlers
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  const { name, arguments: args } = request.params;

  switch (name) {
    case "upload_and_index":
      return await handleUploadAndIndex(args);
    case "query_documents":
      return await handleQuery(args);
    case "list_documents":
      return await handleListDocuments(args);
    case "delete_document":
      return await handleDeleteDocument(args);
    default:
      throw new Error(`Unknown tool: ${name}`);
  }
});

// Start server
async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("FLTR MCP server running on stdio");
}

main().catch(console.error);
```

### Day 2: FLTR API Integration

**File:** `/mcp-server/src/fltr-client.ts`

```typescript
interface FltrConfig {
  apiUrl: string;
  apiKey: string;
  sessionToken?: string;
}

export class FltrClient {
  constructor(private config: FltrConfig) {}

  async uploadDocument(filePath: string, documentId?: string) {
    const formData = new FormData();
    const file = await fs.readFile(filePath);
    formData.append("file", new Blob([file]));

    const response = await fetch(
      `${this.config.apiUrl}/api/documents/upload`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${this.config.sessionToken}`,
        },
        body: formData,
      }
    );

    return response.json();
  }

  async queryDocuments(query: string, options: {
    limit?: number;
    documentIds?: string[];
  } = {}) {
    const response = await fetch(
      `${this.config.apiUrl}/api/mcp/search`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${this.config.sessionToken}`,
        },
        body: JSON.stringify({
          query,
          limit: options.limit || 5,
          document_ids: options.documentIds,
        }),
      }
    );

    return response.json();
  }

  async listDocuments() {
    const response = await fetch(
      `${this.config.apiUrl}/api/documents`,
      {
        headers: {
          Authorization: `Bearer ${this.config.sessionToken}`,
        },
      }
    );

    return response.json();
  }

  async deleteDocument(documentId: string) {
    const response = await fetch(
      `${this.config.apiUrl}/api/documents/${documentId}`,
      {
        method: "DELETE",
        headers: {
          Authorization: `Bearer ${this.config.sessionToken}`,
        },
      }
    );

    return response.json();
  }
}
```

### Day 3: Tool Handlers

**File:** `/mcp-server/src/handlers.ts`

```typescript
import { FltrClient } from "./fltr-client.js";
import fs from "fs/promises";
import path from "path";

const fltrClient = new FltrClient({
  apiUrl: process.env.FLTR_API_URL || "https://api.fltr.com",
  apiKey: process.env.FLTR_API_KEY!,
  sessionToken: process.env.FLTR_SESSION_TOKEN,
});

export async function handleUploadAndIndex(args: any) {
  const { file_path, document_id } = args;

  // Validate file exists
  try {
    await fs.access(file_path);
  } catch {
    return {
      content: [
        {
          type: "text",
          text: `Error: File not found at ${file_path}`,
        },
      ],
      isError: true,
    };
  }

  // Upload and index
  try {
    const result = await fltrClient.uploadDocument(file_path, document_id);

    return {
      content: [
        {
          type: "text",
          text: JSON.stringify({
            success: true,
            document_id: result.document_id,
            status: result.status,
            message: `Successfully indexed ${path.basename(file_path)}`,
          }, null, 2),
        },
      ],
    };
  } catch (error) {
    return {
      content: [
        {
          type: "text",
          text: `Error indexing document: ${error.message}`,
        },
      ],
      isError: true,
    };
  }
}

export async function handleQuery(args: any) {
  const { query, limit = 5, document_ids } = args;

  try {
    const results = await fltrClient.queryDocuments(query, {
      limit,
      documentIds: document_ids,
    });

    // Format results with citations
    const formattedResults = results.map((result: any, index: number) => {
      return `
**Result ${index + 1}** (Score: ${result.score.toFixed(2)})

${result.text}

*Source:* ${result.metadata.document_name} (Page ${result.metadata.page || "N/A"})
*Document ID:* ${result.document_id}
`;
    }).join("\n---\n");

    return {
      content: [
        {
          type: "text",
          text: formattedResults || "No results found for your query.",
        },
      ],
    };
  } catch (error) {
    return {
      content: [
        {
          type: "text",
          text: `Error querying documents: ${error.message}`,
        },
      ],
      isError: true,
    };
  }
}

export async function handleListDocuments(args: any) {
  try {
    const documents = await fltrClient.listDocuments();

    const formatted = documents.map((doc: any) => {
      return `- **${doc.name}** (ID: ${doc.id})
  Status: ${doc.status}
  Uploaded: ${new Date(doc.created_at).toLocaleDateString()}
  Chunks: ${doc.chunk_count || 0}`;
    }).join("\n");

    return {
      content: [
        {
          type: "text",
          text: `# Indexed Documents\n\n${formatted || "No documents indexed yet."}`,
        },
      ],
    };
  } catch (error) {
    return {
      content: [
        {
          type: "text",
          text: `Error listing documents: ${error.message}`,
        },
      ],
      isError: true,
    };
  }
}

export async function handleDeleteDocument(args: any) {
  const { document_id } = args;

  try {
    await fltrClient.deleteDocument(document_id);

    return {
      content: [
        {
          type: "text",
          text: `Successfully deleted document: ${document_id}`,
        },
      ],
    };
  } catch (error) {
    return {
      content: [
        {
          type: "text",
          text: `Error deleting document: ${error.message}`,
        },
      ],
      isError: true,
    };
  }
}
```

### Day 4: Configuration & Testing

**File:** `/mcp-server/package.json`

```json
{
  "name": "@fltr/mcp-server",
  "version": "1.0.0",
  "description": "MCP server for FLTR - semantic search and RAG for your documents",
  "main": "dist/index.js",
  "type": "module",
  "bin": {
    "fltr-mcp": "./dist/index.js"
  },
  "scripts": {
    "build": "tsc",
    "prepare": "npm run build",
    "inspector": "mcp-inspector node dist/index.js"
  },
  "keywords": [
    "mcp",
    "semantic-search",
    "rag",
    "documents",
    "ai",
    "claude"
  ],
  "author": "FLTR",
  "license": "MIT",
  "dependencies": {
    "@modelcontextprotocol/sdk": "latest",
    "node-fetch": "^3.3.2"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "typescript": "^5.3.0"
  }
}
```

**File:** `/mcp-server/README.md`

```markdown
# FLTR MCP Server

Upload and search your documents using semantic AI search.

## Features

- 📄 Upload PDFs, text files, markdown, and Word documents
- 🔍 Natural language search across all your documents
- 📚 Automatic chunking and indexing
- 🎯 Results with source citations
- ⚡ Fast semantic search powered by FLTR

## Installation

```bash
npm install -g @fltr/mcp-server
```

## Configuration

Add to your Claude Desktop config (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "fltr": {
      "command": "npx",
      "args": ["-y", "@fltr/mcp-server"],
      "env": {
        "FLTR_API_URL": "https://api.fltr.com",
        "FLTR_SESSION_TOKEN": "your-session-token"
      }
    }
  }
}
```

## Getting Your Session Token

1. Sign up at [fltr.com](https://fltr.com)
2. Go to Settings → API Keys
3. Generate a new API key
4. Copy it to your config

## Usage

Once installed, you can use these tools in Claude Desktop:

### Upload and Index a Document

```
Upload this file: /path/to/document.pdf
```

Claude will use the `upload_and_index` tool to process your document.

### Search Documents

```
Search my documents for information about machine learning
```

Claude will use the `query_documents` tool to find relevant passages.

### List Documents

```
Show me all my indexed documents
```

### Delete a Document

```
Delete document with ID: doc-123
```

## Example Workflow

```
User: Upload this research paper: ~/Downloads/transformer-paper.pdf

Claude: I've uploaded and indexed the paper. It has been processed into 45 chunks.

User: What are the key innovations in the transformer architecture?

Claude: Based on the paper, here are the key innovations:

**Result 1** (Score: 0.94)
The Transformer architecture introduces a self-attention mechanism that allows
the model to weigh the importance of different words in a sequence...

*Source:* transformer-paper.pdf (Page 3)

**Result 2** (Score: 0.89)
Unlike recurrent models, the Transformer processes all tokens in parallel,
significantly reducing training time...

*Source:* transformer-paper.pdf (Page 5)
```

## Supported File Types

- PDF (`.pdf`)
- Text (`.txt`)
- Markdown (`.md`)
- Word Documents (`.docx`)

## Features Coming Soon

- Multi-file upload
- Document collections
- Advanced filtering
- OCR for scanned documents

## Powered by FLTR

This MCP server is powered by [FLTR](https://fltr.com) - semantic search for your documents.

Want to search your company's documents? Try FLTR Enterprise.

## License

MIT
```

### Day 5: Polish & Launch

**Tasks:**
1. Test with Claude Desktop locally
2. Create demo video (2 minutes)
3. Publish to npm
4. Submit to MCP marketplace
5. Announce on social media

---

## 🎨 Marketing Assets

### Demo Video Script (2 minutes)

```
[0:00-0:20] Problem
"Ever wished you could just ask questions about your documents instead of searching manually?"

[0:20-0:40] Solution
"Introducing FLTR MCP Server - upload any document and search it with natural language in Claude Desktop"

[0:40-1:00] Demo: Upload
Screen recording:
- Show Claude Desktop
- "Upload this research paper: ~/Downloads/paper.pdf"
- Claude uploads and confirms indexing

[1:00-1:20] Demo: Search
- "What are the main findings in this paper?"
- Claude returns relevant excerpts with citations
- Show source links

[1:20-1:40] Demo: Multiple Documents
- Upload 3 more documents
- "Compare the methodologies across all papers"
- Claude searches all documents

[1:40-2:00] CTA
"Try it now: npm install -g @fltr/mcp-server
Get started at fltr.com"
```

### Social Media Posts

**Twitter/X:**
```
🚀 New MCP server: Search your documents with AI

Just shipped @fltr/mcp-server for Claude Desktop:

📄 Upload PDFs, docs, markdown
🔍 Search with natural language
📚 Get answers with citations
⚡ Powered by semantic search

Try it: npm install -g @fltr/mcp-server

[Demo video]

#AI #MCP #Claude
```

**LinkedIn:**
```
Excited to launch FLTR MCP Server! 🎉

We built an MCP server that brings semantic document search to Claude Desktop.

What it does:
• Upload any document (PDF, DOCX, TXT, MD)
• Ask questions in natural language
• Get answers with source citations
• Search across multiple documents instantly

Perfect for:
• Research
• Documentation
• Knowledge management
• Study notes

Built with the Model Context Protocol and powered by FLTR's semantic search engine.

Try it: npm install -g @fltr/mcp-server

Demo video: [link]

#AI #SemanticSearch #RAG #MCP
```

**Reddit r/ClaudeAI:**
```
Title: [Tool] I built an MCP server for semantic document search

Body:
Hey r/ClaudeAI! I just released an MCP server that lets you upload and search your documents using semantic AI.

**What it does:**
- Upload PDFs, Word docs, text files, markdown
- Search using natural language queries
- Get results with source citations
- Works right in Claude Desktop

**How to use:**
1. Install: `npm install -g @fltr/mcp-server`
2. Add to Claude Desktop config
3. Upload documents and start asking questions

**Example:**
- Upload a research paper
- Ask: "What methodology did they use?"
- Get relevant excerpts with page numbers

I built this because I was tired of manually searching through PDFs. Now I just ask Claude and it finds the answer.

It's powered by FLTR (semantic search platform I'm building).

Demo video: [link]
GitHub: [link]
Docs: [link]

Let me know if you have questions!
```

---

## 📦 MCP Marketplace Listing

### Listing Information

**Name:** FLTR Semantic Document Search

**Short Description:**
Upload and search your documents using AI-powered semantic search. Get answers with citations.

**Long Description:**
```
FLTR brings semantic document search to Claude Desktop via MCP.

Upload any document (PDF, DOCX, TXT, MD) and ask questions in natural language.
Get relevant answers with source citations and page numbers.

Perfect for:
• Research papers and academic work
• Documentation and knowledge bases
• Legal documents and contracts
• Meeting notes and reports
• Study materials

Features:
• Semantic search across all your documents
• Automatic chunking and indexing
• Source citations with page numbers
• Support for multiple file formats
• Fast and accurate results

How it works:
1. Upload your documents
2. Ask questions naturally
3. Get precise answers with sources

Powered by FLTR - production-grade semantic search infrastructure.

Try it free at fltr.com
```

**Category:** Productivity, Research, Documents

**Tags:** semantic-search, documents, rag, ai, pdf, search, knowledge-base

**Author:** FLTR

**Repository:** https://github.com/fltr/mcp-server

**Homepage:** https://fltr.com

**Support:** https://github.com/fltr/mcp-server/issues

**License:** MIT

---

## 🎯 Launch Checklist

### Pre-Launch (Day 1-4)
- [ ] Build MCP server core functionality
- [ ] Test locally with Claude Desktop
- [ ] Write comprehensive README
- [ ] Create example documents for demos
- [ ] Record demo video
- [ ] Set up npm package
- [ ] Create GitHub repository
- [ ] Write CONTRIBUTING.md
- [ ] Add LICENSE file

### Launch Day (Day 5)
- [ ] Publish to npm (`npm publish --access public`)
- [ ] Submit to MCP marketplace
- [ ] Post on Twitter/X
- [ ] Post on LinkedIn
- [ ] Post on Reddit r/ClaudeAI
- [ ] Post on HackerNews (Show HN)
- [ ] Share in MCP Discord
- [ ] Email announcement to FLTR users

### Post-Launch (Week 2)
- [ ] Monitor npm downloads
- [ ] Respond to GitHub issues
- [ ] Engage with social media comments
- [ ] Track FLTR signups from MCP
- [ ] Collect user feedback
- [ ] Plan v1.1 features

---

## 📊 Success Metrics

### Week 1 Goals
- 100 npm downloads
- 10 GitHub stars
- 5 FLTR signups from MCP users
- 50+ social media engagements

### Month 1 Goals
- 500 npm downloads
- 50 GitHub stars
- 25 FLTR signups
- 10 testimonials/feedback

### Month 3 Goals
- 2,000 npm downloads
- 200 GitHub stars
- 100 FLTR signups
- Featured on MCP marketplace homepage

---

## 🔄 Iteration Plan

### v1.0 (Launch - Week 1)
- Upload single file
- Query documents
- List documents
- Delete documents

### v1.1 (Week 2-3)
- Batch upload (multiple files)
- Document collections
- Better error handling
- Progress indicators

### v1.2 (Week 4-6)
- OCR support for scanned PDFs
- Advanced filtering (by date, type)
- Document metadata editing
- Export search results

### v2.0 (Month 2-3)
- Local-first mode (no cloud needed)
- Custom embeddings
- Hybrid search (keyword + semantic)
- Team collaboration

---

## 🎪 HackerNews Launch Strategy

**Title:** "Show HN: MCP server for semantic document search in Claude Desktop"

**Post:**
```
Hi HN,

I built an MCP server that lets you upload and search documents using semantic AI right in Claude Desktop.

The problem I was solving: I have hundreds of research papers, documentation, and notes. Finding specific information across them was painful.

What it does:
• Upload PDFs, Word docs, text files, markdown
• Ask questions in natural language
• Get answers with source citations and page numbers
• Search across all your documents instantly

Example workflow:
1. Upload a research paper: ~/papers/transformer.pdf
2. Ask: "What is the attention mechanism?"
3. Get relevant excerpts with exact page numbers

It's built on the Model Context Protocol (MCP) so it works natively in Claude Desktop with zero setup after installation.

Tech stack:
• TypeScript + MCP SDK
• FastAPI backend for indexing
• Milvus for vector search
• Modal for serverless processing

Try it: npm install -g @fltr/mcp-server
Demo: [link]
GitHub: [link]

It's powered by FLTR, a semantic search platform I'm building. The MCP server is a free tier - perfect for personal use.

Would love feedback from the HN community! What features would make this more useful for you?
```

**Timing:**
- Post Tuesday-Thursday
- 8-10 AM PT
- Prepare for 2-3 hours of active engagement

**Engagement Strategy:**
- Reply to every comment within 30 minutes
- Be helpful, not salesy
- Share technical details when asked
- Acknowledge limitations honestly
- Thank people for feedback

---

## 💡 Viral Mechanics

### Built-in Attribution
Every MCP server interaction shows:
```
🔍 Powered by FLTR
Want to search your company docs? Try FLTR Enterprise
fltr.com
```

### GitHub README
- Clear "Powered by FLTR" section
- Link to FLTR homepage
- Enterprise CTA

### Demo Videos
- FLTR logo in corner
- "Powered by FLTR" end card
- Link to fltr.com in description

### NPM Package
- Keywords include "fltr"
- Homepage links to fltr.com
- README has FLTR branding

### Social Sharing
Every result Claude returns includes:
```
*Powered by FLTR semantic search*
```

When users share results, they share FLTR attribution.

---

## 🎁 Freemium Model

### Free Tier (MCP Server)
- 10 documents max
- 100 queries/day
- Basic search
- Perfect for personal use

### Paid Tier (Drive to FLTR)
- Unlimited documents
- Unlimited queries
- OCR support
- Team collaboration
- API access
- Priority support

**Upgrade CTA in MCP:**
```
You've reached your document limit (10/10).
Upgrade to FLTR Pro for unlimited documents: fltr.com/pricing
```

---

## 🚀 Quick Start (Right Now)

**Today:**
1. Create `/mcp-server` directory
2. Set up npm package structure
3. Start implementing core tools

**This Weekend:**
1. Build FLTR API integration
2. Test locally with Claude Desktop
3. Record quick demo

**Next Week:**
1. Publish to npm
2. Submit to MCP marketplace
3. Launch on HN/Twitter/Reddit
4. Monitor and iterate

---

## ❓ FAQ for Launch

### "How is this different from other document tools?"

"Most tools require you to leave Claude. This works natively in Claude Desktop via MCP. Upload once, search anytime without context switching."

### "What about privacy?"

"Documents are processed securely via FLTR's API. We use industry-standard encryption and don't train on your data. For ultra-sensitive docs, we're building a local-only mode."

### "Can I self-host?"

"Not yet, but it's on the roadmap. For now, we offer a generous free tier (10 docs, 100 queries/day) that works great for personal use."

### "What file types are supported?"

"PDF, DOCX, TXT, and Markdown. OCR for scanned PDFs coming in v1.2."

### "How does billing work?"

"Free tier is 10 documents and 100 queries/day. Paid plans start at $20/month for unlimited documents and queries. See fltr.com/pricing"

---

## 🎯 Expected Outcomes

### Week 1
- 100+ npm downloads
- 10+ GitHub stars
- 500+ social media impressions
- 5-10 FLTR signups

### Month 1
- 500+ npm downloads
- 50+ GitHub stars
- 2,000+ social media impressions
- 25-50 FLTR signups
- 2-3 testimonials

### Month 3
- 2,000+ npm downloads
- 200+ GitHub stars
- Featured on MCP marketplace
- 100+ FLTR signups
- 10+ paying customers from MCP

**ROI:**
- Zero marketing cost
- High-quality developer leads
- Viral distribution potential
- Showcases FLTR capabilities

---

## 🔥 Action Items (Next 24 Hours)

**Do these TODAY:**

1. **Set up repository** (30 min)
   ```bash
   mkdir mcp-server
   cd mcp-server
   npm init -y
   git init
   ```

2. **Copy existing MCP code** (1 hour)
   - Look at `/Users/jamesjulius/Coding/FLTR/markdrop-mcp-server`
   - Adapt for FLTR use case
   - Update tool definitions

3. **Create basic README** (30 min)
   - Installation instructions
   - Quick start guide
   - Example usage

4. **Test locally** (1 hour)
   - Run with mcp-inspector
   - Verify tools work
   - Test with Claude Desktop

5. **Record demo** (1 hour)
   - Screen record workflow
   - Add voiceover or captions
   - Upload to YouTube

**By end of week:**
- Publish to npm
- Submit to marketplace
- Launch on HN/Twitter

---

**REMEMBER:** This is a quick win. Don't overthink it. Get v1.0 out ASAP, then iterate based on feedback.

Ship fast, ship now! 🚢

---

**END OF DOCUMENT**
