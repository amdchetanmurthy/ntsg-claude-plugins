# AMD AI Infrastructure Plugin

A comprehensive, AI-powered unified debugging and knowledge management system for AMD's complete AI cluster infrastructure stack.

## Overview

The AMD AI Infrastructure Plugin provides systematic debugging workflows for AMD AI clusters spanning three critical layers:
- **GPU Compute Layer** - AMD Instinct GPUs with ROCm
- **Scale-UP Network** - Broadcom TH6 switches (GPU-to-GPU fabric)
- **Scale-OUT Network** - AMD Pensando AI NICs (inter-node networking)

Instead of manually searching logs and running diagnostic commands, this plugin provides specialized AI agents that understand your infrastructure, search knowledge bases of known issues, and guide you through debugging with precision.

## Philosophy

Debugging distributed AI infrastructure requires more than just running commands. You need to:
- **Understand the stack** across GPU, networking, and storage layers
- **Search historical knowledge** of similar issues and solutions
- **Run targeted diagnostics** specific to the failure mode
- **Coordinate across teams** that own different components

This plugin embeds these practices into structured workflows with specialized agents that automatically orchestrate debugging across the stack.

## Quick Start

### Installation

```bash
# Clone the plugin repository
cd ~/.claude/plugins
git clone <repository-url> amd-ntsg-debug

# Install Python dependencies
cd amd-ntsg-debug
pip install -r requirements.txt

# Initialize ChromaDB vector stores for all subsystems
python -m tools.gpu.kb.cli vector-store init
python -m tools.scaleup.kb.cli vector-store init
python -m tools.scaleout.kb.cli vector-store init

# Verify installation
python -m tools.mcp_server --help
```

### Restart Claude Code

After installation, restart Claude Code to load the plugin:

```bash
# Exit your current session
exit

# Start a new session
claude
```

## Core Agents

### `cluster-orchestrator`

**Purpose**: Orchestrates debugging across GPU, network, and storage layers

**What it does:**
- Receives high-level issue descriptions
- Determines which subsystems are involved
- Delegates to specialized debuggers (GPU, scale-up, scale-out)
- Synthesizes findings from all layers
- Provides root cause analysis with specific fixes

**When to use:**
- Training job failures or slowdowns
- Distributed workload issues
- Performance degradation
- "Something is wrong but not sure where"

**Example:**
```
You: Training job on node waco1-1 is running at 50% expected speed
Cluster-orchestrator will:
1. Check GPU utilization via gpu-debugger
2. Check switch congestion via scaleup-debugger
3. Check NIC RDMA stats via scaleout-debugger
4. Identify bottleneck (e.g., RDMA timeout causing retransmits)
5. Provide fix: "Check ROCE settings on NICs, tune flow control"
```

### `gpu-debugger`

**Purpose**: Diagnoses AMD GPU and ROCm issues

**Focus areas:**
- GPU memory errors (ECC, HBM)
- ROCm driver issues
- GPU utilization and performance
- VRAM allocation problems
- GPU topology and PCIe connectivity

**Knowledge Base Integration:**
- Searches GPU KB for similar issues
- References ROCm documentation
- Suggests known workarounds

**Diagnostic Commands:**
- `rocm-smi` - GPU status and utilization
- `rocminfo` - GPU topology and capabilities
- `radeontop` - Real-time GPU monitoring
- `dmesg` - Kernel messages for GPU errors

**Example output:**
```
Issue: GPU memory allocation failure on gpu-node-5

Analysis:
- GPU 0 has 12 ECC errors in last 24h (rocm-smi --showerrors)
- HBM temperature at 95°C (thermal throttling threshold)
- PCIe link trained at Gen3 x8 (should be Gen4 x16)

KB Search Results:
- #42: Similar ECC pattern caused by memory voltage drift
- #67: HBM overheating due to fan failure
- #103: PCIe link degradation after firmware update

Root Cause: Fan failure causing HBM overheating + ECC errors

Fix:
1. Check fan status: rocm-smi --showfan
2. If failed, replace fan assembly
3. Monitor temps: rocm-smi --showtemp --interval 1
4. Once cooled, reset GPU: rocm-smi --gpureset -d 0
```

### `scaleup-debugger`

**Purpose**: Diagnoses Broadcom TH6 switch fabric issues

**Focus areas:**
- Port flapping and link errors
- Switch congestion and buffer drops
- ECMP load balancing issues
- Multicast distribution problems
- Switch firmware and configuration

**Knowledge Base Integration:**
- Searches switch KB for port errors
- References Broadcom TH6 documentation
- Suggests known configuration fixes

**Diagnostic Commands:**
- `bcmcmd` - Broadcom switch CLI
- `show port` - Port status
- `show counters` - Traffic statistics
- `show qos` - Queue depths and drops

**Example output:**
```
Issue: High latency on traffic between racks

Analysis:
- Port 1/25 showing 2.3% packet drops (bcmcmd "show counters")
- Queue 7 depth at 100% (congestion on priority traffic)
- ECMP hash distribution skewed (80% on 2/4 paths)

KB Search Results:
- #28: Queue drops on priority traffic due to misconfigured QoS
- #55: ECMP imbalance caused by hash seed mismatch
- #91: TH6 buffer tuning for AI workloads

Root Cause: QoS misconfiguration + ECMP hash imbalance

Fix:
1. Reconfigure QoS: bcmcmd "config add qos_map_priority"
2. Update ECMP hash seed: bcmcmd "config set ecmp_hash_seed 0x1234"
3. Monitor queue depths: bcmcmd "show qos" (verify queue 7 drains)
```

### `scaleout-debugger`

**Purpose**: Diagnoses AMD Pensando NIC and RDMA issues

**Focus areas:**
- RDMA timeout and retransmits
- NIC firmware and driver issues
- Network congestion and PFC storms
- SR-IOV configuration problems
- NIC-to-NIC connectivity

**Knowledge Base Integration:**
- Searches NIC KB for RDMA issues
- References Pensando documentation
- Suggests known driver/firmware fixes

**Diagnostic Commands:**
- `nicctl` - Pensando NIC management
- `nicctl status` - NIC health
- `nicctl stats` - Traffic and error counters
- `rdma_perf_test` - RDMA performance testing

**Example output:**
```
Issue: RDMA timeout errors causing training job restarts

Analysis:
- NIC on waco1-1 showing 450 RDMA timeouts/min (nicctl stats)
- PFC pause frames received: 12.3M (flow control storm)
- Firmware version: 1.2.3 (known issue with ROCE)
- TX queue 5 stuck (nicctl status shows queue hang)

KB Search Results:
- #15: RDMA timeouts due to PFC storm on congested fabric
- #34: NIC firmware 1.2.3 has ROCE bug (upgrade to 1.2.5)
- #78: TX queue hang requires NIC reset

Root Cause: Firmware bug + PFC storm

Fix:
1. Upgrade firmware: nicctl firmware update --version 1.2.5
2. Reset NIC: nicctl reset --device pci-0000:17:00.0
3. Tune ROCE parameters: nicctl roce set-timeout 100ms
4. Monitor RDMA stats: nicctl stats --interval 1 (verify timeouts stop)
```

### Subsystem Explainers

Each subsystem has an **explainer** agent for architectural questions:

#### `gpu-explainer`
- Explains ROCm architecture, GPU topology, memory hierarchy
- References AMD GPU documentation
- Clarifies GPU-specific concepts

#### `scaleup-explainer`
- Explains Broadcom TH6 architecture, switching fabric, QoS
- References Broadcom documentation
- Clarifies network fabric concepts

#### `scaleout-explainer`
- Explains Pensando NIC architecture, RDMA, SR-IOV
- References Pensando documentation
- Clarifies NIC and RDMA concepts

**Example:**
```
You: How does GPU memory allocation work in ROCm?
gpu-explainer will:
- Explain HBM architecture and VRAM management
- Describe ROCm memory allocator
- Reference documentation for hip_malloc
- Show memory topology with rocminfo
```

### Knowledge Base Updaters

Each subsystem has a **kb-updater** agent that maintains the knowledge base:

#### `gpu-kb-updater`
- Monitors GPU-related Slack channels
- Ingests Confluence documentation
- Creates KB entries for new issues
- Updates existing entries with new solutions

#### `scaleup-kb-updater`
- Monitors switch-related Slack channels
- Ingests switch documentation
- Creates KB entries for switch issues

#### `scaleout-kb-updater`
- Monitors NIC-related Slack channels
- Ingests NIC documentation
- Creates KB entries for RDMA issues

**Usage:**
```
"Update GPU KB with recent memory errors from #gpu-support"
"Sync scale-out KB from Pensando confluence"
```

### Cross-Subsystem Agents

#### `performance-analyzer`

**Purpose**: End-to-end performance analysis across all layers

**Focus areas:**
- Training job performance
- Distributed workload efficiency
- Bottleneck identification
- Performance regression analysis

**Example:**
```
You: Why is my 64-GPU training job only reaching 70% efficiency?
performance-analyzer will:
1. Check GPU utilization (should be 95%+)
2. Check switch fabric utilization (look for congestion)
3. Check NIC RDMA throughput (compare to baseline)
4. Check storage I/O (rule out data loading bottleneck)
5. Identify bottleneck: "Switch congestion on rack-to-rack links"
6. Quantify impact: "30% efficiency loss = $X/hour"
7. Provide fix with expected improvement
```

#### `kb-validator`

**Purpose**: Quality assurance for knowledge base entries

**Focus areas:**
- Schema compliance validation
- Duplicate entry detection
- Missing metadata identification
- Broken reference checking

**Usage:**
```
"Validate all GPU KB entries"
"Check for duplicate NIC KB entries"
```

#### `docs-publisher`

**Purpose**: Maintains Confluence documentation automatically

**Focus areas:**
- Publishes debugging guides from KB entries
- Updates architecture diagrams
- Generates troubleshooting flowcharts
- Maintains change logs

**Usage:**
```
"Publish top 20 GPU issues to Confluence"
"Update NIC debugging guide"
```

## MCP Tools (51 Total)

The plugin provides 51 MCP tools accessible via the unified MCP server:

### GPU Tools (17)

**Knowledge Base (9):**
- `amd_gpu_kb_search_semantic` - Semantic search using embeddings
- `amd_gpu_kb_search_keyword` - Keyword search
- `amd_gpu_kb_search_hybrid` - Combined semantic + keyword (recommended)
- `amd_gpu_kb_create_entry` - Create new KB entry
- `amd_gpu_kb_get_entry` - Retrieve entry by ID
- `amd_gpu_kb_update_entry` - Update existing entry
- `amd_gpu_kb_delete_entry` - Delete entry
- `amd_gpu_kb_get_statistics` - KB statistics
- `amd_gpu_kb_list_categories` - List all categories

**Lab Topology (8):**
- `amd_gpu_lab_get_node` - Get GPU node details
- `amd_gpu_lab_search_nodes` - Search nodes by criteria
- `amd_gpu_lab_get_gpu` - Get GPU device info
- `amd_gpu_lab_search_gpus` - Search GPUs by criteria
- `amd_gpu_lab_get_connection` - Get node connections
- `amd_gpu_lab_search_connections` - Search connections
- `amd_gpu_lab_get_stats` - Lab statistics
- `amd_gpu_lab_list_nodes` - List all nodes

### Scale-UP Tools (17)

**Knowledge Base (9):**
- `amd_scaleup_kb_search_semantic`
- `amd_scaleup_kb_search_keyword`
- `amd_scaleup_kb_search_hybrid`
- `amd_scaleup_kb_create_entry`
- `amd_scaleup_kb_get_entry`
- `amd_scaleup_kb_update_entry`
- `amd_scaleup_kb_delete_entry`
- `amd_scaleup_kb_get_statistics`
- `amd_scaleup_kb_list_categories`

**Lab Topology (8):**
- `amd_scaleup_lab_get_switch` - Get switch details
- `amd_scaleup_lab_search_switches` - Search switches
- `amd_scaleup_lab_get_port` - Get port info
- `amd_scaleup_lab_search_ports` - Search ports
- `amd_scaleup_lab_get_connection` - Get switch connections
- `amd_scaleup_lab_search_connections` - Search connections
- `amd_scaleup_lab_get_stats` - Lab statistics
- `amd_scaleup_lab_list_switches` - List all switches

### Scale-OUT Tools (17)

**Knowledge Base (9):**
- `amd_scaleout_kb_search_semantic`
- `amd_scaleout_kb_search_keyword`
- `amd_scaleout_kb_search_hybrid`
- `amd_scaleout_kb_create_entry`
- `amd_scaleout_kb_get_entry`
- `amd_scaleout_kb_update_entry`
- `amd_scaleout_kb_delete_entry`
- `amd_scaleout_kb_get_statistics`
- `amd_scaleout_kb_list_categories`

**Lab Topology (8):**
- `amd_scaleout_lab_get_server` - Get server details
- `amd_scaleout_lab_search_servers` - Search servers
- `amd_scaleout_lab_get_nic` - Get NIC info
- `amd_scaleout_lab_search_nics` - Search NICs
- `amd_scaleout_lab_get_connection` - Get server connections
- `amd_scaleout_lab_search_connections` - Search connections
- `amd_scaleout_lab_get_stats` - Lab statistics
- `amd_scaleout_lab_list_servers` - List all servers

## Common Debugging Workflows

### Training Job Failure

**Symptom:** Training job crashed or hung

**Workflow:**
1. Invoke `cluster-orchestrator` with error details
2. Orchestrator checks:
   - GPU status (memory errors, crashes)
   - Network connectivity (RDMA timeouts)
   - Switch health (port errors, congestion)
3. Identifies root cause
4. Searches relevant KB for solutions
5. Provides fix with specific commands

### Performance Degradation

**Symptom:** Training slower than expected

**Workflow:**
1. Invoke `performance-analyzer` with baseline metrics
2. Analyzer checks:
   - GPU utilization (should be 95%+)
   - Network throughput (compare to baseline)
   - Switch congestion (buffer drops)
   - Storage I/O (data loading bottleneck)
3. Identifies bottleneck layer
4. Quantifies performance loss
5. Provides optimization recommendations

### Recurring Issues

**Symptom:** Same issue keeps appearing

**Workflow:**
1. Search KB: `amd_gpu_kb_search_hybrid(query="GPU memory error pattern")`
2. If found: Follow documented solution
3. If not found: Create new KB entry with `amd_gpu_kb_create_entry()`
4. Share with team via `docs-publisher`

### Infrastructure Questions

**Symptom:** Need to understand architecture or configuration

**Workflow:**
1. Invoke appropriate explainer:
   - `gpu-explainer` for GPU/ROCm questions
   - `scaleup-explainer` for switch/fabric questions
   - `scaleout-explainer` for NIC/RDMA questions
2. Explainer provides:
   - Architectural explanation
   - Documentation references
   - Example commands
   - Visual diagrams (via topology tools)

## Usage Patterns

### Full orchestrated debugging (recommended for complex issues):

```
You: "Training job failed on waco1-1 with NCCL timeout"
```

The `cluster-orchestrator` will automatically:
1. Check GPU health
2. Check RDMA connectivity
3. Check switch fabric
4. Search all KB subsystems
5. Provide comprehensive root cause analysis

### Subsystem-specific debugging:

**GPU issue:**
```
You: "Launch gpu-debugger to investigate memory errors on node gpu-5"
```

**Network issue:**
```
You: "Launch scaleout-debugger to check RDMA timeouts on waco1-1"
```

**Switch issue:**
```
You: "Launch scaleup-debugger to analyze port flapping on th6-switch-1"
```

### Direct KB search:

**Search for known issues:**
```python
# Via MCP tools
amd_gpu_kb_search_hybrid(query="ECC memory error", limit=10)
amd_scaleout_kb_search_hybrid(query="RDMA timeout", limit=10)
```

### Lab topology queries:

**Get equipment details:**
```python
# Get GPU node
amd_gpu_lab_get_node(node_name="gpu-node-5")

# Get switch info
amd_scaleup_lab_get_switch(switch_name="th6-switch-1")

# Get NIC server
amd_scaleout_lab_get_server(server_name="waco1-1")
```

## Knowledge Base

Each subsystem maintains a RAG-enabled knowledge base using ChromaDB:

### KB Entry Types

- **Issue** - Known problems and solutions
- **Guide** - Debugging procedures and workflows
- **Reference** - Documentation and architecture notes
- **FAQ** - Common questions and answers

### KB Sources

- **Slack** - Debugging discussions from team channels
- **Confluence** - Documentation pages
- **Jira** - Issue tickets with resolutions
- **Manual** - Hand-curated entries

### Search Modes

**Semantic Search:**
- Uses sentence-transformers embeddings
- Finds conceptually similar entries
- Best for: "Find issues like this one"

**Keyword Search:**
- Full-text search on title/description
- Exact match on terms
- Best for: "Find entries mentioning 'ECC error'"

**Hybrid Search (Recommended):**
- Combines semantic + keyword
- Ranks results by relevance
- Best for: Most debugging scenarios

### KB Storage

Knowledge bases are stored in ChromaDB collections:
- GPU KB: `~/.cache/amd-ntsg-debug/gpu-kb-vectors/`
- Scale-UP KB: `~/.cache/amd-ntsg-debug/scaleup-kb-vectors/`
- Scale-OUT KB: `~/.cache/amd-ntsg-debug/scaleout-kb-vectors/`

## Lab Topology

### Topology Data Sources

Each subsystem has a JSON file in `labinfo/`:
- `gpu_topology.json` - GPU nodes, PCIe connections
- `scaleup_topology.json` - Switches, ports, fabric links
- `scaleout_topology.json` - Servers, NICs, network connections

### Topology Information

**GPU Nodes:**
- Node name, IP address, BMC access
- GPU devices per node
- PCIe topology
- Console connection details

**Switches:**
- Switch name, management IP
- Port mappings
- Uplink/downlink connections
- SNMP community strings

**Servers:**
- Server name, IP address, BMC access
- NIC devices per server
- Network interface mappings
- Console access

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│              Claude Code User Interface                 │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────┴──────────────────────────────────┐
│              Cluster Orchestrator Agent                  │
│  (Cross-stack debugging coordinator)                     │
└──────┬───────────────┬────────────────┬─────────────────┘
       │               │                │
   ┌───┴───┐      ┌────┴────┐     ┌────┴─────┐
   │  GPU  │      │ Scale-UP│     │Scale-OUT │
   │Debugger     │Debugger │     │Debugger  │
   └───┬───┘      └────┬────┘     └────┬─────┘
       │               │                │
   ┌───┴──────┐   ┌────┴────┐     ┌────┴────┐
   │ GPU KB   │   │Switch KB│     │ NIC KB  │
   │(ChromaDB)│   │(ChromaDB)     │(ChromaDB)│
   └───┬──────┘   └────┬────┘     └────┬────┘
       │               │                │
   ┌───┴──────┐   ┌────┴────┐     ┌────┴────┐
   │GPU Nodes │   │ TH6     │     │Pensando │
   │(ROCm)    │   │Switches │     │AI NICs  │
   └──────────┘   └─────────┘     └─────────┘
```

## Best Practices

1. **Use cluster-orchestrator for complex issues**: Let it determine which subsystems to check
2. **Search KB before deep debugging**: Someone may have solved it already
3. **Create KB entries for new issues**: Help future debugging efforts
4. **Use hybrid search**: Combines semantic understanding with keyword matching
5. **Keep agent memory under 200 lines**: Agents have persistent memory - keep it concise
6. **Document in Confluence**: Use docs-publisher to share knowledge widely

## When to Use This Plugin

**Use for:**
- Distributed training job failures
- Performance degradation across the stack
- Recurring infrastructure issues
- Understanding complex architecture
- Building institutional knowledge

**Don't use for:**
- Application code bugs (use regular debugging)
- Non-AMD infrastructure
- Issues outside GPU/network scope
- Secrets management (use secure storage)

## Requirements

- Claude Code installed
- Python 3.10+
- AMD GPU nodes with ROCm (for GPU debugging)
- Broadcom TH6 switches (for scale-up debugging)
- AMD Pensando NICs (for scale-out debugging)
- Access to lab equipment (SSH, BMC, console)

## Technology Stack

- **Language:** Python 3.10+
- **MCP SDK:** mcp ≥1.0.0
- **Vector DB:** ChromaDB ≥0.4.22
- **Embeddings:** sentence-transformers ≥2.3.1 (all-MiniLM-L6-v2)
- **ML Framework:** PyTorch ≥2.0.0 (for embeddings)
- **Shell:** Bash 4.0+

## Project Structure

```
amd-ntsg-debug/
├── .claude-plugin/
│   └── plugin.json          # Plugin metadata
├── agents/                  # Agent definitions (to be created)
│   ├── cluster-orchestrator.md       # (planned)
│   ├── gpu-debugger.md               # (planned)
│   ├── gpu-explainer.md              # (planned)
│   ├── gpu-kb-updater.md             # (planned)
│   ├── scaleup-debugger.md           # (planned)
│   ├── scaleup-explainer.md          # (planned)
│   ├── scaleup-kb-updater.md         # (planned)
│   ├── scaleout-debugger.md          # (planned)
│   ├── scaleout-explainer.md         # (planned)
│   ├── scaleout-kb-updater.md        # (planned)
│   ├── performance-analyzer.md       # (planned)
│   ├── kb-validator.md               # (planned)
│   └── docs-publisher.md             # (planned)
├── .claude/                 # Claude runtime directories
│   └── agent-memory/        # Agent persistent memory (auto-created)
│       ├── cluster-orchestrator/
│       ├── gpu-debugger/
│       └── ...
├── tools/
│   ├── mcp_server.py        # Unified MCP server (51 tools)
│   ├── gpu/                 # GPU subsystem (planned)
│   │   ├── kb/              # GPU knowledge base
│   │   ├── lab/             # GPU lab topology
│   │   ├── rocm/            # ROCm CLI wrappers
│   │   └── diagnostics/     # GPU diagnostics
│   ├── scaleup/             # Scale-UP network subsystem (planned)
│   │   ├── kb/              # Switch knowledge base
│   │   ├── lab/             # Switch lab topology
│   │   ├── broadcom/        # Broadcom CLI wrappers
│   │   └── diagnostics/     # Switch diagnostics
│   ├── scaleout/            # Scale-OUT network subsystem (working)
│   │   ├── kb/              # NIC knowledge base (implemented)
│   │   ├── lab/             # NIC lab topology (implemented)
│   │   ├── nicctl/          # Pensando CLI wrappers (planned)
│   │   └── diagnostics/     # NIC diagnostics (planned)
│   └── common/              # Shared frameworks (planned)
│       ├── kb_framework/    # Base KB classes
│       ├── topology_framework/  # Base topology classes
│       ├── cli_framework/   # Common CLI patterns
│       └── diagnostics_framework/  # Shared diagnostics
├── labinfo/                 # Lab topology data
│   ├── gpu_topology.json           # (to be created)
│   ├── scaleup_topology.json       # (to be created)
│   └── scaleout_topology.json      # (exists)
├── scripts/                 # Automation scripts (to be created)
├── docs/                    # Documentation
├── tests/                   # Test suites
├── .mcp.json                # MCP server configuration
├── requirements.txt         # Python dependencies
├── CLAUDE.md                # Development guidelines
├── AMD-AI-INFRA-PLAN.md     # Implementation plan
└── README.md                # This file
```

## Development Status

**Current Phase:** Foundation (Week 1-2)

**Completed:**
- [x] Project structure created
- [x] Scale-OUT subsystem integrated
- [x] MCP server skeleton with 51 tool definitions
- [x] Basic documentation

**In Progress:**
- [ ] Common framework implementation
- [ ] GPU subsystem implementation
- [ ] Scale-UP subsystem implementation
- [ ] Agent definitions (13 agents)
- [ ] Cross-subsystem integration

**Planned:**
- [ ] Slack monitoring integration
- [ ] Automated KB updates
- [ ] Confluence documentation publishing
- [ ] Performance benchmarking
- [ ] Production deployment

## Troubleshooting

### MCP Server Won't Start

**Issue:** `python -m tools.mcp_server` fails

**Solutions:**
- Check Python version: `python --version` (need 3.10+)
- Install dependencies: `pip install -r requirements.txt`
- Check PYTHONPATH: Should include plugin directory

### ChromaDB Initialization Fails

**Issue:** Vector store init fails

**Solutions:**
- Check disk space (ChromaDB needs ~1GB per subsystem)
- Check permissions on `~/.cache/amd-ntsg-debug/`
- Reinstall ChromaDB: `pip install --upgrade chromadb`

### Agents Can't Find Equipment

**Issue:** Lab topology queries return empty

**Solutions:**
- Check topology JSON files exist in `labinfo/`
- Verify JSON format is valid: `python -m json.tool labinfo/gpu_topology.json`
- Update topology data if lab configuration changed

### KB Search Returns No Results

**Issue:** Knowledge base searches return nothing

**Solutions:**
- Check if KB is initialized: `amd_gpu_kb_get_statistics()`
- Populate KB with initial entries
- Verify embeddings are generated (can take time on first run)

## Tips

- **Start with cluster-orchestrator**: Let it determine the debugging path
- **Use hybrid search**: Best balance of semantic and keyword matching
- **Check KB first**: Save time by finding known solutions
- **Create KB entries**: Document new issues for future reference
- **Use explainer agents**: Understand architecture before debugging
- **Monitor agent memory**: Keep MEMORY.md files under 200 lines
- **Update topology data**: Keep lab information current

## Documentation

- [Implementation Plan](./AMD-AI-INFRA-PLAN.md) - Complete 12-week implementation plan
- [CLAUDE.md](./CLAUDE.md) - Development guidelines
- [Confluence](https://amd.atlassian.net/wiki/spaces/EN/pages/1540933740/) - Online documentation

## Support

- **Internal Team:** AMD NTSG (Network, Transport, Security Group)
- **Documentation:** See [AMD-AI-INFRA-PLAN.md](./AMD-AI-INFRA-PLAN.md)
- **Confluence:** [AMD AI Infrastructure MCP Plugin](https://amd.atlassian.net/wiki/spaces/EN/pages/1540933740/)

## License

Proprietary - AMD Internal Use Only

## Version

**Version:** 0.1.0
**Last Updated:** 2026-03-17
**Status:** In Development - Phase 1 (Foundation)

---

**Ready to debug smarter, not harder.** 🚀
