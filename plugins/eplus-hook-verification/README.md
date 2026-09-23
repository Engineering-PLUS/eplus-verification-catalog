# eplus-hook-verification

The test suite for every hook in every EPLUS plugin, run from inside a live Cowork
session on the seat under test, with the results written where the session export
picks them up. One command reruns it: `/eplus-hook-verification:verify-hooks`.

It replaces the always-on generic logger approach of `hook-testing-plugin` (which
fires on thirty events and drowns the transcript) with an on-demand replay: nothing
runs until asked, and one run proves every wiring of every installed plugin against
fixtures and expectations in a few minutes.

## Contents

| Component | Path | Purpose |
|-----------|------|---------|
| Hooks | [`hooks/hooks.json`](hooks/hooks.json) | Three wirings of the same runner: `UserPromptSubmit`, `UserPromptExpansion` (matcher `verify-hooks`), `PreToolUse` (matcher `Glob`). All have `timeout: 300` because the default for `UserPromptSubmit` command hooks is 30 s. |
| Runner | [`scripts/run-suite.ps1`](scripts/run-suite.ps1) | Host-side PowerShell 5.1 runner: trigger parsing, plugin discovery, static checks, fixture replay in per-case sandboxes, assertions, results, report, live-attachment count. Also runs from a terminal on the dev box. |
| Command | [`commands/verify-hooks.md`](commands/verify-hooks.md) | The trigger. Carries the `EPLUS-HOOK-VERIFY:` marker line and tells the model to call Glob with the marker pattern, relay the summary verbatim, and read `report.md`. |
| Skill | [`skills/hook-verification/SKILL.md`](skills/hook-verification/SKILL.md) | How the suite works, how to read results, how to add expectations. |
| Fixtures | [`fixtures/`](fixtures/) | 31 base payloads, one per hook event, copied from hook-testing-plugin so this plugin has no runtime dependency on it. |
| Expectations | [`expectations/`](expectations/) | `_suite.json` (skips, budget) and one `<plugin>.json` per plugin with named cases and assertions. Plugins without a file get protocol-only smoke cases. |

## Why the runner is a hook

Cowork executes plugin hooks on the Windows host under PowerShell; the model's shell
tool runs in a Linux VM. The suite has to spawn `powershell.exe` thirty-odd times,
so it can only run as a hook. Three triggers point at the same script so the run does
not depend on which prompt events this build dispatches:

1. `UserPromptSubmit` sees the raw prompt. If it starts with the slash command, the
   tagged `<command-name>` form, or the `EPLUS-HOOK-VERIFY:` marker, the run starts.
2. `UserPromptExpansion` fires when a slash command expands (documented, never yet
   observed in an EPLUS export).
3. `PreToolUse` on `Glob`: the command body has the model call Glob with pattern
   `EPLUS-HOOK-VERIFY <args>`. The hook sees `tool_input.pattern`, runs the suite,
   allows the harmless Glob, and returns the summary next to the tool result. This
   is the trigger that is guaranteed to reach a hook.

A lock file keyed on `prompt_id` under the run folder makes sure one prompt starts
only one run whichever event wins. The fast path (no trigger) costs one PowerShell
start per prompt and per Glob call; enable this plugin on testing profiles only.

## What one run produces

Under `<dirname(transcript_path)>\<session_id>\`, the folder the exporter zips next
to `subagents\` and `tool-results\`:

```
hook-verification.log                       one line per case across all runs (Logs tab in Log Lens)
hook-verification\<run id>\report.md        the human summary the model reads back
hook-verification\<run id>\summary.json     status, verdict, counts, env probe, artifact paths
hook-verification\<run id>\results.jsonl    one record per static check and replay case
hook-verification\<run id>\inventory.json   every discovered plugin, layout, wiring
hook-verification\<run id>\cases\           <plugin>__<case>.stdin.json / .stdout.txt / .stderr.txt
hook-verification\<run id>\sandbox\         per-case CLAUDE_PLUGIN_DATA, TEMP, project, transcript dirs
hook-verification\<run id>\live-counts.json with --live: hook attachments by event, script, type
```

The summary is also injected as `additionalContext` on the triggering event and the
command has the model reproduce it verbatim, so the export shows the run as a hook
attachment, as assistant text, and as files.

## Verdicts

`PASS`, `FAIL` (with codes), `ERROR`, `SKIP`, `WARN` (static findings), `NOT_WIRED`;
run status `COMPLETE` or `INCOMPLETE` (budget exhausted). Smoke-only passes are
labelled so nobody reads "did not crash" as "behaved". See the skill for the codes
and the expectations schema.

## Running on the dev box

```
powershell -NoProfile -ExecutionPolicy Bypass -File plugins\eplus-hook-verification\scripts\run-suite.ps1 -PluginsRoots "<eplus-plugin-catalog>\plugins;<vo-plugin-catalog>\plugins" -Selection all -OutDir %TEMP%\hv
```

Prints `report.md` and writes the same artifacts under `<OutDir>\hook-verification\`.

## Field status

First field run 2026-09-17 (export 1789684357094, testing profile, desktop 1.52386.x):

- The slash command reached the `UserPromptSubmit` hook as raw text; that trigger
  carried the run. The 300 s timeout held (run took 77 s for 144 checks). The model
  reproduced the summary and read `report.md` from the host path.
- `CLAUDE_CODE_SESSION_ID`, `CLAUDE_PLUGIN_DATA` and `CLAUDE_PROJECT_DIR` were present
  in the hook environment; `CLAUDE_CODE_PLUGIN_CACHE_DIR` was not.
- The run folder and `hook-verification.log` were in the export zip.
- Live plugins under `marketplaces\` all passed. Every failure came from stale copies
  under `cache\` (older versions, and a marketplace no longer registered). 0.1.1 lists
  those as stale and does not replay them unless `--include-cache` is given.
- Every script on the seat had CRLF endings (the seat's git checkout converts them);
  0.1.1 no longer warns about CRLF in `.ps1`.
- `--live` counted nothing: the transcript is open for writing and the read failed
  silently. 0.1.1 opens it shared and records the line count or the error.

Field run 2026-09-23 (export 1790148676704, 0.1.1): 51 of 51 checks passed.

- `--live` on the session's FIRST prompt found no transcript: Cowork writes it only
  after that prompt's hooks return. 0.1.2 says so in plain words (`Live hook counts:
  NOT AVAILABLE`) and points at `/eplus-hook-verification:verify-hooks --static --live`
  as a later prompt; it no longer claims the counts were taken.
- eplus-punch-reports has no hooks, so it was missing from the plugin list and the
  model called it "only a stale cache copy". 0.1.2 lists installed plugins without
  hooks (`Installed, no hooks to replay`) in the summary, `report.md`,
  `summary.json` and `inventory.json`.
- Subagent hand-backs reach `UserPromptSubmit` as queued prompts starting with
  `<agent-message from="...">`. 0.1.2 never treats one as a trigger.

## Versioning

Version lives in the catalog's `.claude-plugin/marketplace.json` entry; bump it
whenever a change should reach seats.
