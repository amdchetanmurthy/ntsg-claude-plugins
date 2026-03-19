---
name: debug-nic
description: Start guided NIC debugging workflow
args:
  - name: server
    description: Server name or card name
    required: false
---

Launch the guided NIC debugging workflow for AMD Pensando AI NICs.

**Usage:**
```
/debug-nic
/debug-nic --server waco-001
```

This command will:
1. Load the `debug-nic` skill with systematic debugging workflow
2. If server specified, fetch server/card details from topology
3. Guide you through diagnostics, KB searches, and solutions
4. Help document findings in the KB

**Workflow Steps:**
1. Gather context (symptoms, affected resources)
2. Search KB for similar issues
3. Run diagnostic commands
4. Analyze results
5. Propose solutions
6. Document findings

**Tip:** Have server name and symptom description ready before starting.
