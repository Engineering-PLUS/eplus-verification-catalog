# eplus-acceptance

The live acceptance suite for the EPLUS plugins: one command that has the model
exercise each installed plugin in a real Cowork session and write a verdict report
with evidence, so a person can decide whether to advance a profile pin without taking
anyone's word for it.

No hooks. A skill and a command only, so the plugin costs nothing while idle and can
stay enabled on testing profiles.

| Component | Path | Purpose |
|---|---|---|
| Command | [`commands/run.md`](commands/run.md) | `/eplus-acceptance:run [all | --quick | <plugin>] [--egress-host <host>] [--punch] [--gate]` |
| Skill | [`skills/plugin-acceptance/SKILL.md`](skills/plugin-acceptance/SKILL.md) | The check cards per plugin, the evidence rules, the budget rules, the report format |

## What a run does

1. Uses the `eplus-hook-verification` replay result already in the session (the user
   runs `/eplus-hook-verification:verify-hooks all --live` first) as card 1.
2. error-reporting: identity note present, deliberate tool failure produces the nudge,
   one report from the main thread, one report from a Sonnet worker (the subagent
   case the 0.4.0 stamp exists for), optional egress block against a host you name.
3. eplus-model-routing: routing note present, one Haiku worker round trip, spawn gate
   only on request (it needs a click).
4. eplus-rfis-submittals: one read-only connector call, researcher echo file present;
   the commit gate is never exercised (it would prompt and write).
5. eplus-punch-reports and eplus-office-skills: presence only; the punch smoke test
   on request.

The report lands as `acceptance-report.md` in the session evidence folder (zipped
into the export next to `hook-verification.log`) and in the session outputs folder.

## What it deliberately does not prove

Things the model cannot see from inside the session are marked NOT_OBSERVABLE with
the pointer to where they can be seen: the backend record for the two test reports
(does `requested_by` carry the seat identity for the subagent's report?), and the
export's hook attachments for silent hooks. Two test reports are filed per run, both
category `other`, message starting `acceptance test`.

## Cost

Designed for an expensive main-thread model: under 40 tool calls, two subagents
(one Haiku, one Sonnet; a third, the RFI researcher, only with `all`), no raw log or
transcript reads. A `--quick` run is cards 1 to 3 only.
