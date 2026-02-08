# MCP (Model Context Protocol) Ecosystem

> Research compiled 2026-02-08

## 1. What Is MCP?

MCP is an **open-source protocol** (created by Anthropic, now community-governed) that standardises how AI applications connect to external data sources, tools, and workflows. Think of it as **USB-C for AI** — one standard interface that any AI app can use to talk to any service.

- **Protocol**: JSON-RPC 2.0 over stdio or Streamable HTTP
- **Spec**: https://modelcontextprotocol.io/specification/2025-03-26
- **Registry**: https://registry.modelcontextprotocol.io
- **GitHub**: https://github.com/modelcontextprotocol

### Why It Matters

Before MCP, every AI app built bespoke integrations for every service (N×M problem). MCP reduces this to N+M: each service builds one MCP server, each AI app builds one MCP client.

---

## 2. Architecture

### Participants

| Role | Description | Examples |
|------|-------------|----------|
| **Host** | AI application that coordinates MCP clients | Claude Desktop, Claude Code, VS Code, Cursor, Windsurf |
| **Client** | Component within host that maintains connection to one server | Created per-server by the host |
| **Server** | Program that provides context/tools to clients | Filesystem server, GitHub server, Slack server |

### Two Layers

1. **Data Layer** (inner): JSON-RPC protocol — lifecycle management, capability negotiation, tools/resources/prompts
2. **Transport Layer** (outer): Communication mechanism — stdio or Streamable HTTP

### Transport Mechanisms

| Transport | How It Works | Use Case |
|-----------|-------------|----------|
| **stdio** | Client spawns server as subprocess, communicates via stdin/stdout | Local servers, single client |
| **Streamable HTTP** | HTTP POST + optional SSE streaming | Remote servers, multi-client, production deployments |

---

## 3. Core Primitives (Server Features)

### Tools
Functions the AI model can call. The primary mechanism for agents to take actions.

```json
{
  "name": "create_issue",
  "description": "Create a GitHub issue",
  "inputSchema": {
    "type": "object",
    "properties": {
      "repo": { "type": "string" },
      "title": { "type": "string" },
      "body": { "type": "string" }
    }
  }
}
```

### Resources
Contextual data the model or user can read. Like files, database records, API responses.

```json
{
  "uri": "file:///project/README.md",
  "name": "Project README",
  "mimeType": "text/markdown"
}
```

### Prompts
Templated messages and workflows. Reusable interaction patterns exposed by servers.

### Client Features (reverse direction)
- **Sampling**: Server can ask the client to run an LLM completion (enables agentic recursion)
- **Elicitation**: Server can request user input
- **Logging**: Server can send log messages to client

---

## 4. Available SDKs (10 languages)

| Language | Repo | Tier |
|----------|------|------|
| **TypeScript** | `modelcontextprotocol/typescript-sdk` | Tier 1 (official) |
| **Python** | `modelcontextprotocol/python-sdk` | Tier 1 (official) |
| **Java** | `modelcontextprotocol/java-sdk` | Tier 1 |
| **Kotlin** | `modelcontextprotocol/kotlin-sdk` | Tier 1 |
| **C#** | `modelcontextprotocol/csharp-sdk` | Tier 1 |
| **Go** | `modelcontextprotocol/go-sdk` | Tier 1 |
| **Rust** | `modelcontextprotocol/rust-sdk` | Tier 1 |
| **Swift** | `modelcontextprotocol/swift-sdk` | Tier 1 |
| **Ruby** | `modelcontextprotocol/ruby-sdk` | Community |
| **PHP** | `modelcontextprotocol/php-sdk` | Community |

All SDKs support: creating servers (tools/resources/prompts), building clients, both transports, and type-safe protocol compliance.

---

## 5. Available MCP Servers — Landscape Map

### Reference Servers (Official)

| Server | Purpose |
|--------|---------|
| **Everything** | Test/reference server demonstrating all features |
| **Fetch** | Web content fetching and conversion |
| **Filesystem** | Secure file operations with access controls |
| **Git** | Read, search, manipulate git repos |
| **Memory** | Knowledge-graph persistent memory |
| **Sequential Thinking** | Dynamic problem-solving through thought sequences |
| **Time** | Time and timezone conversion |

### Major Official Integration Servers

| Category | Servers |
|----------|---------|
| **Dev Platforms** | GitHub, GitLab, Sentry, Linear, Jira |
| **Communication** | Slack (Zencoder-maintained), Discord |
| **Cloud/Infra** | AWS KB Retrieval, Alibaba Cloud (AnalyticDB, RDS, DataWorks, OpenSearch, OPS), Aiven (Postgres, Kafka, ClickHouse) |
| **Databases** | PostgreSQL, SQLite, Redis, Snowflake (via Alkemi), BigQuery |
| **Search** | Brave Search, Exa, Algolia |
| **Design/Media** | Figma, 21st.dev (UI components), Blender, EverArt |
| **Productivity** | Google Drive, Google Maps, Google Calendar, Notion |
| **Browser/Web** | Puppeteer, Playwright, AgentQL |
| **Payments** | Stripe, Alpaca (trading), Alby (Bitcoin), Adfin, AlipayPlus |
| **Observability** | AgentOps, Sentry |
| **Multi-SaaS** | Paragon ActionKit (130+ SaaS via one server), AgentRPC |

### The MCP Registry

The official registry at `registry.modelcontextprotocol.io` is the canonical directory for published MCP servers. Hundreds of servers are listed, covering virtually every major SaaS platform.

---

## 6. MCP Clients (Hosts That Support MCP)

| Client | Type | Notes |
|--------|------|-------|
| **Claude Desktop** | Desktop app | First-class MCP support, config via `claude_desktop_config.json` |
| **Claude Code** | CLI agent | Native MCP, uses filesystem/git servers |
| **VS Code** (Copilot) | IDE | Built-in MCP client support |
| **Cursor** | IDE | MCP integration for tool use |
| **Windsurf** | IDE | MCP support |
| **Cline** | VS Code extension | MCP-native |
| **Continue** | IDE extension | MCP support |
| **OpenClaw** | Agent platform | Uses skills/tools system (see §8) |
| **Zed** | Editor | MCP integration |
| **ChatGPT** | Web/app | Added MCP support |

---

## 7. How to Build a Custom MCP Server

### Minimal TypeScript Server (stdio)

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";

const server = new McpServer({
  name: "my-server",
  version: "1.0.0"
});

// Register a tool
server.tool(
  "greet",
  "Greet someone by name",
  { name: { type: "string", description: "Name to greet" } },
  async ({ name }) => ({
    content: [{ type: "text", text: `Hello, ${name}!` }]
  })
);

// Register a resource
server.resource(
  "status",
  "app://status",
  async () => ({
    contents: [{ uri: "app://status", text: "System OK", mimeType: "text/plain" }]
  })
);

// Connect via stdio
const transport = new StdioServerTransport();
await server.connect(transport);
```

### Minimal Python Server

```python
from mcp.server import Server
from mcp.server.stdio import stdio_server

app = Server("my-server")

@app.tool()
async def greet(name: str) -> str:
    """Greet someone by name"""
    return f"Hello, {name}!"

async def main():
    async with stdio_server() as (read, write):
        await app.run(read, write)

import asyncio
asyncio.run(main())
```

### Remote Server (Streamable HTTP)

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { StreamableHTTPServerTransport } from "@modelcontextprotocol/sdk/server/streamableHttp.js";
import express from "express";

const app = express();
const server = new McpServer({ name: "remote-server", version: "1.0.0" });

// Add tools...

app.post("/mcp", async (req, res) => {
  const transport = new StreamableHTTPServerTransport("/mcp");
  await server.connect(transport);
  await transport.handleRequest(req, res);
});

app.listen(3000);
```

### Connecting to Claude Desktop

Add to `~/.config/claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "my-server": {
      "command": "node",
      "args": ["/path/to/my-server/index.js"]
    }
  }
}
```

### Development & Testing

- **MCP Inspector**: Interactive debugging tool (`npx @modelcontextprotocol/inspector`)
- **Claude Desktop Developer Tools**: View MCP logs via `Command+Option+Shift+i`

---

## 8. OpenClaw and MCP

OpenClaw uses a **skills-based architecture** rather than directly implementing MCP client protocol. Key observations:

- **OpenClaw's tool system** provides similar capabilities to MCP: tools, resources, browser control, node management, messaging — but through its own plugin/skill framework
- **Skills** in OpenClaw are the equivalent of MCP servers — they provide tools and capabilities to the agent
- OpenClaw **could integrate MCP** by building an MCP client skill that connects to any MCP server, effectively bridging the two ecosystems
- The current OpenClaw architecture (exec, browser, canvas, nodes, message, etc.) covers many use cases that MCP servers address, but through direct tool integration rather than the MCP protocol

### Potential MCP Integration Path for OpenClaw

1. Build an OpenClaw skill that acts as an MCP client
2. Configure it to connect to any MCP server (stdio or HTTP)
3. Dynamically expose MCP server tools as OpenClaw tools
4. This would give OpenClaw agents access to the entire MCP ecosystem

---

## 9. Using MCP to Give Agents Access to Any Service

### The Pattern

1. **Find or build an MCP server** for the target service
2. **Configure your MCP host** (Claude Desktop, VS Code, etc.) to connect to it
3. The AI agent automatically discovers available tools and can call them

### Giving an Agent Access to a Database

```json
{
  "mcpServers": {
    "my-database": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": {
        "POSTGRES_CONNECTION_STRING": "postgresql://user:pass@localhost/mydb"
      }
    }
  }
}
```

The agent can now query the database, inspect schemas, and analyze data.

### Giving an Agent Access to GitHub

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "ghp_..."
      }
    }
  }
}
```

### Composing Multiple Servers

An agent can connect to **multiple MCP servers simultaneously**, giving it access to many services at once:

```json
{
  "mcpServers": {
    "filesystem": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-filesystem", "/home/user/projects"] },
    "github": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-github"] },
    "slack": { "command": "npx", "args": ["-y", "@zencoderai/slack-mcp-server"] },
    "postgres": { "command": "npx", "args": ["-y", "@modelcontextprotocol/server-postgres"] }
  }
}
```

This single config gives the agent filesystem access, GitHub integration, Slack messaging, and database queries.

---

## 10. Security Considerations

| Concern | MCP's Approach |
|---------|---------------|
| **User Consent** | Hosts MUST get explicit consent before tool invocation |
| **Data Privacy** | Hosts must not transmit user data without consent |
| **Tool Safety** | Tool descriptions are untrusted; users must understand what tools do |
| **Sampling Controls** | Users approve all LLM sampling requests from servers |
| **DNS Rebinding** | HTTP servers must validate Origin headers |
| **Access Control** | Filesystem server supports configurable allowed directories |

### Trust Model
- Tool annotations/descriptions from servers should be treated as **untrusted** unless from a verified source
- MCP servers have the same trust level as any third-party code — review before deploying

---

## 11. Key Takeaways

1. **MCP is the emerging standard** for AI-tool integration — backed by Anthropic, adopted by OpenAI, Microsoft, JetBrains, and hundreds of companies
2. **10 official SDKs** make it accessible in any language
3. **Hundreds of pre-built servers** exist for major services (GitHub, Slack, databases, cloud providers, etc.)
4. **Building a custom server is simple** — ~20 lines of code for a basic tool
5. **Two transports**: stdio for local, Streamable HTTP for remote/production
6. **Composable**: agents can connect to many servers simultaneously
7. **OpenClaw has its own tool system** that overlaps with MCP's goals; an MCP client bridge would unlock the entire MCP ecosystem for OpenClaw agents
8. **The MCP Registry** (`registry.modelcontextprotocol.io`) is the canonical directory for discovering servers
9. **Security is protocol-level concern** — consent, access control, and trust boundaries are baked into the spec

---

## 12. Resources

- **Spec**: https://modelcontextprotocol.io/specification/2025-03-26
- **Docs**: https://modelcontextprotocol.io/docs
- **SDKs**: https://modelcontextprotocol.io/docs/sdk
- **Reference Servers**: https://github.com/modelcontextprotocol/servers
- **Registry**: https://registry.modelcontextprotocol.io
- **Inspector**: `npx @modelcontextprotocol/inspector`
- **Awesome MCP Servers**: https://github.com/punkpeye/awesome-mcp-servers
