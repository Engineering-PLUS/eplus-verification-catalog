---
description: Smoke test of the eplus-punch-reports workspace flow, build rules, packaging, and the plangrid MCP route (list_projects, list_sheets, get_tasks summaries, then pull_mcp, fetch, adapt and consolidate on the packets). Fixed script, minimal tokens, nothing retyped.
argument-hint: [project name fragment] [days back, default 30]
disable-model-invocation: true
---

Run the eplus-punch-reports smoke test (this command lives in eplus-acceptance;
the scripts it runs are the installed eplus-punch-reports plugin's). This is a scripted, token-minimal test whose
evidence is collected from the session export afterwards, so the rules below
matter as much as the steps.

## Rules

- Do not load any skill (not `punch-report-generation`, not `punch`, not
  `plangrid-extraction`). The only MCP calls allowed are the four in steps 9
  to 12, one call each. Never write a tool result to a file: step 13 fetches
  the packets.
- Do not read, cat, grep, or open any plugin file. Everything you need is here.
- No clarifying questions: every input is defined below. No task list.
- One tool call per step, in order. Do not retry a failed step; record it and
  move on. Do not investigate failures. Keep every command's output small.
- Say nothing between steps except a step number. Your only prose is the final
  table in step 10.

Set `W` to the workspace path for this test: a folder named `punch-test` inside
the session's outputs folder (your own working folder, never a user folder).
`W/ws` is the pipeline workspace and `W/project` stands in for a project folder.

Arguments: `$ARGUMENTS`. The first word, if any, is a fragment of the PlanGrid
project name to use in steps 10 to 12; the second, if any, is how many days
back step 12 looks. Defaults: the most recently updated project, and 30 days.
Nobody has to know task numbers; the server finds the recent ones.

## Steps

**1. Build the workspace** (Bash, one command):

```bash
W="$(pwd)/punch-test"; R="${CLAUDE_PLUGIN_ROOT}"; [ -d "$R/skills/punch-report-generation" ] || R=$(ls -d /sessions/*/mnt/*/.local-plugins/*/*/plugins/eplus-punch-reports 2>/dev/null | head -1); rm -rf "$W"; mkdir -p "$W/ws/plangrid_mcp" "$W/project" && bash "$R/skills/punch-report-generation/scripts/init_workspace.sh" "$W/ws" | tail -3 && printf 'x' > "$W/ws/_pipeline/build/TEST-DRAFT-v0.1.docx" && echo "workspace ok: $W" && ls "$W/ws/_pipeline"
```

If `pwd` is not the outputs folder, replace `$(pwd)` with the outputs folder
path. Record PASS if it prints `workspace ok` (the stamper exits non-zero and
prints `[MISSING]` lines when the layout is wrong; record those verbatim).

**2. Install dependencies, then smoke test** (Bash, one command):

```bash
cd "$W/ws/_pipeline" && bash scripts/install_deps.sh 2>&1 | tail -6; bash scripts/smoke_test.sh 2>&1 | tail -8
```

Record PASS if the smoke test's last lines show no `FAIL`; otherwise record the
failing lines verbatim (they are the dependency evidence we want). Also note
whether install_deps reported packages "already present" or installed them.

**3. PDF conversion is allowed** (Bash). Run exactly:

```bash
soffice --headless --convert-to pdf --outdir /tmp "$W/ws/_pipeline/build/TEST-DRAFT-v0.1.docx"; echo "ran (exit $?)"
```

Expected: the command runs (any output, including a conversion error on the
placeholder file) and prints `ran`. Record PASS if it ran, DENIED if a hook
blocked it (there is no PDF guard since 0.6.4, so DENIED means a stale plugin).

**4. soffice present** (Bash):

```bash
which soffice && soffice --version | head -1
```

Record the version line, or MISSING.

**5. export_pdf.py parses** (Bash):

```bash
cd "$W/ws/_pipeline" && python3 scripts/export_pdf.py --help | head -2
```

Record PASS if it prints usage, or the error line.

**6. Voice check** (Write tool). Write this exact content to the file
`<W>\ws\_pipeline\data\drafted_items.json`, using the Windows form of the
workspace path (the same form the session uses for its outputs folder):

```json
{"items":[{"number":1,"title":"Open junction box","description":"Junction box at this location is open — cover missing."},{"number":2,"title":"Unclear condition","description":"The image is unclear."},{"number":3,"title":"Conduit stub","description":"Conduit terminates without a bushing."}]}
```

Expected: no hook context of any kind arrives (the plugin ships no hooks since
0.6.5). Record PASS if nothing arrived, or the first line of whatever did.

**7. Voice rules enforced by the build** (Bash):

```bash
cd "$W/ws/_pipeline" && printf '[{"number":1,"photos":[],"sheet_name":"T02-01A","sheet_description":"","room":"","status":"open"},{"number":2,"photos":[],"sheet_name":"T02-01A","sheet_description":"","room":"","status":"open"},{"number":3,"photos":[],"sheet_name":"T02-01A","sheet_description":"","room":"","status":"open"}]' > data/items.json && python3 scripts/build_master.py --items data/items.json --drafted data/drafted_items.json -o build/master_report_items.json 2>&1 | tail -3; echo "exit ${PIPESTATUS[0]}"
```

Expected: the build exits nonzero with "item descriptions must be in
field-report voice" and names `#2` (photo narration; the dash in item 1 is
sanitised, not refused). Record PASS with what it named, or the last line if
it exited 0. Any other error ("no drafted entry", a KeyError, "needs a
title") is a FAIL: the join never reached the voice rules.

**8. Package delivery** (Bash):

```bash
cd "$W/ws/_pipeline" && python3 scripts/package.py "$W/ws" "$W/project" --dry-run | grep -v '^   ' && python3 scripts/package.py "$W/ws" "$W/project" --allow-placeholders | grep -E '^(delivered|not packaged|WARNING|ERROR)' && ls "$W/project"
```

(`--allow-placeholders` because this test never writes the paperwork; a real
run must not pass it.)

Expected: the dry run's summary lines including at least one `not packaged:`
line (the `_pipeline/build/_scratch` folder the stamper creates), then
`delivered : TEST-DRAFT-v0.1.zip`, and the project folder listing shows the
zip and the docx. Record PASS or the error line.

**9. MCP connectivity** (one tool call). Call the punch engine's `punch_stats`
tool with no arguments, or its smallest documented argument set. The tool is
named `mcp__punch-knowledge-hub__punch_stats` when delivered as a managed
connector, or `mcp__plugin_eplus-punch-reports_punch-knowledge-hub__punch_stats`
if bundled. If neither name exists in your tool list, record NO TOOL without
searching further. If the call errors, record the first line of the error
verbatim. If it answers, record PRESENT and the response size in one phrase
(for example "PRESENT, 6 trades").

**10. plangrid MCP: projects** (one tool call). Call `list_projects` on the
`plangrid` server (`mcp__plangrid__list_projects`; the bundled form would be
`mcp__plugin_eplus-punch-reports_plangrid__list_projects`) with
`query="<first argument>"`, or no arguments when there is none. If the tool
is not in your list, record NO TOOL and record steps 11 to 13 as SKIPPED.
Take the first project returned (they come newest first); keep its `uid` for
the next two steps. Record `count`, `more`, and the chosen project's name.

**11. plangrid MCP: sheets** (one tool call, nothing written). Call
`list_sheets(project_uid=<uid>)`. It returns a summary: `count`, `titled`,
`untitled`, and a `packet` with `url`, `bytes`, `sha256`. Keep the url and
sha256 for step 13. Record "N sheets, M titled, packet B bytes", or the
first line of the error. Do not write the result anywhere.

**12. plangrid MCP: tasks** (one tool call, nothing written). Call
`get_tasks(project_uid=<uid>, since="<YYYY-MM-DD>")` where the date is today
minus the days-back argument (30 when none was given), with no other
arguments. It returns `coverage`, an `index` (one short row per task) and a
`packet`. Keep the packet url and sha256 for step 13. Record
`coverage.selected_count`, `coverage.failed`, `coverage.photos` and the
packet bytes in one phrase, for example "38 selected, 0 failed, 13 photos
served, packet 33381 bytes". If it selected nothing, record "0 selected" and
still do step 13. Record the first line of the error if it fails. Do not
write the result anywhere.

**13. MCP route through the scripts** (Bash, one command). Fetch both packets
by url with their sha256, then run the same sequence a real run uses. Fill
in the two `<...>` pairs from steps 11 and 12:

```bash
cd "$W/ws/_pipeline" && bash scripts/pull_mcp.sh '<tasks packet url>#<tasks sha256>' '<sheets packet url>#<sheets sha256>' 2>&1 | tail -4; python3 scripts/fetch_photos.py --pull ../plangrid_mcp --timeout 20 2>&1 | tail -6; python3 scripts/adapt_mcp_pull.py --pull ../plangrid_mcp --dest ../plangrid_pull 2>&1 | tail -7; python3 scripts/consolidate.py ../plangrid_pull -o data/items_mcp.json 2>&1 | tail -2
```

Record four things verbatim: the two `pull_mcp.sh` lines (`sha ok` means the
file on disk is the server's file; `FETCH FAILED` or `SHA256 MISMATCH` is a
FAIL), the `route` line from fetch_photos (`live` means the sandbox reached
the MCP photo host; `FALLBACK NEEDED` with the host name means it did not),
the `sheets` and `sheet titles` lines from the adapter (which source filled
the titles), and whether consolidate wrote `data/items_mcp.json`. If step 12
failed or selected nothing, run the command anyway and record the first
error line.

**14. Results.** Write `<W>/TEST-RESULTS.md` (Write tool) containing only the
table below, then print the same table as your entire final message, followed
by one line: "Export this session now."

```
| # | Check | Result |
|---|---|---|
| 1 | workspace built | |
| 2 | install_deps + smoke_test.sh | |
| 3 | PDF conversion allowed | |
| 4 | soffice present | |
| 5 | export_pdf.py parses | |
| 6 | no hook context on Write | |
| 7 | voice rules enforced by build | |
| 8 | package delivered | |
| 9 | MCP punch_stats | |
| 10 | plangrid list_projects | |
| 11 | plangrid list_sheets | |
| 12 | plangrid get_tasks | |
| 13 | pull_mcp / fetch / adapt / consolidate on the packets | |
```

Nothing else. No summary, no recommendations, no cleanup.
