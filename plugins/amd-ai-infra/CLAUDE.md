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

1. **MCP Server Layer** (`tools/mcp_server.py`)
   - Unified MCP server providing 51 tools
   - GPU tools (17): `amd_gpu_*`
   - Scale-UP tools (17): `amd_scaleup_*`
   - Scale-OUT tools (17): `amd_scaleout_*`

2. **Knowledge Base Subsystems** (`tools/{gpu,scaleup,scaleout}/kb/`)
   - RAG-enabled knowledge bases using ChromaDB
   - Local embeddings via sentence-transformers
   - Stores debugging knowledge from Slack, Confluence, Jira
   - Location: `~/.cache/amd-ai-infra/{subsystem}-kb-vectors/`

3. **Lab Topology Subsystems** (`tools/{gpu,scaleup,scaleout}/lab/`)
   - Manages lab equipment information
   - Provides node/switch/server IPs, BMC access, console connections
   - Source: `labinfo/{gpu,scaleup,scaleout}_topology.json`

4. **Agent System** (`.claude/agents/`)
   - **13 Total Agents:**
     - 9 subsystem agents (debugger, explainer, kb-updater per subsystem)
     - 4 cross-subsystem agents (cluster-orchestrator, performance-analyzer, kb-validator, docs-publisher)
   - Each has persistent memory in `.claude/agent-memory/<agent>/`

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
amd_{subsystem}_{component}_{action}
```

Examples:
- `amd_gpu_kb_search_hybrid`
- `amd_scaleup_lab_get_switch`
- `amd_scaleout_lab_get_server`

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
amd_gpu_kb_search_hybrid(query="GPU memory error", limit=10)
amd_scaleup_kb_search_hybrid(query="port flapping", limit=10)
amd_scaleout_kb_search_hybrid(query="RDMA timeout", limit=10)
```

**Create KB Entry:**
```python
amd_{subsystem}_kb_create_entry(
    type="issue",
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
node = amd_gpu_lab_get_node(node_name="gpu-node-1")

# Switch
switch = amd_scaleup_lab_get_switch(switch_name="th6-switch-1")

# NIC server
server = amd_scaleout_lab_get_server(server_name="waco1-1")
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
