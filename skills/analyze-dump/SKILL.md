---
name: analyze-dump
description: Analyze .NET memory dumps (.dmp files) — crash dumps, procdump output, memory dumps. Runs native and managed diagnostics via cdb + SOS and produces a structured report. Supports multi-dump comparison for memory leak investigations.
argument-hint: [path-to-dmp-file-or-directory]
---

You are analyzing .NET memory dump file(s). Follow this multi-phase workflow carefully.

**ARGUMENTS:** `$ARGUMENTS` — path to a `.dmp` file or a directory containing one or more.

If you have any doubts on how some command or argument work, make sure to check the documentation at `https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/cdb-command-line-options`. Be sure of what you are doing, if not, make sure to double check first.

---

## Phase 1: Locate Dump(s) and Determine Mode

1. Check if `$ARGUMENTS` is a file ending in `.dmp`. If so, use it directly → **Single-Dump Mode**.
2. If it's a directory, glob for `*.dmp` inside it (non-recursively first, then recursively).
3. Based on the number of `.dmp` files found:
   - **1 file** → **Single-Dump Mode**. Set `DUMP_FILE` and `DUMP_DIR`. Proceed to Phase 2.
   - **2 files** → Ask the user: "Found 2 dumps. Compare them, or analyze one individually?"
     - If compare → **Comparison Mode**
     - If individual → ask which one → **Single-Dump Mode**
   - **3+ files** → **Comparison Mode**. Print: "Found {N} dumps. Entering comparison mode."
4. If no `.dmp` found, fail with a clear message.

### Comparison Mode Setup (when 3+ dumps)

Sort all dump files alphabetically. Set these variables:
- `ALL_DUMPS` — sorted list of all `.dmp` file paths
- `BASELINE_DUMP` — the first file whose name contains "baseline" or "_00_". If no match, use the first file alphabetically.
- `ITERATION_DUMPS` — all dumps except `BASELINE_DUMP`
- `DUMP_COUNT` — total number of dumps
- `DUMP_DIR` — directory containing the dumps (output files go here)

Print: "Baseline: {BASELINE_DUMP filename}. Iteration dumps: {DUMP_COUNT - 1}."

**Skip to Phase 2-C.**

### Single-Dump Mode Setup

Set these variables:
- `DUMP_FILE` — absolute path to the `.dmp` file
- `DUMP_DIR` — directory containing the dump (output files go here)

**Continue to Phase 2.**

---

# ═══════════════════════════════════════════════════════════════
# SINGLE-DUMP MODE (Phases 2–5)
# ═══════════════════════════════════════════════════════════════

## Phase 2: Symbol Setup

- Symbol cache directory: `$LOCALAPPDATA/Temp/SymbolCache`
- Always use `.symfix+` to add Microsoft public symbol servers
- If the dump is from a known product (e.g. Photon, Unity), note that private symbols may be needed but proceed without them

---

## Phase 3: Native Diagnostics (Pass 1 — cdb)

### CRITICAL: Never use `-c` with semicolons

**cdb interprets semicolons as symbol path separators, not command separators.** Inline commands like `-c "cmd1; cmd2"` WILL break silently. **Always write commands to a file and use `-cf <file>`.**

### Steps

1. Write a command file to `$DUMP_DIR/native_commands.txt`:

```
.symfix+ <LOCALAPPDATA>\Temp\SymbolCache
.reload
!analyze -v
vertarget
|
~
~* kb
lm
.exr -1
qq
```

Replace `<LOCALAPPDATA>` with the actual expanded path.

2. Run cdb with a **10-minute timeout** (symbol downloads can be very slow on first run):

```bash
cdb -z "$DUMP_FILE" -cf "$DUMP_DIR/native_commands.txt" > "$DUMP_DIR/native_diag.txt" 2>&1
```

Use `timeout: 600000` on the Bash tool call.

3. Verify the output file was created and is non-empty.

---

## Phase 4: Managed Diagnostics (Pass 2 — SOS)

### Ensure SOS is installed

Check if `~/.dotnet/sos/sos.dll` exists. If not:

```bash
dotnet tool install -g dotnet-sos && dotnet-sos install
```

### Steps

1. Write a command file to `$DUMP_DIR/managed_commands.txt`:

```
.symfix+ <LOCALAPPDATA>\Temp\SymbolCache
.reload
.load <HOME>\.dotnet\sos\sos.dll
!threads
~* e !clrstack
!eeheap -gc
!dumpheap -stat
!threadpool
!syncblk
!finalizequeue
!pe
qq
```

Replace `<LOCALAPPDATA>` and `<HOME>` with actual expanded paths. Use **backslashes** for `.load` paths (Windows cdb requirement).

2. Run cdb with a **10-minute timeout**:

```bash
cdb -z "$DUMP_FILE" -cf "$DUMP_DIR/managed_commands.txt" > "$DUMP_DIR/managed_diag.txt" 2>&1
```

3. If the output contains errors about SOS failing to load (e.g. "not a managed dump", CLR not found), **skip managed analysis** and note this in the final report. The dump may be native-only.

---

## Phase 5: Analysis & Report

The output files can be **100,000+ lines**. Do NOT try to read them all at once. Use this strategy:

1. **Use Grep** to locate section headers and key output markers:
   - `!analyze -v` / `EXCEPTION_RECORD` / `FAULTING_IP` — crash info
   - `vertarget` — OS/process info
   - `!threads` / `ThreadCount` — managed thread listing
   - `!clrstack` — managed call stacks
   - `!eeheap` — GC heap summary
   - `!dumpheap -stat` / `Statistics:` — heap object counts
   - `!threadpool` — thread pool stats
   - `!syncblk` — lock contention
   - `!finalizequeue` — finalizer health

2. **Read surrounding context** (use offset+limit in the Read tool) around each section header to extract the relevant data.

3. **Produce a structured report** using the report template at `templates/report-template.txt` as the exact formatting guide. Follow the template format exactly — fill in all `{{PLACEHOLDER}}` fields with data extracted from the diagnostics output. The template defines 12 sections: Environment, Exception/Crash Cause, Crashing Thread Managed Stack, Managed Exception, Thread Summary, Thread Pool, GC Heap, Top Heap Consumers, Sync Blocks, Finalizer Queue, Key Loaded Modules, and Key Findings & Recommendations. Each section includes an `ANALYSIS:` commentary block — write clear, actionable analysis for each.

4. **Write the report** to `$DUMP_DIR/full_analysis.txt`
5. **Display the Key Findings & Recommendations** section to the user inline.

---

# ═══════════════════════════════════════════════════════════════
# COMPARISON MODE (Phases 2-C through 5-C)
# ═══════════════════════════════════════════════════════════════

## Phase 2-C: Symbol Setup

Same as Phase 2. Symbol cache directory: `$LOCALAPPDATA/Temp/SymbolCache`. Use `.symfix+`.

**Important:** Symbols only need to download once. The baseline diagnostics in Phase 3-C will populate the cache; all subsequent iteration runs will reuse cached symbols and run much faster.

---

## Phase 3-C: Baseline Full Diagnostics (SERIAL — runs first)

Run three cdb passes on the **baseline dump only**, serially. This populates the symbol cache for all later parallel runs.

### CRITICAL: Never use `-c` with semicolons

**cdb interprets semicolons as symbol path separators, not command separators.** Inline commands like `-c "cmd1; cmd2"` WILL break silently. **Always write commands to a file and use `-cf <file>`.**

### Ensure SOS is installed

Check if `~/.dotnet/sos/sos.dll` exists. If not:

```bash
dotnet tool install -g dotnet-sos && dotnet-sos install
```

### Step 1: Native Diagnostics

Write `$DUMP_DIR/baseline_native_commands.txt`:

```
.symfix+ <LOCALAPPDATA>\Temp\SymbolCache
.reload
!analyze -v
vertarget
|
~
~* kb
lm
.exr -1
qq
```

Run with **10-minute timeout** (first run downloads symbols):

```bash
cdb -z "$BASELINE_DUMP" -cf "$DUMP_DIR/baseline_native_commands.txt" > "$DUMP_DIR/baseline_native_diag.txt" 2>&1
```

### Step 2: Managed Diagnostics

Write `$DUMP_DIR/baseline_managed_commands.txt`:

```
.symfix+ <LOCALAPPDATA>\Temp\SymbolCache
.reload
.load <HOME>\.dotnet\sos\sos.dll
!threads
~* e !clrstack
!eeheap -gc
!dumpheap -stat
!threadpool
!syncblk
!finalizequeue
!pe
qq
```

Run with **10-minute timeout**:

```bash
cdb -z "$BASELINE_DUMP" -cf "$DUMP_DIR/baseline_managed_commands.txt" > "$DUMP_DIR/baseline_managed_diag.txt" 2>&1
```

If SOS fails to load, note this and continue — managed comparison may not be possible.

### Step 3: Address Summary (native memory breakdown)

Write `$DUMP_DIR/address_commands.txt`:

```
.symfix+ <LOCALAPPDATA>\Temp\SymbolCache
.reload
!address -summary
qq
```

Run with **10-minute timeout**:

```bash
cdb -z "$BASELINE_DUMP" -cf "$DUMP_DIR/address_commands.txt" > "$DUMP_DIR/baseline_address.txt" 2>&1
```

---

## Phase 4-C: Parallel Iteration Diagnostics

This phase processes all iteration dumps in parallel using the **Agent tool**. The baseline is already done (Phase 3-C), so symbols are cached and each iteration run will be fast (~30-60 seconds).

### Iteration Command Set (reduced — no crash analysis needed)

Each iteration dump uses a lighter command set focused on quantitative metrics only:

```
.symfix+ <LOCALAPPDATA>\Temp\SymbolCache
.reload
.load <HOME>\.dotnet\sos\sos.dll
!eeheap -gc
!dumpheap -stat
!threads
!threadpool
!syncblk
!finalizequeue
qq
```

No `!analyze -v`, `vertarget`, `lm`, `~* kb`, `~* e !clrstack`, or `!pe` — those are only needed on the baseline.

### Parallelization Strategy

1. Split `ITERATION_DUMPS` into **batches of 3–4 dumps** each. If individual dumps are over 2 GB, reduce batch size to 2 to avoid excessive memory usage.

2. Determine **sampled dumps** for `!address -summary`: select the **midpoint** dump (index `DUMP_COUNT / 2`) and the **final** dump (last in sorted order). These will also run `!address -summary` in addition to the managed commands.

3. For each batch, launch an **Agent tool** sub-agent with a prompt like:

```
Analyze these .NET memory dumps by running cdb with SOS diagnostics.

For each dump in this batch:
1. Write a command file `$DUMP_DIR/iter_NN_commands.txt` containing:
   [the iteration command set above, with <LOCALAPPDATA> and <HOME> expanded]

2. Run cdb:
   cdb -z "<dump_path>" -cf "$DUMP_DIR/iter_NN_commands.txt" > "$DUMP_DIR/iter_NN_managed_diag.txt" 2>&1
   Use timeout: 600000

3. [If this dump is a sampled dump for !address -summary]:
   Also run: cdb -z "<dump_path>" -cf "$DUMP_DIR/address_commands.txt" > "$DUMP_DIR/iter_NN_address.txt" 2>&1

After all runs complete, extract and return these metrics for each dump:
- GC Allocated Heap Size and GC Committed Heap Size (from !eeheap -gc)
- The last ~25 lines before "Total NNN objects" from !dumpheap -stat (top heap consumers)
- The "Total" line with object count and total bytes
- ThreadCount from !threads
- CPU utilization and Workers stats from !threadpool
- SyncBlk totals (Total, CCW, RCW, Free)
- Finalizer queue: objects per heap per generation, Ready for finalization counts
- [If address summary was run]: MEM_COMMIT, Heap, PAGE_READWRITE, Image, Stack lines from !address -summary
```

4. Launch **all batch agents in a single message** (parallel tool calls) for maximum concurrency.

5. Wait for all agents to complete and collect the extracted metrics.

---

## Phase 5-C: Comparison Analysis & Report

### Step 1: Gather File Sizes

Run `ls -la $DUMP_DIR/*.dmp` to get file sizes for all dumps. Compute deltas from the baseline file size.

### Step 2: Extract Baseline Environment

From `baseline_native_diag.txt`, use Grep to find and extract:
- `vertarget` — OS version, processor count, system/process uptime
- `|` command output — process name and path
- `PROCESS_NAME` — from `!analyze -v`
- `lm` — key loaded modules

From `baseline_managed_diag.txt`, extract:
- `Number of GC Heaps` — from `!eeheap -gc` (reveals Server vs Workstation GC)

### Step 3: Extract Native Memory Trends

From `baseline_address.txt`, midpoint address file, and final address file, extract using Grep:
- `MEM_COMMIT` line (total committed memory)
- `Heap` line under "Usage Summary" (native heap size)
- `PAGE_READWRITE` line (writable pages)
- `Image` line (loaded DLLs — should be stable)
- `Stack` line (thread stacks — should be stable)
- Count of heap regions (the RgnCount next to the Heap line)

### Step 4: Build Comparison Tables

Combine the baseline metrics (from Phase 3-C) with the iteration metrics (returned by Phase 4-C agents) into the following tables:

**Section 2 — File Size Trend:**
```
  Dump        File Size        Delta from Baseline
  ─────────   ──────────────   ───────────────────
  00 (base)   XXX MB           —
  01           XXX MB           +XXX MB
  ...
```
Compute the average growth rate per iteration.

**Section 3 — Native Memory:**
```
  Category           Baseline       Midpoint        Final
  ────────────────   ────────────   ────────────    ────────────
  MEM_COMMIT          XXX MB         XXX GB          XXX GB
  Heap (native)       XXX MB         XXX GB          XXX GB
  ...
```

**Section 4 — GC Heap Trend:**
```
  Dump     GC Allocated    GC Committed    Objects     Top Consumer
  ───────  ────────────    ────────────    ─────────   ────────────
  00 base   XXX MB          XXX MB         NNN,NNN     type (size)
  01         XXX MB          XXX MB         NNN,NNN     type (size)
  ...
```
Identify patterns: linear growth (leak), sawtooth (normal cycling), stable, etc.

**Section 5 — Top Heap Consumers:**
Compare the top 5–7 types by TotalSize across baseline vs peak dump. Identify types that grow significantly.

**Section 6 — Thread Trend:**
```
  Dump     ThreadCount  Background  Dead  Workers (Total/Running/Idle)  CPU%
  ───────  ───────────  ──────────  ────  ──────────────────────────    ────
```

**Section 7 — SyncBlk & Finalizer:**
Summarize stability across all dumps. Flag any growth in SyncBlk count, RCW/CCW count, or finalizer backlog (Ready for finalization > 0).

### Step 5: Analyze and Write Findings

For each section, write an `ANALYSIS:` block that interprets the data:
- Is the native heap growing? (compare `!address -summary` across samples)
- Is the managed heap leaking or cycling? (look at GC Allocated trend)
- Are there dominant types driving growth? (compare top consumers)
- Is the finalizer keeping up? (Ready for finalization should be 0)
- Are threads accumulating? (ThreadCount trend)

For Key Findings & Recommendations (Section 9), use severity tags:
- `[CRITICAL]` — active leak or severe issue requiring immediate action
- `[WARNING]` — concerning trend that needs monitoring
- `[INFO]` — healthy behavior worth noting, or minor observations

Include specific recommended actions (UMDH tracing for native leaks, GCConserveMemory for committed memory drift, etc.)

### Step 6: Write the Report

Use the template at `templates/comparison-report-template.txt` as the formatting guide. Fill in all `{{PLACEHOLDER}}` fields with the data and analysis from Steps 1–5.

Write the report to `$DUMP_DIR/full_analysis.txt`.

### Step 7: Display Results

Display the **Key Findings & Recommendations** section (Section 9) inline to the user.

---

## Pitfall Reference

| Pitfall | Solution |
|---------|----------|
| `-c "cmd1; cmd2"` breaks in cdb | Always use `-cf <command-file>` |
| SOS not loaded by default | Explicitly `.load` with full path to `sos.dll` |
| `dotnet-sos` not installed | Install via `dotnet tool install -g dotnet-sos && dotnet-sos install` |
| Path is a directory, not a file | Always resolve to actual `.dmp` file first |
| Output files are huge | Read in chunks; use Grep to navigate to sections |
| Symbol downloads are slow | Use 10-minute timeout on cdb calls |
| `.loadby sos clr` fails | Use `.load` with absolute path instead |
| Running cdb serially on 10+ dumps is very slow | Use Agent tool to parallelize in batches of 3–4 |
| `!address -summary` missing from iteration analysis | Always run on baseline + sampled points (midpoint + final) |
| Native heap leak invisible without `!address` | Managed-only diagnostics miss native leaks; always include `!address -summary` in comparison mode |
| Trying to read 10+ diagnostic files fully | Use Grep to extract specific metrics from each file; never read more than ~50 lines at a time |
| Dumps >2 GB cause excessive memory when run in parallel | Reduce Agent batch size to 2 for large dumps |
