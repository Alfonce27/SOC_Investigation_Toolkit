# Quickstart

Clone → run setup → ready, in about 5 minutes (plus whatever your Azure RBAC ticket takes).

For an opinionated 30-minute first-run walkthrough with a worked example, see
[`docs/ANALYST-ONBOARDING.md`](docs/ANALYST-ONBOARDING.md). For deploying the toolkit for a team
(identity, least-privilege API clients, secrets, CI), see
[`docs/ENTERPRISE-SETUP.md`](docs/ENTERPRISE-SETUP.md). For the visual map of how the tools, skills
and shared scripts fit together, open `docs/workflows.html` in any browser.

## macOS / Linux

```bash
git clone https://github.com/Alfonce27/SOC_Investigation_Toolkit.git
cd SOC_Investigation_Toolkit

bash scripts/setup.sh
```

Interactive prompts the first time (config UUIDs); silent on re-runs. To accept all `az`-sourced
defaults without prompting:

```bash
bash scripts/setup.sh --yes
```

To preview what setup would do without touching anything:

```bash
bash scripts/setup.sh --check
```

## Windows (PowerShell or cmd.exe — no WSL required)

```powershell
git clone https://github.com/Alfonce27/SOC_Investigation_Toolkit.git
cd SOC_Investigation_Toolkit

.\scripts\setup.ps1
```

`cmd.exe` users:

```cmd
scripts\setup.cmd
```

## What just happened

`setup.sh` / `setup.ps1` walks five idempotent phases:

| Phase | What it does |
|---|---|
| A — prereqs | Probes Python ≥ 3.11, pipx, Azure CLI. Installs `python@3.13` via brew (macOS) or `Python.Python.3.13` via winget (Windows) if missing. Bootstraps pipx if missing. |
| B — pipx install | Installs every `tools/inv-*` package via `pipx install --python <resolved>`. Sidesteps the macOS-system-Python-is-3.9 / `requires-python = ">=3.11"` mismatch automatically. |
| C — config wizard | Walks you through `~/.config/sentinel-hunt/config.toml`. Pre-fills `tenant_id` and `subscription_id` from `az account show`; offers menus for `resource_group` and `workspace_id` from `az group list` and `az monitor log-analytics workspace list` when you're logged in. |
| D — MCP register | Adds the `inv-hunt-mcp` entry to `~/.claude.json` via `tools/inv-hunt-mcp/scripts/register-mcp.py`. |
| E — final doctor | Runs the environment detectors. Any failure surfaces its specific fix command, not just a count. |

## Once setup is green

```bash
az login --tenant <YOUR-TENANT-UUID>   # the UUID you put in config.toml
bash scripts/install-skills.sh         # symlinks skills/ into ~/.claude/skills/
```

Then restart Claude Code (Cmd+Q / Ctrl+Q + reopen) so it picks up the new MCP entry, and you're
done. Try a hunt:

> In Claude Code: "Find Entra sign-ins from new IPs in the last 24h. Show one row per IP."

The `sentinel-hunt` skill drafts the KQL; the MCP runs it read-only; the result lands at
`~/.local/state/sentinel-hunt/results/`.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `pipx install ./tools/inv-hunt-mcp` fails with `Package requires a different Python: 3.9.6` | Run `bash scripts/setup.sh` instead — it passes the right `--python` flag automatically. |
| Windows PowerShell: `cannot be loaded because running scripts is disabled` | Run `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`, or use `scripts\setup.cmd` (cmd.exe, no execution-policy gate). |
| `az login` redirected to the wrong tenant | `az login --tenant <UUID-from-config.toml-tenant_id>`. |
| `~/.claude.json` doesn't pick up the MCP | Restart Claude Code (Cmd+Q on macOS, Ctrl+Q on Linux/Windows + reopen). The config is read at session start. |
| Setup wrote config.toml but workspace probe fails | Confirm your `Microsoft Sentinel Reader` + `Log Analytics Reader` RBAC on the workspace. The MCP is read-only by design and refuses to start without them. |

For the full reference, see [`README.md`](README.md). For the case-and-launch flow (clone → setup →
case workspace → continuous hunt-loop), see `scripts/quickstart.sh`.
