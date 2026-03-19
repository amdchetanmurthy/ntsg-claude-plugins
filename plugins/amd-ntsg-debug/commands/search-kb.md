---
name: search-kb
description: Search the knowledge base for issues, solutions, and documentation
args:
  - name: query
    description: Search query (keywords or natural language question)
    required: true
  - name: subsystem
    description: Filter by subsystem (scaleout, scaleup, gpu)
    required: false
  - name: method
    description: Search method (hybrid, semantic, keyword)
    required: false
    default: hybrid
---

Search the AMD AI infrastructure knowledge base for issues, solutions, commands, and facts.

**Usage:**
```
/search-kb "RDMA timeout errors"
/search-kb "port flapping" --subsystem scaleout
/search-kb "how to reset NIC" --method semantic
```

**Search Methods:**
- `hybrid` (default) - Combines semantic + keyword for best results
- `semantic` - Conceptual/meaning-based search (best for "how to" queries)
- `keyword` - Exact keyword matching

**Subsystems:**
- `scaleout` - Pensando AI NICs (RDMA, networking)
- `scaleup` - Broadcom TH6 switches (GPU fabric)
- `gpu` - AMD Instinct GPUs (ROCm, compute)
- (empty) - Search across all subsystems

The command will display matching KB entries with:
- Entry ID and title
- Severity and category
- Symptoms and solutions
- Related commands and JIRA refs
