# eplus-verification-catalog

Claude Cowork plugin marketplace `eplus-verification`: EPLUS verification tooling,
kept apart from the production catalogs (`eplus-plugin-catalog`, `vo-plugin-catalog`)
so that test suites can never reach a production profile by accident. Registered on
testing profiles only, through the desktop bootstrap config.

| Plugin | Version | What it does |
|---|---|---|
| [eplus-hook-verification](plugins/eplus-hook-verification/) | 0.1.1 | `/eplus-hook-verification:verify-hooks` replays every hook of every installed EPLUS plugin on the seat and writes the results into the session export. |
| [eplus-acceptance](plugins/eplus-acceptance/) | 0.1.0 | `/eplus-acceptance:run` exercises each plugin live in the session (reports, workers, connector calls) and writes an evidence-backed verdict report. No hooks. |

Order on a seat: run `verify-hooks` first, then `acceptance`; the second uses the
first's result as its card 1.

See [CLAUDE.md](CLAUDE.md) for why this repository exists, how it is delivered, and the
rules for working in it.
