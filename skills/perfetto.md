---
name: perfetto-trace-processor
description: Analyze Perfetto/Android system traces using the trace_processor CLI. Use when working with .perfetto-trace, .pftrace, systrace, or ftrace files. Enables SQL-based analysis of scheduling, CPU frequency, memory, app slices, counters, and more.
---

# Perfetto Trace Processor

## Setup

```bash
# Download (Linux/Mac)
curl -LO https://get.perfetto.dev/trace_processor
chmod +x ./trace_processor
```

## Usage

```bash
# Interactive SQL shell
./trace_processor trace.perfetto-trace

# Run query file
./trace_processor trace.perfetto-trace -q query.sql

# One-shot query
./trace_processor trace.perfetto-trace -q - <<< "SELECT * FROM slice LIMIT 10"
```

## Core Tables

| Table | Description |
|-------|-------------|
| `slice` | Userspace spans (ts, dur, name, track_id, depth, parent_id) |
| `counter` | Time-series values (ts, value, track_id) |
| `sched_slice` | CPU scheduling (ts, dur, cpu, utid, end_state, priority) |
| `thread_state` | Thread states (ts, dur, state, utid, cpu) |
| `thread` | Thread info (utid, tid, name, upid) |
| `process` | Process info (upid, pid, name) |
| `track` | All tracks (id, type, name) |
| `thread_track` | Thread-associated tracks (id, utid) |
| `process_track` | Process-associated tracks (id, upid) |
| `cpu_counter_track` | CPU counter tracks (id, cpu) |
| `args` | Key-value metadata for slices/counters |

## Key Concepts

- **utid/upid**: Unique thread/process IDs (stable across tid/pid reuse)
- **track_id**: Links slices/counters to context (thread, process, cpu)
- **ts**: Timestamp in nanoseconds
- **dur**: Duration in nanoseconds

## Common Queries

### Slice Analysis
```sql
-- All slices with thread/process context
SELECT s.ts, s.dur, s.name, t.name AS thread, p.name AS process
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t USING(utid)
LEFT JOIN process p USING(upid);

-- Slowest slices
SELECT name, dur/1e6 AS dur_ms FROM slice ORDER BY dur DESC LIMIT 20;

-- Slice by name pattern
SELECT ts, dur, name FROM slice WHERE name GLOB '*binder*';
```

### Scheduling
```sql
-- CPU time per process
SELECT p.name, SUM(ss.dur)/1e6 AS cpu_ms
FROM sched_slice ss
JOIN thread t USING(utid)
JOIN process p USING(upid)
GROUP BY p.name ORDER BY cpu_ms DESC;

-- Thread states (Running, Runnable, Sleeping, etc.)
SELECT state, SUM(dur)/1e6 AS total_ms
FROM thread_state
WHERE utid = (SELECT utid FROM thread WHERE name = 'main')
GROUP BY state;
```

### Counters
```sql
-- Memory counters
SELECT ts, value, t.name
FROM counter c
JOIN process_counter_track t ON c.track_id = t.id
WHERE t.name LIKE '%mem%';

-- CPU frequency
SELECT ts, value, cpu
FROM counter c
JOIN cpu_counter_track t ON c.track_id = t.id
WHERE t.name = 'cpufreq';
```

### Trace Bounds
```sql
SELECT 
  (SELECT MIN(ts) FROM slice) AS trace_start,
  (SELECT MAX(ts + dur) FROM slice) AS trace_end;
```

## Standard Library

Import modules for pre-built views:
```sql
INCLUDE PERFETTO MODULE slices.with_context;
SELECT * FROM thread_slice;  -- slices with thread/process joined

INCLUDE PERFETTO MODULE linux.cpu.utilization.slice;
SELECT * FROM cpu_cycles_per_thread_slice;
```

## Join Patterns

```sql
-- Slice → Thread → Process
FROM slice s
JOIN thread_track tt ON s.track_id = tt.id
JOIN thread t ON tt.utid = t.utid
LEFT JOIN process p ON t.upid = p.upid

-- Counter → Track context
FROM counter c
JOIN track t ON c.track_id = t.id  -- check t.type for track kind
```

## Args Extraction

```sql
-- Get arg value for a slice
SELECT EXTRACT_ARG(arg_set_id, 'arg_name') FROM slice WHERE name = 'foo';

-- All args for a slice
SELECT key, display_value FROM args WHERE arg_set_id = <id>;
```

## Tips

- Timestamps are nanoseconds; divide by 1e6 for ms, 1e9 for seconds
- Use `GLOB` for pattern matching (case-sensitive), `LIKE` for case-insensitive
- `thread_state.state`: 'R' (Running), 'S' (Sleeping), 'D' (Uninterruptible), 'T' (Stopped)
- Filter by process: `WHERE upid = (SELECT upid FROM process WHERE name GLOB '*myapp*')`
- Perfetto UI: https://ui.perfetto.dev (drag-drop trace files)
