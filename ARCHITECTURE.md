# NTSG System Architecture

Complete architecture overview of the AMD AI Infrastructure debugging system.

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Claude Code Client                       │
│  ┌───────────────────────────────────────────────────────┐  │
│  │           amd-ntsg-debug Plugin (Local)                 │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌─────────┐│  │
│  │  │ 13 Agents│  │ 1 Skill  │  │4 Cmds  │  │.mcp.json││  │
│  │  └──────────┘  └──────────┘  └────────┘  └─────────┘│  │
│  └───────────────────────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTP/SSE
                           │ http://192.168.67.4:8765/sse
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              MCP Server (192.168.67.4:8765)                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │            FastAPI/Starlette Application              │  │
│  │  ├─ /sse          (MCP SSE endpoint)                  │  │
│  │  ├─ /health       (Simple health check)               │  │
│  │  └─ /health/detailed  (Full system status)            │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  MCP Tool Layer (13 tools)            │  │
│  │                                                         │  │
│  │  KB Tools (5):           Lab Tools (8):               │  │
│  │  ├─ ntsg_kb_search      ├─ ntsg_lab_search            │  │
│  │  ├─ ntsg_kb_get_entry   ├─ ntsg_lab_get_server        │  │
│  │  ├─ ntsg_kb_create      ├─ ntsg_lab_get_card          │  │
│  │  ├─ ntsg_kb_list        ├─ ntsg_lab_get_connections   │  │
│  │  └─ ntsg_kb_stats       ├─ ntsg_lab_list              │  │
│  │                          ├─ ntsg_lab_stats             │  │
│  │                          ├─ ntsg_lab_set_credentials   │  │
│  │                          └─ ntsg_lab_import_creds      │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌─────────────────────┐  ┌────────────────────────────┐   │
│  │  Knowledge Base     │  │  Lab Topology              │   │
│  │  ─────────────      │  │  ─────────────             │   │
│  │  ChromaDB Vector    │  │  JSON topology file        │   │
│  │  21 entries         │  │  6 servers, 5 cards        │   │
│  │  RAG/embeddings     │  │  Subsystem metadata        │   │
│  │  Hybrid search      │  │  Encrypted credentials     │   │
│  └─────────────────────┘  └────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Component Breakdown

### 1. Claude Code Plugin (`amd-ntsg-debug`)

**Location:** `~/ntsg/ntsg-claude-plugins/plugins/amd-ntsg-debug/`

**Purpose:** Provides unified debugging interface for AMD AI infrastructure (GPU, scale-up, scale-out)

**Components:**

#### Agents (13)
- **Scaleout (NIC):**
  - `scaleout-debugger` - NIC/RDMA diagnostics
  - `scaleout-explainer` - NIC architecture expert
  - `scaleout-kb-updater` - KB maintenance

- **Scaleup (Switch):**
  - `scaleup-debugger` - Switch diagnostics
  - `scaleup-explainer` - Switch architecture expert
  - `scaleup-kb-updater` - KB maintenance (not yet implemented)

- **GPU:**
  - `gpu-debugger` - GPU/ROCm diagnostics
  - `gpu-explainer` - GPU architecture expert
  - `gpu-kb-updater` - KB maintenance (not yet implemented)

- **Cross-Subsystem:**
  - `cluster-orchestrator` - Multi-subsystem coordination
  - `performance-analyzer` - Performance analysis
  - `kb-validator` - KB quality assurance
  - `docs-publisher` - Confluence sync

#### Skills (1)
- `debug-nic` - Guided NIC debugging workflow (7-step process)

#### Commands (4)
- `/search-kb` - Search knowledge base
- `/find-server` - Find servers/cards in topology
- `/debug-nic` - Start debugging workflow
- `/lab-stats` - Show statistics

#### MCP Integration
- `.mcp.json` - Connects to remote MCP server
- URL: `http://192.168.67.4:8765/sse`

### 2. MCP Server

**Location:** `~/ntsg/ntsg-mcp-server/`

**Purpose:** Provides remote access to KB and lab topology via MCP protocol

**Components:**

#### Server Application (`server/`)
- `app.py` - FastAPI/Starlette application with SSE
- `config.py` - Server configuration
- `health.py` - Health check endpoints

**Endpoints:**
- `GET /` - Server info
- `GET /health` - Simple health check
- `GET /health/detailed` - Full system status
- `GET /sse` - MCP SSE endpoint (for MCP clients)

#### Tool Layer (`tools/`)

**KB Tools (`tools/kb/`):**
- `mcp_server.py` - MCP tool definitions
- `entries.py` - Entry management (CRUD)
- `vector_store.py` - ChromaDB vector store
- `embeddings.py` - Sentence transformer embeddings
- `search.py` - Hybrid/semantic/keyword search

**Lab Tools (`tools/lab/`):**
- `topology.py` - Topology management
- `credentials.py` - Encrypted credential storage
- `cli_helper.py` - CLI utilities

#### Data Stores

**Knowledge Base:**
- Storage: ChromaDB vector store
- Location: `~/.cache/scaleout-debugger/kb-vectors/`
- Entries: 21 (issues, solutions, commands, facts)
- Capabilities: Hybrid search (semantic + keyword)

**Lab Topology:**
- File: `labinfo/vulcano_topology_debug.json`
- Servers: 6 (with subsystem metadata)
- Cards: 5 (Pensando AI NICs)
- Subsystems: `scaleout`, `scaleup`, `gpu`

**Credentials:**
- File: `~/.cache/scaleout-debugger/credentials.enc`
- Format: Encrypted JSON (Fernet)
- Contents: SSH/console/PDU credentials

### 3. Data Flow

#### KB Search Flow
```
User → /search-kb "RDMA timeout"
  → Plugin invokes Skill tool
  → Skill loads, calls ntsg_kb_search via MCP
  → MCP server forwards to KB search
  → ChromaDB performs hybrid search
  → Results returned via SSE
  → Plugin displays to user
```

#### Topology Lookup Flow
```
User → /find-server waco-001
  → Plugin invokes Skill tool
  → Skill loads, calls ntsg_lab_search via MCP
  → MCP server searches topology JSON
  → Server details + connections returned
  → Plugin displays to user
```

#### Debugging Workflow Flow
```
User → /debug-nic --server waco-001
  → Plugin loads debug-nic skill
  → Skill calls ntsg_lab_get_server
  → Skill calls ntsg_kb_search for symptoms
  → User provides symptom info
  → Skill searches KB for matches
  → Skill proposes diagnostic commands
  → User runs commands, provides output
  → Skill analyzes and proposes solutions
  → If new issue, skill calls ntsg_kb_create_entry
```

## Subsystem Architecture

### Scaleout (Pensando AI NICs)
- **Hardware:** AMD Pensando AI NICs
- **Protocols:** RDMA, RoCE, PFC, DCQCN
- **CLI Tool:** `nicctl`
- **KB Categories:** nic, rdma, port, qos, firmware
- **Common Issues:** Link flapping, RDMA timeouts, PFC storms

### Scaleup (Broadcom TH6 Switches)
- **Hardware:** Broadcom Tomahawk 6 switches
- **Purpose:** GPU-to-GPU fabric (scale-up networking)
- **KB Categories:** switch, fabric, routing
- **(Not yet fully implemented in KB/topology)**

### GPU (AMD Instinct)
- **Hardware:** AMD Instinct MI300X GPUs
- **Software:** ROCm stack
- **KB Categories:** gpu, rocm, compute, memory
- **(Not yet fully implemented in KB/topology)**

## File Locations Summary

```
~/ntsg/
├── ai-lab/                              # Source project (symlink)
│   └── projects/scaleout-debugger/      # Original debugger
├── ntsg-claude-plugins/
│   └── plugins/amd-ntsg-debug/            # Claude Code plugin
│       ├── .claude-plugin/plugin.json
│       ├── .mcp.json                    # MCP connection config
│       ├── agents/                      # 13 agents
│       ├── skills/                      # 1 skill
│       └── commands/                    # 4 commands
├── ntsg-mcp-server/                     # MCP server
│   ├── server/                          # FastAPI app
│   ├── tools/                           # MCP tools + KB/lab
│   ├── labinfo/                         # Topology data
│   └── requirements.txt
└── ~/.cache/scaleout-debugger/
    ├── kb-vectors/                      # ChromaDB
    └── credentials.enc                  # Encrypted creds
```

## Network Architecture

```
Claude Code Client
  │ (local or remote)
  │
  ├─ Plugin loaded locally
  │
  └─ HTTP/SSE to 192.168.67.4:8765
       │
       └─ MCP Server (FastAPI)
            │
            ├─ ChromaDB (local)
            ├─ Topology JSON (local)
            └─ Encrypted credentials (local)
```

## Technology Stack

**Client Side:**
- Claude Code CLI
- Plugin system (markdown-based)
- MCP client (SSE transport)

**Server Side:**
- Python 3.10+
- FastAPI/Starlette (async web framework)
- SSE (Server-Sent Events) for MCP transport
- ChromaDB (vector database)
- Sentence Transformers (embeddings)
- Paramiko (SSH, future use)

**Data:**
- JSON (topology)
- ChromaDB (vector store)
- Fernet encryption (credentials)

## Security Considerations

1. **Credentials:**
   - Encrypted at rest (Fernet)
   - Never stored in topology JSON
   - Key stored in `~/.cache/scaleout-debugger/credentials.key`

2. **Network:**
   - MCP server listens on all interfaces (0.0.0.0:8765)
   - No authentication currently (internal network only)
   - Consider adding API key auth for production

3. **SSH Access:**
   - Credentials stored for automation
   - Used by agents to run remote commands
   - Not yet implemented (future enhancement)

## Scalability

**Current:**
- Single server deployment
- 21 KB entries, 6 servers, 5 cards
- Local ChromaDB

**Future:**
- Multi-server MCP deployment (nginx load balancer)
- Shared ChromaDB or migration to Pinecone
- Distributed topology (per-rack servers)
- Caching layer (Redis)

## Monitoring & Observability

**Health Checks:**
- `/health` - Simple liveness probe
- `/health/detailed` - Component status

**Logs:**
- MCP server: `/tmp/mcp-server.log`
- Claude Code: varies by installation

**Metrics:**
- KB entry count
- Topology resource count
- Tool call latency (future)

## Next Steps

1. **Test plugin integration** - Install and verify MCP connection
2. **Add GPU/Scaleup topology** - Expand topology with GPU/switch data
3. **Populate KB** - Add more entries for GPU and switch issues
4. **Add authentication** - API key or token-based auth
5. **Add SSH automation** - Enable agents to run remote commands
6. **Add monitoring** - Prometheus metrics, dashboards
7. **Add Slack integration** - Notifications for critical issues
