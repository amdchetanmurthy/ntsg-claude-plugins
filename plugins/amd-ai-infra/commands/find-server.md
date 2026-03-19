---
name: find-server
description: Find servers or cards in the lab topology
args:
  - name: query
    description: Server or card name (full or partial)
    required: true
  - name: subsystem
    description: Filter by subsystem (scaleout, scaleup, gpu)
    required: false
---

Search the lab topology for servers and cards.

**Usage:**
```
/find-server waco-001
/find-server waco --subsystem scaleout
/find-server GPU-01
```

The command will show:
- Server/card details (hostname, IPs, management info)
- Subsystem associations
- Connection info (SSH, console, PDU)
- Associated cards (for servers) or server (for cards)

**Tip:** Use partial names to find multiple matches, e.g., `/find-server waco` will find all Waco servers.
