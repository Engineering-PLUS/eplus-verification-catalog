# eplus-verification-catalog

## Why this repository exists

EPLUS ships two plugin marketplaces to Claude Cowork seats: `eplus-plugin-catalog`
(production, `main` stable and `dev` for a subset) and `vo-plugin-catalog` (Victor's
experiments). Both are delivered by the desktop bootstrap config, which pins each user
profile to a branch and commit. Anything in those catalogs is a candidate to reach a
real seat.

Test tooling does not belong there. A test suite that fires hooks on every prompt, or
that replays every plugin's hooks with synthetic payloads, must never be one config
mistake away from a production profile, and it must not clutter the production
marketplaces' plugin lists. So verification tooling lives here, in its own marketplace
named `eplus-verification`, registered only on testing profiles.

This repo holds the tools that verify the other two catalogs. It never holds a
production feature.

## What is in it

| Plugin | Purpose |
|---|---|
| `plugins/eplus-hook-verification` | `/eplus-hook-verification:verify-hooks` replays every hook wiring of every installed plugin on the seat against fixtures and expectations, on the Windows host, and writes the results into the session folder the exporter zips. See its README and skill. |
| `plugins/eplus-acceptance` | `/eplus-acceptance:run` is the live layer: the model exercises each plugin for real in the session (test reports, workers, connector reads, presence) and writes `acceptance-report.md` with one evidence-backed row per check. No hooks. Run after `verify-hooks`. |

The old always-on generic logger (`hook-testing-plugin` in vo-plugin-catalog) stays
where it is, disabled, as reference. Do not edit it; copy from it.

## How it reaches a seat

Bootstrap only, no terminal commands. The testing profile's settings payload registers
the marketplace and enables the plugin:

```json
{
  "extraKnownMarketplaces": {
    "eplus-verification": {
      "source": { "source": "github", "repo": "Engineering-PLUS/eplus-verification-catalog" }
    }
  },
  "enabledPlugins": {
    "eplus-hook-verification@eplus-verification": true,
    "eplus-acceptance@eplus-verification": true
  }
}
```

Plugin sources are relative paths inside this repo (`./plugins/<name>`); an object
source (`github`, `url`) is treated as external by managed deployments and not
installed. Marketplace names `org`, `org-provisioned` and `unknown` are rejected by
Desktop's managed sync; keep the name `eplus-verification`.

## Rules that apply here

- **Hooks run on the Windows host under PowerShell 5.1**, never in the Linux VM. Every
  hook is a single `& "${CLAUDE_PLUGIN_ROOT}\scripts\x.ps1"` call with
  `"shell": "powershell"`. Scripts are ASCII, no BOM, LF endings, always `exit 0`, the
  decision travels in the JSON body, one `EPLUS_NO_*` escape hatch each.
- **Seats refresh a plugin only when its `version` in `marketplace.json` increments.**
  Do not bump without being asked.
- **Evidence comes from session exports** (`G:\Software\VO\Claude Log Debugging\exports`,
  viewed in Claude Log Lens or with the `claude-export-forensics` skill). Never search
  the dev machine for test-run artifacts.
- **Commit, bump and push only on Victor's explicit instruction**, each one separately.
- **One folder per repository.** Switch branches in place; never create worktrees or a
  second checkout folder.
- Before every bump, run the suite on the dev box against both production catalogs:

```
powershell -NoProfile -ExecutionPolicy Bypass -File plugins\eplus-hook-verification\scripts\run-suite.ps1 -PluginsRoots "<eplus-plugin-catalog>\plugins;<vo-plugin-catalog>\plugins" -Selection all -OutDir %TEMP%\hv
```

## Where things are documented

- `plugins/eplus-hook-verification/README.md`: architecture, triggers, artifacts, field status.
- `plugins/eplus-hook-verification/skills/hook-verification/SKILL.md`: how to read a run, verdict codes, expectations schema, how to add coverage.
- `plugins/eplus-hook-verification/expectations/<plugin>.json`: the cases per production plugin. When a production hook changes, its expectations change here in the same batch.
