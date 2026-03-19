---
name: lab-stats
description: Show lab topology and knowledge base statistics
args:
  - name: subsystem
    description: Filter by subsystem (scaleout, scaleup, gpu)
    required: false
---

Display statistics about the lab topology and knowledge base.

**Usage:**
```
/lab-stats
/lab-stats --subsystem scaleout
```

Shows:
- **Topology Stats:**
  - Server count by subsystem
  - Card count by subsystem
  - Switch count (scaleup)
  - MTP count (connections)

- **KB Stats:**
  - Total entries
  - Breakdown by type (issue, solution, command, fact)
  - Breakdown by severity
  - Breakdown by category
  - Recent additions

This helps understand the scope of available resources and knowledge.
