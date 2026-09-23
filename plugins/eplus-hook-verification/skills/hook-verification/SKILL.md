---
name: hook-verification
description: How the EPLUS hook verification suite works, how to read a run's results (report.md, results.jsonl, summary.json, hook-verification.log in the session folder), how to add or change expectations for a plugin's hooks, and how to check a session export for the run. Use when the user asks whether the plugin hooks work on this seat, asks about a hook-verification run, or wants a new hook covered by the suite. To start a run, use the /eplus-hook-verification:verify-hooks command; never run the suite from bash.
---

# EPLUS hook verification

One plugin that tests every hook wiring of every installed EPLUS plugin on the seat
it runs on, and leaves the evidence where the session export picks it up.

## What runs where

- Cowork executes plugin hooks on the Windows host under PowerShell 5.1. The model's
  shell runs inside the Linux VM. So the suite is itself a hook:
  `scripts/run-suite.ps1`, wired three ways in `hooks/hooks.json`, all pointing at the
  same script, all with a 300 second timeout:
  - `UserPromptSubmit`: fires on every prompt; the script exits in a few hundred
    milliseconds unless the prompt carries the trigger.
  - `UserPromptExpansion` with matcher `verify-hooks`: fires when the slash command
    expands, if this build dispatches that event.
  - `PreToolUse` with matcher `Glob`: fires when the model calls Glob with a pattern
    starting `EPLUS-HOOK-VERIFY`. The command body tells the model to do that, so the
    run does not depend on either prompt event. A trigger lock keyed on `prompt_id`
    keeps one prompt from starting two runs.
- The runner discovers plugins by walking up from its own `${CLAUDE_PLUGIN_ROOT}` to
  the `cowork_plugins` folder and scanning both layouts,
  `marketplaces\<marketplace>\plugins\<plugin>` and `cache\<marketplace>\<plugin>\<version>`.
  Plugins named in `expectations/_suite.json` `skip_plugins` (this plugin and
  hook-testing-plugin) are inventoried but never replayed.

## Three layers per hook

1. **Static**: the hooks.json entry parses, the matcher compiles, the script exists,
   is ASCII with no BOM and LF endings, ends in `exit 0`, names an `EPLUS_NO_*`
   escape hatch, and the command has no `sh` half. Verdict `PASS` or `WARN` with codes.
2. **Replay**: each wiring is run the way Cowork runs it (`powershell.exe -Command <the
   hooks.json command>` with `${CLAUDE_PLUGIN_ROOT}` resolved), with a fixture on stdin
   and a per-case sandbox: `CLAUDE_PLUGIN_DATA`, `TEMP`, `TMP`, `CLAUDE_PROJECT_DIR`,
   `transcript_path`, `cwd` and `session_id` all point into
   `<run>\sandbox\<plugin>__<case>\`, every inherited `EPLUS_NO_*` variable is
   cleared, and `EPLUS_HOOK_VERIFY_REPLAY=1` is set so a script can tell it is under
   test. Exit code, stdout, stderr and duration are captured; stdout is checked
   against the hook protocol and the case's assertions.
3. **Live** (`--live`): counts the `hook_*` attachments already in this session's
   transcript by event and script, so you can see which wirings have fired for real.
   Silent hooks (exit 0, no stdout) leave no attachment, so absence is not proof.
   On the first prompt of a session there is no transcript yet (Cowork writes it
   after that prompt's hooks finish), so the summary says `Live hook counts: NOT
   AVAILABLE`; rerun as a later prompt with `--static --live` for the counts.

## Cases and expectations

`expectations/<plugin>.json` lists named cases per plugin. Without a file, every
wiring gets one generic `smoke-*` case (base fixture for the event, a `tool_name`
derived from the matcher when that is unambiguous) that only checks the protocol:
exit 0, empty stdout or a JSON object with known keys, empty stderr. Smoke passes are
labelled `smoke` so a report never confuses "did not crash" with "behaved".

Case fields:

```
{ "id": "stamp-unknown", "event": "PreToolUse", "matcher": "<optional, selects one wiring>",
  "fixture": "PreToolUse",                          // optional, defaults to <event>.json
  "payload": { "tool_name": "...", "tool_input": { ... } },   // deep-merged over the fixture
  "env": { "EPLUS_NO_X": "1" },                     // extra environment for this case
  "timeout_ms": 20000,
  "skip": "reason",                                 // records SKIP instead of running
  "expect": {
    "exit_code": 0,
    "stdout": "empty" | "json" | "empty_or_json" | "any",
    "stderr": "empty" | "any",
    "max_ms": 5000,
    "json_required": ["hookSpecificOutput.permissionDecision"],
    "json_forbidden": ["hookSpecificOutput.updatedInput"],
    "json_equals": { "hookSpecificOutput.permissionDecision": "allow" },
    "json_regex":  { "hookSpecificOutput.additionalContext": "Reporter identity" },
    "stdout_regex": "...",
    "files": [ { "path": "${SESSION_DIR}/hook-probe.log", "regex": "SubagentStart" },
               { "path": "${PLUGIN_DATA}/x.txt", "exists": false } ]
  } }
```

Placeholders usable in `payload`, `env`, `json_equals` values and `files.path`:
`${SANDBOX}`, `${PLUGIN_ROOT}`, `${PLUGIN_DATA}`, `${TEMP}`, `${PROJECT}`,
`${SESSION_ID}`, `${TRANSCRIPT}`, `${SESSION_DIR}` (the `<transcript dir>\<session_id>`
folder that hooks such as hook-probe and show-researcher-final write into).

Fixtures live in `fixtures/<Event>.json`, one per hook event, copied from the
hook-testing-plugin fixture set so this plugin has no runtime dependency on it.

## Verdicts

`PASS`, `FAIL` (with codes such as `EXIT_CODE:1`, `STDOUT_PROTOCOL`, `STDERR_NOT_EMPTY`,
`MISSING:<path>`, `FORBIDDEN_PRESENT:<path>`, `VALUE:<path>`, `REGEX:<path>`,
`FILE_MISSING:<name>`, `FILE_CONTENT:<name>`, `TIMEOUT`, `DURATION:<ms>`,
`HOOK_EVENT_NAME_MISMATCH`, `UNKNOWN_TOP_KEY`), `ERROR` (`SPAWN_ERROR`,
`FIXTURE_INVALID`, `EXPECTATIONS_INVALID`, `HOOKS_JSON:<why>`), `SKIP` (a `skip`
reason, `MATCHER_UNRESOLVED`, or `DEADLINE` when the run budget ran out),
`WARN` (static findings), `NOT_WIRED` (an expectations case names an event or matcher
the plugin no longer wires). Run status is `COMPLETE` or `INCOMPLETE`; overall verdict
is `FAIL` if any case is `FAIL` or `ERROR`.

## Where the results are

Inside the session folder the exporter zips, `<dirname(transcript_path)>\<session_id>\`:

- `hook-verification.log`: one line per case, plus RUN START and RUN END lines, across
  every run in the session. Log Lens shows it in the Logs tab.
- `hook-verification\<run id>\report.md`: the human summary (what the command tells
  the model to read). `summary.json`, `results.jsonl` (one record per static check or
  case), `inventory.json` (every discovered plugin and wiring, with layout),
  `cases\<plugin>__<case>.stdin.json|stdout.txt|stderr.txt`, `sandbox\`, and
  `live-counts.json` when `--live` was used.

The hook's own summary is also injected as context on the triggering event, and the
command has the model reproduce it verbatim in its reply, so a session export shows
the run three ways: the hook attachment, the assistant text, and the files.

## Checking an export afterwards

From the dev box, with the export zip: the `<cliSessionId>/hook-verification.log` and
`<cliSessionId>/hook-verification/<run>/` entries are in the zip alongside
`subagents/` and `tool-results/`. In Log Lens open the session, Logs tab, pick
`hook-verification.log`; the Files tab lists the run folder. With the forensics skill,
`explore.py hooks <export>` shows the runner's own attachment under the triggering
event, including how long the run took.

## Running on the dev box

```
powershell -NoProfile -ExecutionPolicy Bypass -File scripts\run-suite.ps1 -PluginsRoots "<catalog>\plugins;<other catalog>\plugins" -Selection all -OutDir <folder>
```

Same runner, same expectations; prints report.md to stdout and writes the same
artifacts under `<folder>\hook-verification\<run>\`. Run it before every plugin bump.

## Adding coverage for a new hook

1. Add or extend `expectations/<plugin>.json` with one case per behaviour: the normal
   path, the escape hatch (`env` with the plugin's `EPLUS_NO_*` set to `1`, expect
   `stdout: empty`), and any file it writes (`files` with `${SESSION_DIR}` or
   `${PLUGIN_DATA}`).
2. If the event has no fixture yet, add `fixtures/<Event>.json` matching the input
   schema in the hooks reference.
3. Run the dev-box command, then bump this plugin's version in
   `.claude-plugin/marketplace.json` so seats refresh it.
