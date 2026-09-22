---
description: Run the EPLUS plugin acceptance suite in this Cowork session and write the verdict report to the session evidence folder and the outputs folder. Arguments - all | --quick | <plugin name> [--egress-host <host>] [--punch] [--gate]
argument-hint: "[all | --quick | error-reporting | eplus-model-routing | eplus-rfis-submittals | eplus-punch-reports] [--egress-host <host>] [--punch] [--gate]"
disable-model-invocation: true
---

Load the `plugin-acceptance` skill from the eplus-acceptance plugin and run it with
these arguments: `$ARGUMENTS` (default `all`).

Before the first check, confirm in one line which cards you will run and which flags
were given. If the `[eplus-hook-verification] Run ... finished` block is not already
in this session's context, ask the user to run
`/eplus-hook-verification:verify-hooks all --live` first and stop; card 1 depends on
it and you must not run that suite yourself.

Flags: `--egress-host <host>` enables check 2e against that host; `--punch` enables
5b; `--gate` enables the manual spawn-gate check 3c (the user will see a permission
prompt and should decline it).

Finish with the report exactly in the skill's format, written to both locations,
then reproduced as your reply. Nothing else in the reply.
