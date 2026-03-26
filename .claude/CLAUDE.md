# Claude Code Instructions

## Goal
This project contains different topics for system design interview preparation and actual use cases. Write short concise points, tables, cheat-sheet style so that its easy to remember.

## Design File Template

Every file in `design/` must follow this structure in order:

### 1. Concepts (if applicable)
Theory, internals, algorithms, tradeoffs — cheat-sheet style tables and bullets. Skip if covered in `concepts/`.

### 2. Functional Requirements
- Bullet list of APIs / operations the system must support
- Format: `operation(params)` → result

### 3. Non-Functional Requirements
Table with: Requirement | Target (latency numbers, availability %, consistency level, durability, scalability)

### 4. Capacity Estimation
Use **80:20 rule**: 20% of time has 80% of traffic → peak = (total × 0.8) / (0.2 × 86400)

Cover all of:
- **QPS**: avg and peak reads + writes
- **Storage**: raw data size, replication factor, overhead → total
- **IOPS**: write IOPS (WAL + storage writes per op), read IOPS (cache miss rate × reads/miss); reference: NVMe=500k random/1M seq, SATA SSD=100k, HDD=200
- **Throughput**: MB/s in and out
- **Server count**: table with role | count | sizing (cores, RAM, disk, NIC)

### 5. High-Level Architecture (HLD)
ASCII diagram showing components and data flow. One paragraph on key design choices.

### 6. Deep Dive / LLD
Cover write path and read path step by step. Include algorithms, data structures, protocols used.

### 7. Key Design Decisions
Table: Decision | Choice | Why

### 8. Failure Modes
Table: Failure | Impact | Mitigation

### 9. Interview Scenarios
Use `### "scenario description"` headers. Each scenario = a constraint change or assumption flip:
- Scale change (10× traffic)
- Consistency requirement change (eventual → strong)
- Failure scenario (node crash, region failure)
- Cost constraint (reduce storage/compute 5×)
- Correctness edge case (duplicate, ordering, hot key)

Format: bullet points with Fix / Trade-off per scenario.

## Output Token Efficiency

- Be maximally concise. Lead with the action or answer, skip all preamble.
- Do not summarize what you just did at the end of a response.
- Do not restate the user's request before acting on it.
- Skip filler phrases: "Great question", "Sure!", "Of course", "I'll now...", "Let me...", "I've successfully...".
- Prefer one sentence over three. Use bullet points only when there are genuinely multiple distinct items.
- Do not add explanations unless the user asks for them or the logic is non-obvious.
- Do not list every file you changed — the diff speaks for itself.
- No closing remarks like "Let me know if you need anything else."
- When writing code, do not add comments that restate what the code does — only comment non-obvious logic.
- Do not add docstrings, type annotations, or error handling that wasn't asked for.
