# Using Superpowers Brainstorming + Planning for DB Backup Integration Scoping

A practical playbook for combining the Superpowers plugin workflow with the `db-backup-integration` skill to scope new database backup/restore integrations in `bkclientd` / `sdatamgr` before writing any code.

---

## Why This Matters

The failure mode you've probably seen: you start implementing a new DB integration (say, KingbaseES or a new Redis 7.x feature), get deep into C++ code, then discover a fundamental mismatch — the DB doesn't support streaming restore, or PITR granularity is coarser than expected, or the cluster topology breaks your assumptions. You lose days of work.

Superpowers forces a **structured front-loading** of these discoveries.

---

## The Two-Phase Approach

```
Phase 1: BRAINSTORMING (Superpowers)          Phase 2: PLANNING (Superpowers)
┌──────────────────────────────┐              ┌──────────────────────────────────┐
│ • Socratic Q&A on scope      │              │ • Micro-tasks (2-5 min each)     │
│ • DB capability survey       │    ──►       │ • Exact file paths & commands    │
│ • Gap & risk identification  │              │ • Test-first task ordering        │
│ • Integration pattern choice │              │ • Dependency graph               │
│ • Design decisions locked    │              │ • Exit criteria per task          │
└──────────────────────────────┘              └──────────────────────────────────┘
```

---

## Phase 1: Brainstorming — The Scoping Session

### How to Start

Open Claude Code in your `drbksoft/i2soft` repo and say something like:

```
I want to add backup/restore support for [DatabaseName] v[X.Y].
Let's brainstorm the integration before writing any code.
```

Superpowers will automatically activate the **brainstorming skill** — it won't jump to code. It will start asking you questions. Here's how to steer the conversation productively:

### Step 1: Feed It the DB Capability Survey

Your `db-backup-integration` skill already defines the exact survey tables (backup types, recovery capabilities, interface options, cluster awareness). **Don't answer from memory** — pull up the vendor docs and work through them with Claude:

```
Here's the YashanDB v23.4 backup doc: [paste URL or key sections]
Let's fill out the capability survey together.
```

This produces a grounded, version-specific capability matrix rather than hallucinated assumptions.

### Step 2: Drive Toward Integration Pattern Selection

The brainstorming skill will explore alternatives. Steer it toward the four patterns from your design reference:

- **Pattern A — XBSA Streaming** (YashanDB, Oracle)
- **Pattern B — CLI Wrapper** (MySQL/xtrabackup, pg_basebackup, redis-cli)
- **Pattern C — SQL Command** (DB2, some DM scenarios)
- **Pattern D — File Copy** (offline cold backup)

Ask Claude to **justify** the pattern choice against the capability survey:

```
Given the survey results, which integration pattern fits best?
What are the tradeoffs vs the other patterns?
Are there hybrid scenarios (e.g., XBSA for backup but CLI for archive log management)?
```

### Step 3: Identify Gaps and Risks Early

This is the highest-value part. Use the gap template from your requirements reference:

```
Let's do the gap analysis now. For each platform requirement 
(PITR, cross-host restore, catalog management, etc.), identify 
where this DB falls short or differs from our existing integrations.
```

Concrete examples from your past work that Superpowers would help surface:

| Gap Type | Example from Your Experience |
|---|---|
| PITR constraint | YashanDB v23.4: dropped tablespaces can't be recovered via tablespace-level ops due to `tablespace_id` metadata loss |
| Streaming semantics | XBSA `RESTORE DATABASE FROM TAG` streams directly to dbfiles — no local staging needed, but catalog must be pre-downloaded |
| Archive log behavior | `BACKUP ARCHIVELOG ALL` re-backs previously backed logs; need sequence/time-range targeting |
| Cluster limitation | YashanDB distributed clusters don't support standalone `BACKUP ARCHIVELOG` or PITR |
| Version sensitivity | Redis 6 vs 7 AOF format (single-file vs MP-AOF with manifest) |

### Step 4: Lock Design Decisions

Before exiting brainstorming, you should have clear answers to:

1. **Integration pattern** (A/B/C/D or hybrid) — with rationale
2. **PITR strategy** — unit (time/SCN/LSN/GTID), archive log management approach
3. **Cluster topology support** — which topologies in v1, which deferred
4. **Error handling approach** — what's retryable, what's fatal, what needs cleanup
5. **Cross-host restore** — path remapping strategy (mapfile? config-based?)
6. **Catalog integration** — what metadata to store per backup job

Say explicitly:

```
Let's summarize our design decisions before moving to planning.
List each decision with its rationale so I can review.
```

---

## Phase 2: Planning — The Implementation Roadmap

### Transitioning from Brainstorm to Plan

Once you approve the brainstorming output, tell Claude:

```
The brainstorming looks good. Let's write a detailed implementation plan.
Break it into tasks I can execute in 2-5 minutes each.
```

Superpowers activates the **planning skill**, which produces a structured plan document.

### What a Good Plan Looks Like for DB Integration

For a new DB backup integration, the plan should follow this task ordering:

```
1. Scaffold & Config
   ├── Task 1.1: Create [DB]BackupClient.h / .cpp stubs
   ├── Task 1.2: Create [DB]RestoreClient.h / .cpp stubs
   ├── Task 1.3: Add config schema to config.ini parser
   └── Task 1.4: Register new DB type in BackupClientFactory

2. Connection Layer
   ├── Task 2.1: Write failing test — connect to [DB] instance
   ├── Task 2.2: Implement [DB]Connection (auth, TLS, version check)
   └── Task 2.3: Green the test

3. Backup Path
   ├── Task 3.1: Write failing test — full backup produces valid output
   ├── Task 3.2: Implement startBackup() + streamData()
   ├── Task 3.3: Implement finalizeBackup() + catalog write
   ├── Task 3.4: Green the test
   └── Task 3.5: Incremental backup (if supported)

4. Restore Path
   ├── Task 4.1: Write failing test — full restore from backup
   ├── Task 4.2: Implement prepareRestore() + streamRestore()
   ├── Task 4.3: Path remapping logic
   └── Task 4.4: Green the test

5. PITR (if applicable)
   ├── Task 5.1: Write failing test — PITR to specific timestamp
   ├── Task 5.2: Archive log restore (streaming or staged)
   ├── Task 5.3: Implement applyLogs() / RECOVER
   └── Task 5.4: Green the test

6. Edge Cases & Hardening
   ├── Task 6.1: Error handling for mid-stream failure
   ├── Task 6.2: Cleanup on abort
   ├── Task 6.3: Catalog integrity on partial failure
   └── Task 6.4: Cluster-specific behavior (if applicable)

7. Integration & Docs
   ├── Task 7.1: End-to-end test against live [DB] instance
   ├── Task 7.2: Write runbook / deployment doc
   └── Task 7.3: Code review checkpoint
```

### Key Things to Demand in the Plan

Tell Claude explicitly:

```
For each task, I need:
- The exact file path to create or modify
- The test to write FIRST (red phase)
- The acceptance criteria to know when it's done
- Any DB-specific gotchas from our brainstorming gaps
```

### Plan Validation Checklist

Before approving the plan, verify:

- [ ] Every task has a test written before implementation
- [ ] Gap analysis items from brainstorming are mapped to specific tasks
- [ ] No task is larger than 5 minutes of focused work
- [ ] Dependencies between tasks are explicit (Task 4.2 depends on 3.3, etc.)
- [ ] Destructive operations (restore, cleanup) have safety checks in the plan
- [ ] Version-sensitive behavior is flagged in the relevant tasks

---

## Practical Example: Scoping a New DB (e.g., KingbaseES)

Here's a realistic session transcript outline:

```
You:   I want to add KingbaseES v8 backup/restore support.
       Let's brainstorm before coding.

Claude: [brainstorming activates]
       Let me understand the scope. A few questions first:
       1. Which KingbaseES deployment modes — standalone only, or HA?
       2. Is PITR a requirement for v1?
       3. Does KingbaseES expose an XBSA interface, or CLI only?

You:   Standalone + primary-standby for v1. PITR required.
       It's PostgreSQL-based, so pg_basebackup should work.
       But they have custom extensions — let me share the docs.

Claude: [works through capability survey]
       Based on the docs, here's what I see:
       - Full backup: pg_basebackup (streaming)
       - Incremental: not natively, but WAL archiving covers PITR
       - Pattern B (CLI wrapper) fits best because...
       
       GAP-1: KingbaseES v8 WAL naming differs from upstream PG...
       GAP-2: Their custom auth plugin requires...

You:   Good catches. Let's lock the decisions and move to planning.

Claude: [planning activates]
       Here's the implementation plan — 23 tasks across 7 phases...
```

---

## Tips for Getting the Most Out of This Workflow

1. **Don't skip brainstorming.** Even for DBs similar to ones you've already integrated (e.g., KingbaseES is PG-based), the gap analysis will catch the 20% that's different — and that 20% is where the bugs live.

2. **Bring vendor docs into the conversation.** Paste or reference specific manual sections. Claude can't hallucinate commands when it's grounded in the actual docs.

3. **Use your existing integrations as anchors.** Say things like "How does this compare to how we did it for YashanDB?" or "The Redis integration had a similar problem with AOF format versions — what's the equivalent here?"

4. **Save the brainstorming output.** The approved brainstorming summary becomes the input to your `references/requirements.md` for that DB. The planning output seeds your `references/design.md`.

5. **Review the plan against your codebase.** Before executing, do a quick `grep` or file listing to make sure the planned file paths and class names align with existing conventions in `bkclientd`.

6. **Iterate on the plan, not the code.** If something feels wrong in the plan, fix it there. Reworking a 3-line task description is infinitely cheaper than reworking 300 lines of C++.
