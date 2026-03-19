---
name: debug-nic
description: Guided NIC debugging workflow for AMD Pensando AI NICs
---

# NIC Debugging Workflow

You are helping debug AMD Pensando AI NIC issues. Follow this systematic workflow:

## 1. Gather Context

**Always start by asking:**
- What symptoms are you seeing? (e.g., link down, RDMA errors, performance issues)
- Which server/card is affected?
- When did the issue start?
- Any recent changes? (firmware, driver, network configuration)

## 2. Identify Resources

Use `ntsg_lab_search` to find the server/card:
```
ntsg_lab_search(query="server-name", subsystem="scaleout")
```

Then get full details:
```
ntsg_lab_get_server(server_name="...", subsystem="scaleout")
ntsg_lab_get_card(card_name="...", subsystem="scaleout")
```

Get connection info (SSH, console, PDU):
```
ntsg_lab_get_connections(server_name="...")
```

## 3. Search Knowledge Base

Based on symptoms, search the KB:
```
ntsg_kb_search(
    query="<symptom keywords>",
    subsystem="scaleout",
    method="hybrid"  # or "semantic" for conceptual search
)
```

Check for similar issues and known solutions.

## 4. Run Diagnostics

Instruct the user to run diagnostic commands (or run via SSH if credentials available):

**Health Checks:**
```bash
nicctl show card heartbeat
nicctl show card coredump
nicctl show environment
```

**Port Status:**
```bash
nicctl show port --brief
nicctl show port fsm
nicctl show port statistics
```

**RDMA/QoS:**
```bash
nicctl show qos pfc --detail
nicctl show dcqcn --roce-device <dev>
nicctl show lif statistics
```

**Debug Bundle (for escalation):**
```bash
nicctl show techsupport -c <card-uuid> -o /tmp/debug-bundle
```

## 5. Analyze Results

- Compare output against known good states
- Look for error patterns matching KB entries
- Check for:
  - Link state mismatches
  - High error/drop counters
  - PFC/DCQCN anomalies
  - Coredumps or heartbeat failures

## 6. Propose Solutions

Based on KB matches and diagnostic results:
1. Start with least disruptive solutions
2. Document each step before executing
3. Verify fix after each change

**Common Solutions:**
- Port flap: `nicctl port disable <port>; sleep 2; nicctl port enable <port>`
- Clear stats: `nicctl clear port statistics`
- Firmware reload: `nicctl card reload` (coordinate downtime!)
- Driver reload: `rmmod ionic; modprobe ionic` (disruptive!)

## 7. Document & Update KB

If this is a new issue or solution:
```
ntsg_kb_create_entry(
    subsystem="scaleout",
    type="issue",
    title="<brief description>",
    description="<detailed description>",
    severity="<critical|high|medium|low>",
    category=["nic", "rdma", ...],
    symptoms=[...],
    solutions=[...],
    commands=[...]
)
```

## Tips

- Always get user consent before running destructive commands
- Use `ntsg_kb_search` with different methods (semantic vs hybrid) if first search doesn't find relevant results
- Check both server and card details - issues can be at either level
- For cross-subsystem issues (GPU + NIC + Switch), invoke the `cluster-orchestrator` agent
- Document your debugging session so others can learn from it

## Error Handling

If you can't find the resource:
- Try partial name search: `ntsg_lab_search(query="partial-name")`
- List all resources: `ntsg_lab_list(subsystem="scaleout", resource_type="servers")`

If KB search returns no results:
- Try broader keywords
- Use semantic search for conceptual matching
- Check if this is a new issue (needs KB entry)

## Escalation

If standard debugging doesn't resolve:
1. Collect techsupport bundle
2. Check for related JIRA issues in KB entries
3. Document findings and propose escalation with evidence
