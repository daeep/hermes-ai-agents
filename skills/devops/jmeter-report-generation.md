---
name: jmeter-report-generation
description: "Use when generating reports from JMeter .jtl/.csv results — HTML Dashboard, CMDRunner plugins (GraphPlotter, SynthesisReport, FilterResults), CLI-based CSV analysis for percentiles/throughput/errors, dual-generator merge with 3-column correction (allThreads+grpThreads+threadName for correct concurrent user counts), and split-by-execution workflows. Covers macOS homebrew install paths and common pitfalls."
version: 1.2.0
author: Antonio
license: MIT
metadata:
  hermes:
    tags: [jmeter, performance, reporting, testing, load-testing, multi-server, dual-generator]
    related_skills: [plan]
---

# JMeter Report Generation

## Overview

Apache JMeter produces raw result files (`.jtl` / `.csv`) that need post-processing into actionable reports. This skill covers three approaches:

1. **Built-in HTML Dashboard** — quick visual report via `jmeter -g`
2. **JMeter Plugins (CMDRunner)** — custom graphs (Response Times Over Time, Active Threads, etc.), synthesis reports, and result filtering via [jmeter-plugins.org](https://jmeter-plugins.org/) Command-Line Graph Plotting Tool
3. **CSV/JTL Analysis** — direct CLI tools (awk, jq, Python) for percentiles, throughput, error rates, and custom metrics

**Host system:** macOS (JMeter installed via Homebrew at `/opt/homebrew/bin/jmeter`). JMeter ships with no plugins — the Dashboard is built-in but CMDRunner and plugins must be installed separately.

## When to Use

- User completed a JMeter test and wants an HTML report
- User needs specific graphs (response times over time, TPS, active threads)
- User wants numerical percentiles (p50, p90, p95, p99) from a .jtl
- User needs to filter or aggregate results after a test run
- User asks "genera reporte de JMeter" or "saca las graficas de JMeter"

## Prerequisites

- JMeter installed (`which jmeter` or homebrew: `brew list jmeter`)
- A `.jtl` or `.csv` result file from a previous test run
- For CMDRunner: `~/.hermes/lib/jmeter-plugins/CMDRunner.jar` + `jmeter-plugins-manager` installed (see section below)

## 1. Built-in HTML Dashboard

The fastest way to get a visual report. Works on any aggregate `.jtl` file.

### Generate Dashboard

```bash
jmeter -g /path/to/results.jtl -o /path/to/report-dir/
```

- `-g` = path to CSV/JTL result file
- `-o` = **empty** output directory (will fail if it exists)
- Output: `index.html` + JS/CSS under `-o` dir — open in browser

### Sample Properties Tuning (optional)

Place a `report.properties` file next to your JTL to control what the dashboard generates:

```properties
# Generate only specific sections to speed up large result files
jmeter.reportgenerator.overall_granularity=1000
# Exclude slow dashboard sections
jmeter.reportgenerator.exclude_sections=Top5ErrorsBySampler,ResponseTimeDistribution
```

Usage:

```bash
jmeter -g results.jtl -o report-dir/ -q report.properties
```

### Common Pitfall: Dashboard Fails with "Cannot write to '...'"

The `-o` directory must not exist and must be writable. Always create a fresh dir:

```bash
mkdir -p report-dir && rm -rf report-dir && jmeter -g results.jtl -o report-dir/
```

---

## 2. CMDRunner (JMeter Plugins) — Custom Graphs & Reports

### Installation

CMDRunner is part of [JMeter Plugins](https://jmeter-plugins.org/). Install it via the JMeter Plugins Manager CLI:

```bash
# 1. Download and run the Plugins Manager CLI (one-time setup)
PLUGIN_DIR="$HOME/.hermes/lib/jmeter-plugins"
mkdir -p "$PLUGIN_DIR"

# Download plugins-manager if not present
if [ ! -f "$PLUGIN_DIR/plugins-manager.jar" ]; then
  curl -sL "https://repo1.maven.org/maven2/kg/apc/plugins-manager/1.9/plugins-manager-1.9.jar" \
    -o "$PLUGIN_DIR/plugins-manager.jar"
fi

# 2. Find JMeter home (brew installs under /opt/homebrew/opt/jmeter/libexec/)
JMETER_HOME=$(brew --prefix jmeter)/libexec

# 3. Install the Command-Line Graph Plotting Tool plugin
java -jar "$PLUGIN_DIR/plugins-manager.jar" \
  --install-dir "$JMETER_HOME" \
  --plugins jpgc-cmd,jpgc-graphs-basic,jpgc-synthesis,jpgc-filterresults,jpgc-mergeresults,jpgc-functions

# 4. CMDRunner.jar is now installed to $JMETER_HOME/lib/ext/CMDRunner.jar
CMDRUNNER="$JMETER_HOME/lib/ext/CMDRunner.jar"
```

### Generating Graphs with CMDRunner

**Basic graph (Response Times Over Time):**

```bash
java -jar "$CMDRUNNER" \
  --tool GraphGenerator \
  --input-file results.jtl \
  --output-file response-times-over-time.png \
  --graph-type ResponseTimesOverTime \
  --width 1200 --height 600
```

**Throughput (TPS) graph:**

```bash
java -jar "$CMDRUNNER" \
  --tool GraphGenerator \
  --input-file results.jtl \
  --output-file throughput-over-time.png \
  --graph-type ThroughputVsThreads \
  --width 1200 --height 600
```

**Latency vs Request:**

```bash
java -jar "$CMDRUNNER" \
  --tool GraphGenerator \
  --input-file results.jtl \
  --output-file latency-vs-request.png \
  --graph-type LatencyOverTime \
  --width 1200 --height 600
```

**Available graph types:** `ResponseTimesOverTime`, `ThroughputVsThreads`, `LatencyOverTime`, `ResponseTimePercentiles`, `ActiveThreadsOverTime`, `BytesThroughputOverTime`, `HitsPerSecond`, `ResponseTimesDistribution`, `SynthesisReport`

### Synthesis Report (Aggregate Report as CSV text)

```bash
java -jar "$CMDRUNNER" \
  --tool Reporter \
  --input-file results.jtl \
  --output-file synthesis-report.csv \
  --plugin-type SynthesisReport
```

Output columns: sampler_label, aggregate_count, average, median, min, max, error%, throughput, kb_sec

### Filter Results (pre-filter a large .jtl before graphing)

```bash
java -jar "$CMDRUNNER" \
  --tool FilterResults \
  --input-file results.jtl \
  --output-file filtered.jtl \
  --exclude-label-string "TearDown|SetUp|Backend Listener" \
  --start-offset 30 \
  --end-offset 30
```

Flags: `--start-offset N` (skip first N seconds), `--end-offset N` (skip last N seconds), `--include-label-string` / `--exclude-label-string`

### Merge Results (combine multiple .jtl files)

```bash
java -jar "$CMDRUNNER" \
  --tool MergeResults \
  --input-file test1.jtl --input-file test2.jtl \
  --output-file merged.jtl
```

---

## 3. CSV/JTL Direct Analysis (No GUI, No Plugins)

JMeter .jtl files are CSV by default. Parse them directly for precise numbers.

### Column Names (default jmeter save format)

```
timeStamp,elapsed,label,responseCode,responseMessage,threadName,dataType,
success,failureMessage,bytes,sentBytes,grpThreads,allThreads,URL,
Latency,IdleTime,Connect
```

- `elapsed` = response time in **milliseconds**
- `success` = `true`/`false`
- `Latency` = time to first byte
- `Connect` = TCP connect time

### Quick Percentiles with awk

```bash
# p50, p90, p95, p99 response times (elapsed column)
tail -n +2 results.jtl | awk -F',' '
{
  vals[NR]=$2
}
END {
  n=asort(vals)
  print "p50:",  vals[int(n*0.50)]
  print "p90:",  vals[int(n*0.90)]
  print "p95:",  vals[int(n*0.95)]
  print "p99:",  vals[int(n*0.99)]
  print "count:", n
}
'
```

### Error Rate

```bash
# Count errors vs total
tail -n +2 results.jtl | awk -F',' '
/success/ { next }                # skip header (if not stripped)
$7 == "false" || $7 == "FALSE" { errors++ }
{ total++ }
END {
  printf "Total: %d  Errors: %d  Error rate: %.2f%%\n", total, errors, (errors/total)*100
}
'
```

### Throughput (requests/sec)

```bash
tail -n +2 results.jtl | awk -F',' '
NR==1 { start=$1 }
{ end=$1 }
END {
  duration = (end - start) / 1000  # convert ms to seconds
  printf "Duration: %.2f sec  Total requests: %d  Throughput: %.2f req/s\n", duration, NR, NR/duration
}
'
```

### Python Analysis (most flexible, use execute_code tool)

Use `execute_code` with the `hermes_tools` `terminal()` function to run a Python script that reads the CSV and returns structured analysis. Pattern:

```python
import csv, json, sys

rows = []
with open("results.jtl") as f:
    reader = csv.DictReader(f)
    for r in reader:
        rows.append(r)

elapsed = sorted([int(r["elapsed"]) for r in rows if r["elapsed"].isdigit()])
n = len(elapsed)
print(json.dumps({
    "total_requests": n,
    "p50": elapsed[int(n*0.50)],
    "p90": elapsed[int(n*0.90)],
    "p95": elapsed[int(n*0.95)],
    "p99": elapsed[int(n*0.99)],
    "mean": sum(elapsed)/n if n else 0,
    "min": min(elapsed) if elapsed else 0,
    "max": max(elapsed) if elapsed else 0,
    "errors": sum(1 for r in rows if r.get("success","").lower() == "false"),
    "throughput_req_s": round(n / ((int(r["timeStamp"]) - int(rows[0]["timeStamp"])) / 1000), 2) if n > 1 and "timeStamp" in rows[0] else 0
}, indent=2))
```

---

## 4. End-to-End Workflow: Run Test + Generate Report

```bash
# 1. Run test in non-GUI mode
jmeter -n -t test-plan.jmx -l results.jtl

# 2. Clone the result to a timestamped directory
REPORT_DIR="report-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$REPORT_DIR"

# 3. Generate HTML Dashboard
rm -rf "$REPORT_DIR/dashboard"
jmeter -g results.jtl -o "$REPORT_DIR/dashboard"

# 4. Generate CMDRunner graphs (if CMDRunner.jar is available)
CMDRUNNER="$JMETER_HOME/lib/ext/CMDRunner.jar"
if [ -f "$CMDRUNNER" ]; then
  java -jar "$CMDRUNNER" --tool GraphGenerator \
    --input-file results.jtl --output-file "$REPORT_DIR/response-times.png" \
    --graph-type ResponseTimesOverTime --width 1200 --height 600
  java -jar "$CMDRUNNER" --tool Reporter \
    --input-file results.jtl --output-file "$REPORT_DIR/synthesis.csv" \
    --plugin-type SynthesisReport
fi

# 5. Extract key metrics
echo "=== JMeter Report Summary ===" > "$REPORT_DIR/summary.txt"
tail -n +2 results.jtl | awk -F',' '
$7 == "false" || $7 == "FALSE" { errors++ }
{ total++; sum+=$2; vals[NR]=$2; last=$1 }
END {
  n=asort(vals)
  duration = (last - start) / 1000
  printf "Total requests: %d\n", total >> "/dev/stderr"
  printf "Errors: %d (%.2f%%)\n", errors, (errors/total)*100 >> "/dev/stderr"
  printf "p50: %d ms | p90: %d ms | p95: %d ms | p99: %d ms\n", \
    vals[int(n*0.50)], vals[int(n*0.90)], vals[int(n*0.95)], vals[int(n*0.99)] >> "/dev/stderr"
  printf "Throughput: %.2f req/s\n", total/duration >> "/dev/stderr"
}' 2>> "$REPORT_DIR/summary.txt"

echo "Report generated in: $REPORT_DIR"
```

## 5. Multi-Server / Dual-Generator Test Handling

When you run the same test plan on two load generators (e.g. srv1=500 users, srv2=500 users → 1000 total), the JTL files have the **same name** on each server. Do NOT treat them as separate tests.

### Merge Pattern

Files from the same test (matching base names) must be **merged into one** execution file:

```text
srv1-Courtyard-500-1.jtl   (row 1..72570)       → exec-06-500users-Courtyard-500-1.jtl
srv2-Courtyard-500-1.jtl   (row 1..72536)       ↗  (header + merged data)
```

Each dual-server execution represents `file_users × 2` total virtual users.

### ⚠️ CRITICAL: The 3-Column Correction

When merging a second generator's data, **three columns must be offset** by the intended per-generator user count. The HTML Dashboard uses `allThreads` for its "Active Threads Over Time" graph — renaming only `threadName` is NOT enough:

| Column | Index | What to do | Why |
|--------|-------|------------|-----|
| `threadName` | 5 | Append `+ OFFSET` to thread number | Unique thread names |
| `grpThreads` | 11 | Add `+ OFFSET` | Group-level thread count |
| `allThreads` | 12 | Add `+ OFFSET` | **Dashboard reads this** |

**Example:** Two generators each running 100 users (200 total):

| Before (srv2 raw) | After correction (offset=100) |
|-------------------|-------------------------------|
| `threadName=...1-1` | `threadName=...1-101` |
| `grpThreads=50` | `grpThreads=150` |
| `allThreads=50` | `allThreads=150` |

Without this fix, the Dashboard reports 100 max concurrent users instead of 200.

### Offset Value: Use the Intended Count, Not Actual Max

The offset must be the **named** per-generator user count from the filename, NOT the actual maximum thread number in the data. During ramp-up, fewer threads are active — using the actual max produces a wrong offset (e.g. 867 instead of 1200).

```python
# CORRECT
offset = 500  # from filename "Courtyard-500-1.jtl"

# WRONG
offset = max_actual_threads_in_data  # might be 500 but also 867
```

## Related References

See `references/jtl-merge-split.md` for the complete Python merge implementation covering:
- **CSV quoting** — always use `csv.reader`, never `line.split(",")`, because `failureMessage` contains quoted commas
- Server file prefix strategy
- Gap detection for execution split
- 3-column offset correction
- HTML Dashboard generation loop

See `references/generate_client_pdf.py` for the Python PDF generator covering:
- Latin-1 sanitization, table layout, commentary, timezone conversion
- **Total row at bottom**, not top
- **Error breakdown by HTTP code** with 1-2 example failed URLs per code (504, 503, 500, 502, SSLHandshakeException), showing actual request paths
- **No agent branding** in client-facing PDFs
- **Total row at bottom** of table, not top
- **Error detail** includes HTTP code breakdown (504, 503, 500, etc.) with percentages and example URLs
- **Timezone** converted to Spain (CEST, UTC+2) for client context — the script has zero agent or framework references

### Split by Time Gaps

After merging all files (with server prefix), detect distinct test executions by finding gaps > 10 minutes between consecutive timestamps. Sort **by timestamp, not filename** — alphabetical order of files does not match chronological test order.

Use `references/jtl-merge-split.md` for the complete Python workflow: grouping, gap detection, execution splitting, and dashboard generation.

## 6. PDF Report Generation (White-Label, Client-Ready)

Generate a client-facing PDF from `statistics.json` after the HTML Dashboard is built. The report focuses on a curated set of transaction labels and presents avg/min/max/p90 in **seconds** (JMeter reports in ms).

### Requirements

```bash
pip3 install fpdf2
```

### Critical: CSV Parsing

JMeter JTL files may contain quoted fields in the `failureMessage` column (e.g. `"Number of samples in transaction : 1, number of failing samples : 0"`). A naive `line.split(",")` will split inside the quotes and shift all subsequent columns, corrupting `allThreads` and `grpThreads`.

**Always use Python's `csv` module** (`csv.reader` / `csv.DictReader`) for reading and `csv.writer` for writing JTL files. The csv module handles RFC 4180 quoting correctly.

```python
# RIGHT — handles quotes
with open("results.jtl", newline="") as f:
    for row in csv.reader(f):
        all_threads = row[12]

# WRONG — breaks on quoted commas
with open("results.jtl") as f:
    for line in f:
        cols = line.split(",")  # may shift columns!
```

### Table Layout

- Transactions appear in logical order (Homepage → Category → ... → Total)
- **Total ALWAYS at the bottom** of the table, not the top
- Values in seconds with 3 decimal places
- Error rate as percentage

### Timezone Handling

JMeter timestamps are epoch milliseconds. Convert to the target timezone using `datetime.fromtimestamp(ts/1000, tz=...)`. For this user's workflow, the client is in Spain (CEST, UTC+2).

```python
from datetime import timezone, timedelta
spain = timezone(timedelta(hours=2), "CEST")
dt = datetime.fromtimestamp(ts / 1000, tz=spain)
```

### Branding Rule

Client-facing deliverables must be **white-label**: no agent name, no framework attribution, no "generated by" footers. The report header shows only the client's data (date, test description, transaction table).

See `references/generate_client_pdf.py` for the complete Python implementation including:
- Latin-1 sanitization (fpdf2 core fonts don't support Unicode bullets or em-dashes)
- Per-transaction commentary generation
- Error detail section
- Throughput summary

## Data Safety: Never Work on Originals

Always copy JTL files to a fresh working directory before merging, filtering, splitting, or analyzing. Original files are the single source of truth — once modified they cannot be recreated without re-running the test.

When files from multiple servers share filenames, prefix them during copy to prevent overwrites:

```bash
mkdir -p workdir
for f in serverA/*.jtl; do cp "$f" workdir/"srvA-$(basename "$f")"; done
for f in serverB/*.jtl; do cp "$f" workdir/"srvB-$(basename "$f")"; done
```

### Renaming an Execution

To relabel a test (e.g. original was 2400 users but should be 1500), rename both the JTL file and its report directory. The data stays the same — only the label changes:

```bash
mv exec-04-2400users-Courtyard-2400-2.jtl exec-04-1500users-Courtyard-1500.jtl
mv report-04-2400users report-04-1500users
```

The report content still reflects the original data; regenerate the dashboard if you want the new label reflected in the HTML.

## 7. Client Deliverable Packaging

After HTML Dashboards and PDFs are generated, package each execution as a standalone deliverable:

```bash
# Structure: for-client/EXECUTION_NAME/
#   report-EXECUTION_NAME.pdf   (standalone PDF)
#   report-EXECUTION_NAME.zip   (HTML dashboard + PDF inside)

for rdir in reports/*/; do
  name=$(basename "$rdir")
  clean="${name//users/}"
  mkdir -p "for-client/$clean"
  cp "$rdir/reporte.pdf" "for-client/$clean/report-$clean.pdf"
  cd "$rdir" && zip -r "../for-client/$clean/report-$clean.zip" . -x "reporte.pdf"
  zip -j "../for-client/$clean/report-$clean.zip" "../for-client/$clean/report-$clean.pdf"
done
```

Each ZIP is self-contained: unzip → open `index.html`. The PDF sits alongside for quick email attachment.

### Deploy to Remote Mac

```bash
tar czf /tmp/pack.tar.gz for-client/
scp /tmp/pack.tar.gz remote-user@remote-host:/tmp/
ssh remote-user@remote-host "tar xzf /tmp/pack.tar.gz -C /Users/target/Documents/"
```

Use `sshpass -p PASS` when the remote Mac has no key-based auth. Destination path must match the local user's actual home — check `/Users/` on the remote to confirm the username exists.

## Common Pitfalls

1. **Dashboard fails with "Cannot write to directory"** — the `-o` directory must NOT exist. Always `rm -rf` it first.

2. **CMDRunner "java.lang.NoClassDefFoundError"** — JMeter's `lib/ext/` must contain the plugin jars. Verify with `ls "$JMETER_HOME/lib/ext/" | grep -i cmd`.

3. **CSV header mismatch** — JMeter allows custom save fields. If the `.jtl` was generated with non-default config, the column order may differ. Always check: `head -1 results.jtl`.

4. **Memory errors for large files** — JMeter default JVM heap is small (512MB). Increase for large results:
   ```bash
   JVM_ARGS="-Xmx4g" jmeter -g results.jtl -o report/
   ```

5. **CMDRunner graph looks empty** — filter out ramp-up/ramp-down with `--start-offset 60 --end-offset 30` (skip first 60s, last 30s).

6. **JTL files with XML format** — JMeter can save as XML. These are NOT CSV. Regenerate with CSV format or use:
   ```bash
   # Re-run the test plan with CSV output
   # or use xsltproc to convert XML JTL to CSV
   ```

7. **Homebrew JMeter on macOS** — brew installs under `/opt/homebrew/opt/jmeter/libexec/`. Set `$JMETER_HOME` to this path for plugin installation commands.

8. **Alphabetical file order != chronological order** — when merging multiple server files, `sorted(glob.glob("*.jtl"))` sorts by filename, not time. `Courtyard-600-1.jtl` sorts before `Courtyard-850-1.jtl` even if 850 ran first. Always base execution boundaries on timestamp analysis, not file list order.

9. **Dual-server files with same name are ONE execution** — never split `srv1-Courtyard-500-1.jtl` and `srv2-Courtyard-500-1.jtl` into separate files. Each generator ran half the users of the same test plan. Combine them before generating dashboards.

10. **Think-time gaps within a test look like boundaries** — a running test has natural gaps of 2-7 minutes between request iterations. Use a threshold > 10 minutes when detecting execution boundaries by timestamp gaps. See `references/jtl-merge-split.md` for the gap-detection pattern.

11. **Dual-generator merge: renaming threadName is NOT enough** — the Dashboard reads `allThreads` (col 12) for the Active Threads Over Time graph. You MUST also offset `grpThreads` (col 11) and `allThreads` by the per-generator user count. Without this, a 2x500 test reports 500 max concurrent users instead of 1000.

12. **Client-facing PDFs must be white-label** — no agent name, no framework attribution, no "generated by" footers. The header shows only the client's data (date, users, transaction table). This is not optional for deliverables sent to paying customers.

13. **Remote Mac destination must match the real user** — `/Users/antonio/` is NOT necessarily the same user as the interactive session. Check `dscl . list /Users` on the remote to find the actual username. Copying to a root-owned `/Users/` directory makes files invisible in Finder.

## Verification Checklist

- [ ] HTML Dashboard generates and opens in browser (tested `jmeter -g`)
- [ ] At least one CMDRunner graph generates successfully (if plugins installed)
- [ ] CSV percentiles match between awk and Python methods (within rounding tolerance)
- [ ] All output files are in a dated report directory (not scattered)
- [ ] Summary text file contains: total requests, error rate, p50/p90/p95/p99, throughput
- [ ] For large files (>500MB), verify JVM heap was increased
