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

Unverified on a seat. First export to check after enabling on the testing profile:
which trigger event carried the run (`summary.json` `trigger_event`), whether the
300 s `timeout` was honoured, whether `CLAUDE_CODE_SESSION_ID` and
`CLAUDE_CODE_PLUGIN_CACHE_DIR` reached the hook (`env_probe`), which install layout
was discovered (`inventory.json`), and whether the run folder made it into the zip.

## Versioning

Version lives in the catalog's `.claude-plugin/marketplace.json` entry; bump it
whenever a change should reach seats.
