# eplus-verification-catalog

Claude Cowork plugin marketplace `eplus-verification`: EPLUS verification tooling,
kept apart from the production catalogs (`eplus-plugin-catalog`, `vo-plugin-catalog`)
so that test suites can never reach a production profile by accident. Registered on
testing profiles only, through the desktop bootstrap config.

| Plugin | Version | What it does |
|---|---|---|
| [eplus-hook-verification](plugins/eplus-hook-verification/) | 0.1.0 | `/eplus-hook-verification:verify-hooks` tests every hook of every installed EPLUS plugin on the seat and writes the results into the session export. |

See [CLAUDE.md](CLAUDE.md) for why this repository exists, how it is delivered, and the
rules for working in it.
