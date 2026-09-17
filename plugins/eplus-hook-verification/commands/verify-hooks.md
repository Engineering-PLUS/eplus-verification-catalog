---
description: Replay every hook of every installed EPLUS plugin on this seat against fixtures and expectations, and write the results into the session folder so the session export carries them. Arguments - all | <plugin> [<plugin>...] | <plugin>:<Event> | --static | --live | --limit=N
argument-hint: "[all | <plugin> | <plugin>:<Event>] [--static] [--live] [--limit=N]"
disable-model-invocation: true
---

EPLUS-HOOK-VERIFY: $ARGUMENTS

You are running the EPLUS hook verification suite. The suite itself runs on the
Windows host inside a plugin hook; you cannot run it from bash and must not try.
Three hooks can pick up this request: the prompt-submit hook, the prompt-expansion
hook, and a PreToolUse hook on the Glob tool. Do the following, in order:

1. If a `[eplus-hook-verification] Run ... finished` block is already in your
   context (injected by a hook alongside this prompt), skip to step 3.
2. Otherwise call the Glob tool exactly once with
   `pattern` = `EPLUS-HOOK-VERIFY $ARGUMENTS` (use `EPLUS-HOOK-VERIFY all` when
   no arguments were given) and `path` = your working directory. The pattern
   matches no file; the PreToolUse hook on that call runs the suite and returns
   the summary next to the tool result. Expect it to take one to four minutes.
   If the tool result arrives with no `[eplus-hook-verification]` block, say so
   plainly: the plugin's hooks are not loaded on this seat (check the export's
   cli-diagnostics for `load_plugin_hooks`), and stop.
3. Reproduce the injected block verbatim in a fenced code block labeled
   `hook-verification` at the top of your reply.
4. Read the `report.md` path the block names with the Read tool (it is a host
   `C:\` path; never bash). Summarise every FAIL, ERROR, WARN and NOT_WIRED row
   with its codes in a short table. If the run status is INCOMPLETE, say which
   plugins were not reached and suggest rerunning per plugin.
5. Do not rerun the suite, do not edit any plugin, and do not delete the run
   folder. The user reruns with the same command; each run gets its own folder
   under `<session folder>\hook-verification\<run id>\` and appends to
   `hook-verification.log`, both of which the session export zips.

Arguments: `all` (default), one or more plugin names, `<plugin>:<Event>` for one
event of one plugin, `--static` for the file and wiring checks only (no replay),
`--live` to also count which hooks have fired organically in this session from
the transcript, `--limit=N` to cap the number of replay cases.
