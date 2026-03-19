# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**amd-ai-infra** is an AI-powered unified debugging and knowledge management system for AMD's complete AI cluster infrastructure stack. It combines specialized AI agents, RAG-enabled knowledge bases, and lab topology management to diagnose and debug issues across GPU compute, scale-up network, and scale-out network layers.

**Key Resources:**
- [Implementation Plan](./AMD-AI-INFRA-PLAN.md)
- [Confluence Documentation](https://amd.atlassian.net/wiki/spaces/EN/pages/1540933740/)
- [AMD Pensando Platform](https://amd.atlassian.net/wiki/spaces/EN/pages/566493698/)

## Architecture Overview

### Three-Tier Infrastructure Stack

1. **GPU Compute Layer** - AMD Instinct GPUs with ROCm
2. **Scale-UP Network** - Broadcom TH6 switches (GPU-to-GPU fabric)
3. **Scale-OUT Network** - AMD Pensando AI NICs (inter-node networking)

### Plugin Components

1. **MCP Server** (provided by `ntsg-mcp-server`)
   - Unified MCP server providing 11 tools with subsystem parameter
   - KB tools (5): `ntsg_kb_search`, `ntsg_kb_get_entry`, `ntsg_kb_create_entry`, `ntsg_kb_list_entries`, `ntsg_kb_stats`
   - Lab tools (6): `ntsg_lab_search`, `ntsg_lab_get_server`, `ntsg_lab_get_card`, `ntsg_lab_get_connections`, `ntsg_lab_list`, `ntsg_lab_stats`
   - Subsystem parameter accepts: "scaleout", "scaleup", "gpu", or empty for all

2. **Knowledge Base** (via MCP server)
   - RAG-enabled knowledge bases using ChromaDB
   - Local embeddings via sentence-transformers
   - Stores debugging knowledge from Slack, Confluence, Jira
   - Location: `~/.cache/ntsg-mcp-server/kb-vectors/`

3. **Lab Topology** (via MCP server)
   - Manages lab equipment information
   - Provides node/switch/server IPs, BMC access, console connections
   - Source: `labinfo/topology.json`

4. **Agent System** (`agents/`)
   - **13 Total Agents:**
     - 9 subsystem agents (debugger, explainer, kb-updater per subsystem)
     - 4 cross-subsystem agents (cluster-orchestrator, performance-analyzer, kb-validator, docs-publisher)
   - Each uses persistent memory via Claude Code auto-memory

## Directory Structure

```
amd-ai-infra/
├── tools/
│   ├── mcp_server.py              # Unified MCP server
│   ├── gpu/                       # GPU subsystem
│   │   ├── kb/                    # GPU KB
│   │   ├── lab/                   # GPU topology
│   │   ├── rocm/                  # ROCm CLI wrappers
│   │   └── diagnostics/           # GPU diagnostics
│   ├── scaleup/                   # Scale-UP subsystem
│   │   ├── kb/                    # Switch KB
│   │   ├── lab/                   # Switch topology
│   │   ├── broadcom/              # Broadcom CLI wrappers
│   │   └── diagnostics/           # Switch diagnostics
│   ├── scaleout/                  # Scale-OUT subsystem
│   │   ├── kb/                    # NIC KB (from scaleout-debugger)
│   │   ├── lab/                   # NIC topology (from scaleout-debugger)
│   │   ├── nicctl/                # nicctl CLI wrappers
│   │   └── diagnostics/           # NIC diagnostics
│   └── common/                    # Shared frameworks
│       ├── kb_framework/          # Base KB classes
│       ├── topology_framework/    # Base topology classes
│       ├── cli_framework/         # Common CLI patterns
│       └── diagnostics_framework/ # Shared diagnostics
├── .claude/
│   ├── agents/                    # Agent definitions
│   └── agent-memory/              # Agent persistent memory
├── scripts/                       # Automation scripts
├── labinfo/                       # Lab topology data
└── docs/                          # Documentation
```

## Tool Naming Convention

```
ntsg_{component}_{action}(subsystem="...")
```

**KB Tools:**
- `ntsg_kb_search(query, subsystem, method, limit)` - Search KB (method: hybrid, keyword, semantic)
- `ntsg_kb_get_entry(entry_id)` - Get specific entry
- `ntsg_kb_create_entry(subsystem, entry_type, title, ...)` - Create new entry
- `ntsg_kb_list_entries(subsystem, entry_type, limit)` - List entries
- `ntsg_kb_stats(subsystem)` - Get KB statistics

**Lab Tools:**
- `ntsg_lab_search(query, subsystem, resource_type)` - Search servers/cards
- `ntsg_lab_get_server(server_name, subsystem)` - Get server details
- `ntsg_lab_get_card(card_name, subsystem)` - Get card info
- `ntsg_lab_get_connections(server_name)` - Get connection details
- `ntsg_lab_list(subsystem, resource_type)` - List resources
- `ntsg_lab_stats(subsystem)` - Lab statistics

**Valid Subsystems:** "scaleout", "scaleup", "gpu", or empty for all

## Common Workflows

### Debugging Across the Stack

When debugging issues:
1. **cluster-orchestrator** receives the issue
2. Determines which subsystems are involved
3. Delegates to subsystem debuggers (gpu, scaleup, scaleout)
4. Each debugger:
   - Searches its KB for similar issues
   - Runs diagnostic commands (rocm-smi, bcmcmd, nicctl)
   - Consults its explainer for architecture context
5. Cluster-orchestrator synthesizes findings
6. Performance-analyzer quantifies impact
7. Provides root cause, solution, and commands

### Knowledge Base Operations

**Search KB:**
```python
# Hybrid search (recommended)
ntsg_kb_search(query="GPU memory error", subsystem="gpu", method="hybrid")
ntsg_kb_search(query="port flapping", subsystem="scaleup", method="hybrid")
ntsg_kb_search(query="RDMA timeout", subsystem="scaleout", method="hybrid")

# Search all subsystems
ntsg_kb_search(query="training crash", subsystem="", method="hybrid")
```

**Create KB Entry:**
```python
ntsg_kb_create_entry(
    subsystem="gpu",
    entry_type="issue",
    title="Brief description",
    description="Detailed description",
    category=["category1", "category2"],
    severity="high",
    source_type="slack",
    source_id="channel-timestamp"
)
```

### Lab Topology Access

**Get Node/Switch/Server Details:**
```python
# GPU node
node = ntsg_lab_get_server(server_name="gpu-node-1", subsystem="gpu")

# Switch
switch = ntsg_lab_get_server(server_name="th6-switch-1", subsystem="scaleup")

# NIC server
server = ntsg_lab_get_server(server_name="waco1-1", subsystem="scaleout")

# Get connections for any server
connections = ntsg_lab_get_connections(server_name="waco1-1")
```

## Development Guidelines

### Code Style
- **Python:** PEP 8 compliance
- **Docstrings:** Google-style docstrings
- **Type Hints:** Use type hints for all functions
- **Error Handling:** Explicit error handling

### Testing Standards
- **Unit Tests:** ≥80% code coverage
- **Integration Tests:** All MCP tools
- **Agent Tests:** All agents invokable

### Quality Metrics
- Schema compliance: 100%
- Embedding coverage: 100%
- Source attribution: 100%
- MEMORY.md size: <200 lines per agent

## Current Development Status

**Phase:** Foundation (Week 1-2)

**Completed:**
- [x] Project structure
- [x] Scale-OUT subsystem integrated
- [x] Basic documentation

**In Progress:**
- [ ] Common framework implementation
- [ ] Unified MCP server
- [ ] GPU subsystem
- [ ] Scale-UP subsystem

**Next Steps:**
- Implement common KB framework
- Implement common topology framework
- Create base MCP server with tool routing
- Set up ChromaDB for all subsystems

## Important Notes

- **Never commit secrets** - Use environment variables
- **Test before commit** - Run unit tests
- **Document changes** - Update README and this file
- **Agent memory limits** - Keep MEMORY.md < 200 lines
- **KB entry format** - Follow schema strictly

## References

- [AMD-AI-INFRA-PLAN.md](./AMD-AI-INFRA-PLAN.md) - Complete implementation plan
- [Confluence](https://amd.atlassian.net/wiki/spaces/EN/pages/1540933740/) - Online documentation
