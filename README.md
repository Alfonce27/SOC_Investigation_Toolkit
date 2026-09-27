# SOC Investigation Toolkit

Python and Go tooling plus Claude Code skills for a Security Incident Response (SIR) / SOC team's
acquisition, triage, hunting and hand-off workflows against a Microsoft-centric estate
(Sentinel, Defender XDR, Entra ID, Purview) with adjacent sources (CrowdStrike Falcon, Slack,
Azure DevOps, Salesforce Experience Cloud, AWS CloudTrail).

The repo is structured so an analyst can `git clone` it once and reach a working end-to-end
**acquire → triage → hunt → hand off** loop in under 30 minutes. Each tool is self-contained under
`tools/`; each in-repo Claude skill lives under `skills/`; repo-wide setup and hygiene helpers live
under `scripts/`; engineering-only surface is quarantined under `engineering/`.

> **First time here?** Read [`docs/ANALYST-ONBOARDING.md`](docs/ANALYST-ONBOARDING.md) — the
> 30-minute guided first run (install, auth, one worked example).
>
> **Deploying for a team?** Read [`docs/ENTERPRISE-SETUP.md`](docs/ENTERPRISE-SETUP.md) — identity,
> RBAC, least-privilege API clients, secrets, CI and rollout.
>
> **Want the map before the manual?** Open [`docs/workflows.html`](docs/workflows.html) in any
> browser (no install, opens from `file://`) for a one-page visual of how the tools, skills and shared
> scripts fit together. Generated from `docs/workflows_manifest.json`.

> **Provenance.** This is a sanitized public release of an internal toolkit. Sprint planning history,
> worklogs and all case-derived fixtures were removed; every identifier in this repo is a documented
> placeholder (`example.com`, `contoso.onmicrosoft.com`, RFC 5737 IPs, all-zero GUIDs).

---

## Design principles

- **Read-only by construction.** Every ingest tool runs a scope preflight and *refuses to start* if
  its token carries a write, manage, admin or create scope (exit code `8`). Missing scope lists fail
  **closed**. The MCP server only exposes read-only KQL.
- **Evidence never lives in the repo.** Durable case state lives off-repo under
  `~/SIR/investigations/<case>/`, managed by the `sir-case` CLI. `.gitignore` rejects evidence-shaped
  files; a banned-strings audit gate runs in pre-commit and CI.
- **Local and deterministic where possible.** Triage pipelines make no network calls, emit stable
  filenames (`00_SECURITY_TRIAGE_REPORT.md`, `01_FINDINGS_INVENTORY.{csv,json}`, `00_NEXT_HUNTS.md`)
  and produce byte-identical output for identical input.
- **Secrets stay in your vault.** Config files hold only routing identifiers (tenant / subscription /
  workspace UUIDs). API tokens come from environment variables, an optional 1Password CLI hydration
  step, or per-tool `0600` config files that the tools never echo.
- **Stdlib-first.** Python tools target 3.11+ with minimal third-party deps; four tools are single
  static Go binaries for locked-down Windows workstations.

---

## Tools

| Tool | What it does | Layout | Language |
|---|---|---|---|
| [`tools/inv-activity-log-triage/`](tools/inv-activity-log-triage/) | Phase-1 IR triage of Sentinel / Entra / Office / Graph / VPN / Purview **activity-log** CSV exports. Subject timeline, SPN activity, confirmed downloads, credential-bait filenames, per-user exposure, MFA/password-reset context, IOC export, prioritized hunt list and a one-page IC brief in < 10 s. | A | Python |
| [`tools/inv-purview-edisco-triage/`](tools/inv-purview-edisco-triage/) | Triage of Microsoft Purview **eDiscovery** exports (metadata CSVs + extracted content). Surfaces credential leaks, certificates and keys, PE binaries, sensitive-info types; per-File-ID deep dives and Sentinel-ready event JSON. | A | Python |
| [`tools/inv-purview-downloader/`](tools/inv-purview-downloader/) | `purview-dl` CLI: pull files **by filename** from a Purview eDiscovery review set via Graph device-code auth (no app registration). The acquisition step that feeds the two triage tools above. | B | Python |
| [`tools/inv-archive-triage/`](tools/inv-archive-triage/) | Generic, offline archive triage. Ingests any `zip` / `tar` / `tar.gz` / `tgz` (or extracted directory); emits a security-triage report plus a structured findings inventory (secrets, certs, PE metadata, PII, nested archives). Backs the `triage-archive` skill. | A | Python |
| [`tools/inv-case-init/`](tools/inv-case-init/) | `sir-case` CLI: durable off-repo case workspace (`case.toml`), hunt ledger, approval lock, bundle/merge of hunt packs, coverage stats, and an unattended **overnight** hunt executor. | B | Python |
| [`tools/inv-hunt-mcp/`](tools/inv-hunt-mcp/) | Local MCP server exposing read-only KQL tools (`kql_query`, `kql_query_async`, `kql_validate`, `kql_get_schema`, `watchlist_exists`) against **both** Sentinel / Log Analytics and Defender XDR Advanced Hunting using the analyst's `az login` identity. Routes per-query by table name. | B | Python |
| [`tools/inv-slack-ingest/`](tools/inv-slack-ingest/) | `slack-ingest` CLI: pulls history + threaded replies from named private channels into per-channel JSONL bundles; optional LLM **classify** pass rolls threads up into a `00_TASKS_SUMMARY.md`. Read-only bot-token preflight. | B | Python |
| [`tools/inv-daily-timeline/`](tools/inv-daily-timeline/) | Scheduler-driven orchestrator: `slack-ingest --incremental` → assemble manifest → `inv-timeline-render` → atomic dated-report pointer swap → 14-day prune. Ships launchd and Windows Task Scheduler examples. | A | Python |
| [`tools/inv-timeline-render/`](tools/inv-timeline-render/) | Single-page, self-contained incident-timeline HTML for wider audiences. Reads triage outputs, hunt JSON, Slack bundles and analyst-authored markdown prose via a TOML manifest. | A | Python |
| [`tools/inv-falcon-ingest/`](tools/inv-falcon-ingest/) | `falcon-ingest` CLI: CrowdStrike Falcon **detections + incidents + hosts** to JSONL. Stdlib OAuth2 client-credentials; read-only scope preflight; compound `(cid, detection_id)` join key for dedup against the Sentinel connector. Python and Go implementations with a parity harness. | B | Python + Go |
| [`tools/inv-ado-ingest/`](tools/inv-ado-ingest/) | `ado-ingest` CLI: the Azure DevOps datasets the Sentinel `AzureDevOpsAuditing` connector does **not** carry — pull requests (self-approval, bypassed policies), PATs, branch policies, service connections, opt-in repo secret scan (redacted match + offset only). PAT read-only preflight. | B | Python |
| [`tools/inv-ado-enum/`](tools/inv-ado-enum/) | Enumerate every Azure DevOps project and Git repository in an organization (JSON / CSV / NDJSON). Python and Go implementations. | B | Python + Go |
| [`tools/inv-entra-triage/`](tools/inv-entra-triage/) | Read-only, Graph-direct snapshots of Entra identity-threat datasets (risky users, risk detections, device-code sign-ins, illicit consent grants, new app role assignments) to JSONL. Single static Go binary. | — | Go |
| [`tools/inv-entra-attr-snapshot/`](tools/inv-entra-attr-snapshot/) | Daily Entra user-registration attribute snapshot (`*RegisteredTime` fields) into a custom Log Analytics table, closing the historical-depth gap for SSPR / account-recovery hunts. | B | Python |
| [`tools/inv-xdr-hunt/`](tools/inv-xdr-hunt/) | Read-only Defender XDR incident and alert snapshots plus batch advanced-hunting exports to JSONL. Single static Go binary. | — | Go |
| [`tools/inv-soc2-evidence/`](tools/inv-soc2-evidence/) | Read-only SOC 2 evidence collection (Entra, Sentinel, GitHub, ADO) with a SHA-256 integrity manifest an auditor can verify offline. Single static Go binary. | — | Go |
| [`tools/inv-salesforce-triage/`](tools/inv-salesforce-triage/) | `sf-triage` CLI: read-only, zero-network triage of Defender `CloudAppEvents` exports for unauthenticated **Experience Cloud guest-user** document exposure. Decodes `RawEventData`, scores per-actor automation, emits the document-channel logging gap as a first-class finding; ships a detection draft (KQL + Terraform). | B | Python |
| [`tools/inv-powershell-scripts/`](tools/inv-powershell-scripts/) | Small PowerShell / LDAP information-gathering scripts (SPN discovery, nested group membership, password-age report). | — | PowerShell |

Repo-root PowerShell utilities in [`scripts/`](scripts/): an AD password/login report and a Defender for
Cloud Apps archived-activity puller, each with a companion `.md`.

**Layouts.** *A* = script collection (`python3 scripts/run_all.py …`). *B* = installable package
(`pipx install ./tools/<x>/`). Go tools build to a single binary via `engineering/scripts/build-go.sh`.

---

## Skills

Claude Code skills the repo ships. `bash scripts/install-skills.sh` symlinks every directory under
`skills/` into `~/.claude/skills/` and rewrites the `__REPO_ROOT__` placeholder to your clone path.

| Skill | Trigger | What it does |
|---|---|---|
| `skills/triage-archive/` | `/triage-archive <path>` | Drives `inv-archive-triage` and interprets the findings. |
| `skills/sentinel-hunt/` | describe the hunt in a session | Drafts KQL hunts against your Sentinel workspace. Ships an editable log-source inventory (which tables hold which source), 86 hunt scenarios with ATT&CK mapping, a KQL cheat sheet, watchlist and `externaldata` patterns, a detection-as-code promotion flow and a Splunk→KQL translator. Read-only — the analyst (or the MCP) runs the query. |
| `skills/hypothesis-generator/` | `/hypothesis-generator` | Seeds **new** hunts from outside intel (CTI URL, CVE, ATT&CK technique) targeted at coverage gaps from `sir-case hunts coverage`. No outbound HTTP; uses the checked-in ATT&CK snapshot. |
| `skills/trinity-hunt/` | `/trinity-hunt` | 12-agent hunt orchestrator: 4 roles (Hunter / Reviewer / Analyzer / Extender) × 3 providers (Claude / Codex / Gemini). Emits a consensus hunt pack and a schema-validated `next-hunts-additions.json` for `sir-case hunts merge`. |
| `skills/hunt-loop/` | `/hunt-loop` or `bash scripts/hunt-loop.sh <case>` | One-prompt continuous-hunt driver: trinity-hunt → merge → bundle → approve → overnight → HTML dashboard → `loop-state.json`. Iteration N auto-writes its intent from iteration N-1 so the hunt set expands each round. Stop with a `STOP` sentinel file. |
| `engineering/skills/trinity-plan/` | `/trinity-plan SPRINT-NNN` | Engineering-only multi-agent sprint planner (draft → cross-critique → merge). |
| `engineering/skills/trinity-execute/` | `/trinity-execute SPRINT-NNN` | Companion executor. Commits to the declared branch; never pushes, never opens PRs. |

Multi-provider skills need `OPENAI_API_KEY` / `GEMINI_API_KEY` in addition to Claude Code; see
[`docs/ENTERPRISE-SETUP.md`](docs/ENTERPRISE-SETUP.md) for the data-egress review those keys imply.

---

## Choose your tool

| You have… | …use |
|---|---|
| Filenames to pull out of a Purview eDiscovery review set | `purview-dl download …` |
| A Purview eDiscovery export (`Items_*.csv` + `content/`) | `tools/inv-purview-edisco-triage/` |
| Any archive and need a fast security triage | `/triage-archive <path>` (or `tools/inv-archive-triage/`) |
| A directory of activity-log CSVs | `tools/inv-activity-log-triage/` |
| Slack channels to preserve as evidence | `slack-ingest` → `inv-daily-timeline` |
| CrowdStrike / Azure DevOps / Entra / Defender XDR datasets to snapshot | `falcon-ingest`, `ado-ingest`, `inv-entra-triage`, `inv-xdr-hunt` |
| A hunt to draft and run against Sentinel or Defender | `sentinel-hunt` skill + `inv-hunt-mcp` |
| A case to run unattended overnight | `sir-case overnight` / `/hunt-loop` |
| A timeline to hand to leadership | `inv-timeline-render` |
| Auditor evidence with integrity hashes | `inv-soc2-evidence collect` |

---

## Quick start (5 minutes)

```bash
git clone https://github.com/Alfonce27/SOC_Investigation_Toolkit.git
cd SOC_Investigation_Toolkit

bash scripts/setup.sh            # macOS / Linux
# .\scripts\setup.ps1            # Windows PowerShell (no WSL required)
# scripts\setup.cmd              # Windows cmd.exe
```

`setup` walks five idempotent phases:

| Phase | What it does |
|---|---|
| A — prereqs | Probes Python ≥ 3.11, pipx, Azure CLI. Installs Python via brew / winget if missing; bootstraps pipx. |
| B — pipx install | Installs every `tools/inv-*` package with the right `--python`. |
| C — config wizard | Writes `~/.config/sentinel-hunt/config.toml`; pre-fills `tenant_id` / `subscription_id` from `az account show`, offers menus for `resource_group` / `workspace_id`. |
| D — MCP register | Adds the `inv-hunt-mcp` entry to `~/.claude.json`. |
| E — doctor | Runs the environment detectors; every failure prints its specific fix. |

Then:

```bash
az login --tenant <YOUR-TENANT-UUID>      # the UUID you put in config.toml
bash scripts/install-skills.sh            # symlink skills into ~/.claude/skills/
bash scripts/inv-doctor.sh                # should be all ✓
```

Restart Claude Code so it picks up the MCP entry, then try a hunt:

> "Find Entra sign-ins from new IPs in the last 24h. Show one row per IP."

`--yes` accepts `az`-sourced defaults silently; `--check` is a dry run. Full reference in
[`QUICKSTART.md`](QUICKSTART.md); Windows specifics (winget, Developer Mode for symlinks, BOM gotcha) in
[`docs/WINDOWS-QUICKSTART.md`](docs/WINDOWS-QUICKSTART.md).

### Case-focused launcher

`scripts/quickstart.{sh,ps1,cmd} --case <id>` extends setup with optional 1Password hydration (LLM
keys + Sentinel UUIDs from a vault; see
[`reference/onepassword-bootstrap-template.md`](reference/onepassword-bootstrap-template.md)),
`inv-doctor --fix`, `sir-case init`, and an unattended `hunt-loop`. Interactive steps it deliberately
leaves to you: `op signin`, `az login`, and provider CLI sign-ins.

---

## Case workspace

Everything durable lives **off-repo** at `~/SIR/investigations/<case>/`, created by
`sir-case init <case> --display-name "…"`. From inside a workspace every tool defaults its `--output`
automatically. Resolution order: `--case-dir` flag → `$SIR_CASE_DIR` → nearest ancestor `case.toml` →
tool's legacy default.

```
~/SIR/investigations/<case>/
├── case.toml                     # id, display name, remembered Purview ids, approvals
├── downloads/                    # purview-dl output
├── inv-slack-ingest/ inv-falcon-ingest/ inv-ado-ingest/ …   # per-tool JSONL runs
├── reports/
│   ├── archive-triage/run_<ts>/  purview-edisco-triage/run_<ts>/  activity-log-triage/run_<ts>/
│   ├── hunt-loop/dashboard-latest.html
│   └── hypothesis/run_<ts>/
└── 00_HANDOFF.md                 # investigator hand-off (template below)
```

---

## End-to-end analyst workflow

1. **Bootstrap the case** — `sir-case init acme-2026-05 --display-name "ACME storm"` and `cd` into it.
2. **Authenticate** — `purview-dl probe` (device-code flow, first time only; lists visible cases).
3. **Acquire** — `purview-dl download --case <case> --reviewset <rs> --names-file names.txt --remember`;
   `slack-ingest`, `falcon-ingest`, `ado-ingest`, `inv-entra-triage`, `inv-xdr-hunt` as needed.
4. **Triage** — `/triage-archive ./downloads/<archive>.zip`, or run the activity-log / eDiscovery
   pipelines directly. Read `00_SECURITY_TRIAGE_REPORT.md` and `00_NEXT_HUNTS.md`.
5. **Hunt** — walk the hunt list with the `sentinel-hunt` skill; the MCP executes read-only KQL;
   `sir-case hunts merge/bundle/approve/overnight` runs the approved set unattended.
6. **Extend** — `/hypothesis-generator` from new intel, `/trinity-hunt` from case findings,
   `/hunt-loop` to iterate.
7. **Hand off** — `inv-timeline-render` for the audience-facing timeline; `00_HANDOFF.md` for the next
   analyst.

---

## Repo layout

```
SOC_Investigation_Toolkit/
├── README.md  QUICKSTART.md  ONBOARDING.md
├── .github/workflows/ci.yml            # Ubuntu / macOS / Windows matrix, SHA-pinned actions
├── .pre-commit-config.yaml             # Opt-in hygiene gate
├── docs/                               # Analyst onboarding, Windows guide, enterprise setup, workflows map
├── reference/                          # JSON schemas, ATT&CK snapshot, 1Password template, hygiene lists
├── scripts/                            # setup / quickstart / inv-doctor / install-skills / hunt-loop (sh, ps1, cmd)
├── skills/                             # Analyst-facing Claude Code skills
├── tools/                              # One directory per tool (see table)
└── engineering/                        # Engineering-only: bootstrap engine, Go shared lib, parity harness,
    ├── inv-bootstrap/                  #   doctor / setup / audit / verify (stdlib Python)
    ├── go/invkit/                      #   shared Go kit: cliargs, httpkit, graphauth, persist, redact, …
    ├── parity/                         #   Python↔Go golden-tape parity harness
    ├── scripts/                        #   audit-no-case-data, verify-tool-layout, build-go, hooks
    ├── skills/                         #   trinity-plan / trinity-execute
    └── docs/                           #   GO-STANDARDS, SUPPLY-CHAIN-PINS
```

---

## Conventions

- **Two sanctioned tool layouts** (see table). Each tool is self-contained: `cd tools/<tool>/` and run it
  without touching anything outside.
- **Evidence files never commit.** Root `.gitignore` rejects `*.csv`, `*.xlsx`, `*.zip`, `*.tar*`,
  `*.7z`, `*.rar`, `**/reports/`, `**/content/`. Synthetic fixtures under `tests/fixtures/` are
  allow-listed per tool.
- **Hygiene gate.** `engineering/scripts/audit-no-case-data.sh` runs `--staged` in pre-commit and
  `--all` in CI. It fails on banned strings (see
  [`reference/repo-hygiene-banned-strings.txt`](reference/repo-hygiene-banned-strings.txt) — add your
  own tenant patterns there) and on evidence-shaped files outside fixtures.
- **Placeholders are synthetic by construction:** `example.com`, `contoso.onmicrosoft.com`, RFC 5737
  IPs, all-zero GUIDs, `inv000000`. Real-tenant shapes are rejected by the gate.
- **Python ≥ 3.11, stdlib-first. Go builds with `GOWORK=off`** and SHA-pinned GitHub Actions
  (`engineering/docs/SUPPLY-CHAIN-PINS.md`).

Enable the hook:

```bash
bash scripts/install-hooks.sh          # sets core.hooksPath → engineering/scripts/hooks/
# or: pip install pre-commit && pre-commit install
```

---

## Adding a tool or skill

1. `tools/inv-<short-domain>/` with Layout A or B canonical files.
2. Add a row to the **Tools** table above and to `docs/workflows_manifest.json`; regenerate
   `docs/workflows.html` with `engineering/scripts/render-workflows.py`.
3. `bash engineering/scripts/verify-tool-layout.sh` and
   `bash engineering/scripts/audit-no-case-data.sh --all` must exit 0.
4. Skills: `skills/<name>/SKILL.md`; use `__REPO_ROOT__/…` for absolute repo paths; add a row to the
   **Skills** table.
5. Open a PR. CI runs the cross-OS matrix and the hygiene gate.

---

## Investigator hand-off template

Drop a `00_HANDOFF.md` into the case workspace (off-repo):

```markdown
# Case Handoff: <case-id>

**Lead analyst:** <name>   **Status:** Active | Contained | Closed   **Last updated:** YYYY-MM-DD

## Workspace
- Case workspace: `<absolute path>` · Purview review set: `<name or GUID>` · Triage outputs: `<path>/reports/`

## Tooling used (version + run ID)
- purview-dl vX.Y.Z · archive-triage run_<ts> · activity-log-triage run_<ts> · sentinel-hunt <sha>

## Key findings to date
## Open threads for the next analyst
## Next steps
```

---

## Security

- Report vulnerabilities privately via [`SECURITY.md`](SECURITY.md). Do not open public issues for
  security reports.
- Never attach real evidence, logs or screenshots to issues or PRs. Reproduce with the synthetic
  fixtures.
- Every ingest tool's read-only preflight is a safety property, not a convenience — do not weaken it.

## License

See [`LICENSE`](LICENSE). Vendored third-party components keep their original headers; see
`THIRD_PARTY_NOTICES.md`.
