# Enterprise Setup — Configuring and Engineering the SOC Investigation Toolkit

This guide is for the platform/SecOps engineer standing the toolkit up for a team, not for an
individual analyst's first run (that is [`ANALYST-ONBOARDING.md`](ANALYST-ONBOARDING.md)). It walks
identity, least-privilege API clients, secrets, source control, workstation baseline, bootstrap,
validation, operations and governance, in the order dependencies require.

Time budget: roughly one working day for a pilot of two analysts once the RBAC tickets clear.

---

## Step 0 — Decide ownership and scope (30 min)

| Role | Owns | Typical team |
|---|---|---|
| **Platform owner** | The fork of this repo, CI, release/signing, hygiene gate | SecOps Engineering |
| **Identity admin** | Entra groups, RBAC, Conditional Access exceptions, Purview roles | IAM |
| **Integration owners** | CrowdStrike API client, Slack app, ADO PAT policy, LLM API keys | EDR admin / Collaboration / DevOps / AI governance |
| **Analysts** | Case workspaces on their own workstations | SIR / SOC |

Decisions to record before you start:

1. Which environments: production Sentinel only, or `prod` + `staging` + `dev` profiles.
2. Which tools are in scope for the pilot (recommended: `inv-hunt-mcp` + `sentinel-hunt`,
   `sir-case`, `inv-archive-triage`, `purview-dl`; add ingest tools per integration approval).
3. Whether multi-provider skills (`trinity-*`, `hunt-loop`) are allowed. They send case context to
   Claude, OpenAI and Google APIs — this needs a data-egress / AI-governance sign-off.
4. Workstation platforms: macOS, Windows (standard-user, no WSL), Linux.

---

## Step 1 — Identity and access in Entra ID (IAM ticket; 1–3 days lead time)

1. **Create a security group** `sg-soc-toolkit-analysts` (and optionally `-engineers`).
2. **Sentinel / Log Analytics RBAC** on each workspace profile the MCP will target:
   - `Microsoft Sentinel Reader` and `Log Analytics Reader` on the workspace (or resource group).
   - The MCP refuses to start without them; nothing in this toolkit needs Contributor.
3. **Defender XDR Advanced Hunting** (for `inv-hunt-mcp` XDR routing and `inv-xdr-hunt`):
   - `Security Reader` in Entra, or a Defender XDR custom role with *Security data basics (read)* and
     *Advanced hunting* permissions.
4. **Microsoft Graph** (for `inv-entra-triage`, `inv-entra-attr-snapshot`, `inv-soc2-evidence`):
   - Delegated, read-only: `IdentityRiskyUser.Read.All`, `IdentityRiskEvent.Read.All`,
     `AuditLog.Read.All`, `Directory.Read.All`, `Application.Read.All`, `User.Read.All`,
     `UserAuthenticationMethod.Read.All` (attr-snapshot). Consent via the Microsoft public client the
     tools use (Azure CLI / Graph PowerShell client IDs are public constants documented in each README).
5. **Purview eDiscovery** (for `purview-dl`): the analyst must hold **eDiscovery Manager** (own cases)
   or **eDiscovery Administrator** (all cases). Tenant admin consents `eDiscovery.Read.All` for the
   Graph PowerShell public client the first time.
6. **Conditional Access:** the tools use **device-code** and **interactive `az login`** flows. If CA
   blocks device-code for the tenant, add an exception scoped to `sg-soc-toolkit-analysts` on compliant
   devices rather than disabling the policy.
7. **Custom Log Analytics table** (only if deploying `inv-entra-attr-snapshot`): grant `Log Analytics
   Contributor` on the *target* workspace to the service identity that runs the daily snapshot, not to
   analysts.

Validation: `az login --tenant <TENANT-UUID>` then
`az monitor log-analytics workspace show -g <rg> -n <ws>` succeeds for a pilot analyst.

---

## Step 2 — Least-privilege third-party API clients (integration owners)

Each ingest tool ships an install runbook and a preflight that **refuses write scopes** (exit `8`) and
**fails closed** on a missing scope list (exit `3`). Provision exactly what the runbook lists.

| Integration | Create | Scopes | Runbook |
|---|---|---|---|
| CrowdStrike Falcon | OAuth2 API client `inv-falcon-ingest-<team>-readonly` in the Falcon console | `Detects:read`, `Hosts:read`, `Incidents:read` only | `tools/inv-falcon-ingest/docs/falcon-app-install-runbook.md` |
| Slack | Slack App with a **bot** token installed to the workspace; invite the bot to each evidence channel | `channels:history`, `channels:read`, `groups:history`, `groups:read`, `users:read` | `tools/inv-slack-ingest/docs/slack-app-manifest.md` (manifest included) |
| Azure DevOps | PAT scoped to the org, 30–90 day expiry, owned by a service account | `vso.auditlog`, `vso.code`, `vso.identity`, `vso.threads`, `vso.work` (all *Read*) | `tools/inv-ado-ingest/docs/ado-pat-install-runbook.md` |
| GitHub (SOC 2 evidence) | Fine-grained PAT, read-only on the audited orgs | repo/org metadata *read* | `tools/inv-soc2-evidence/README.md` |
| LLM providers (optional) | Anthropic / OpenAI / Google keys owned by the team, with spend caps | n/a | `reference/onepassword-bootstrap-template.md` |

Rules:
- Name clients so their purpose is obvious in the vendor's audit log.
- Set expiry on every token and add it to the rotation calendar (Step 9).
- Salesforce triage (`sf-triage`) and all `*-triage` pipelines need **no** credentials — they run
  offline on exported CSVs.

---

## Step 3 — Secrets management (30 min)

1. **Never** put tokens in `~/.config/sentinel-hunt/config.toml`; it holds routing UUIDs only.
2. Standard on **1Password CLI** (or map the same layout onto your enterprise vault):
   - Vault: `SIR-team` (override with `SIR_1P_VAULT`).
   - Item `LLM-API-tokens` (API Credential): `anthropic_api_key`, `openai_api_key`, `gemini_api_key`.
   - Item `Azure-Sentinel-UUIDs` (Secure Note): `tenant_id`, `subscription_id`, `workspace_id`,
     `resource_group`.
   - `scripts/quickstart.sh` / `.ps1` hydrate these automatically when `op whoami` succeeds.
3. Per-tool credentials go in the tool's own `config.toml` (`chmod 600`), created from
   `config.toml.example`, or in environment variables the README names (for example
   `FALCON_CLIENT_SECRET`, `SLACK_BOT_TOKEN`, `ADO_PAT`). Tools never echo secrets; tests enforce it.
4. Purview refresh tokens are cached at `~/.cache/purview-dl/msal_cache.bin` (`0600`) —
   `%LOCALAPPDATA%\purview-dl\` on Windows. Delete on offboarding.
5. Nothing under `~/SIR/investigations/` is ever synced to a shared drive or backed up to a location
   without the same access control as the evidence itself.

---

## Step 4 — Source control and CI (platform owner; 1 h)

1. Fork or import into your GitHub organization as `soc-investigation-toolkit`.
2. Enable on `main`: branch protection (PR required, CI status checks required, no force-push),
   **secret scanning + push protection**, Dependabot alerts, signed commits recommended.
3. The shipped workflow `.github/workflows/ci.yml` runs an Ubuntu / macOS / Windows matrix with
   SHA-pinned actions and needs **no** cloud credentials (credentialed tests are stubbed/skipped).
   Keep pins current per `engineering/docs/SUPPLY-CHAIN-PINS.md`.
4. **Tailor the hygiene gate** before the first internal commit: add `re:` lines to
   `reference/repo-hygiene-banned-strings.txt` for your mail domain, tenant domain, internal DNS
   suffixes and case-ID shape. Run `bash engineering/scripts/audit-no-case-data.sh --all` — it must
   exit 0 on a clean tree and fail when you plant a test string.
5. Add `.github/CODEOWNERS` (engineering owns `engineering/**`, `tools/**/preflight*`, the gate and
   CI).
6. Optional: a release workflow that builds the Go tools (`engineering/scripts/build-go.sh`),
   **code-signs** them, and publishes `SHA256SUMS`. On locked-down Windows fleets the Defender ASR rule
   *Block executable files unless they meet prevalence/age/trusted-list criteria* will block unsigned
   binaries — signing is the durable fix; a per-hash allow-list is the interim one.

---

## Step 5 — Workstation baseline (endpoint/MDM; 1 h to define)

| Platform | Baseline |
|---|---|
| **macOS** | Homebrew allowed; `python@3.13`, `pipx`, `azure-cli`, `coreutils` (`gtimeout`), `1password-cli`, Git, Claude Code. `caffeinate` is used by unattended loops. |
| **Windows** (standard user, no WSL) | `winget` allowed; `Python.Python.3.13`, `Microsoft.AzureCLI`, `AgileBits.1Password.CLI`, Git; PowerShell execution policy `RemoteSigned` for CurrentUser (or use the `.cmd` shims); **Developer Mode** or junction fallback for skill symlinks (`install-skills.ps1` handles both); prefer the prebuilt signed Go binaries for `inv-entra-triage`, `inv-xdr-hunt`, `inv-soc2-evidence`, `inv-falcon-ingest`. |
| **Linux** | Distro Python ≥ 3.11, `pipx`, `azure-cli`, Git. |
| **Engineers only** | Go toolchain (version in `go.work`), `pre-commit`, Codex / Gemini CLIs if using `trinity-*`. |

Egress allow-list (proxy/ZTNA): `login.microsoftonline.com`, `graph.microsoft.com`,
`management.azure.com`, `api.loganalytics.io`, `api.security.microsoft.com`,
`compliance.microsoft.com`, your Falcon API base (`api.crowdstrike.com` / `api.us-2…` / `api.eu-1…`),
`slack.com`, `dev.azure.com`, `api.github.com`, and the LLM provider endpoints if approved.

---

## Step 6 — Bootstrap a workstation (analyst with engineer on call; 15 min)

```bash
git clone https://github.com/<your-org>/soc-investigation-toolkit.git
cd soc-investigation-toolkit
bash scripts/setup.sh --check     # dry run: shows what would change
bash scripts/setup.sh             # phases A–E (prereqs, pipx, config wizard, MCP register, doctor)
```

Windows: `.\scripts\setup.ps1` or `scripts\setup.cmd` — identical phases.

Then the interactive steps the scripts leave to the human:

```bash
az login --tenant <TENANT-UUID>
op signin                          # only if using 1Password hydration
bash scripts/install-skills.sh     # symlinks skills/ into ~/.claude/skills/
bash scripts/inv-doctor.sh --fix   # applies safe auto-fixes; prints the exact command for the rest
```

Restart Claude Code so it reads the new `~/.claude.json` MCP entry.

`inv-doctor` walks the environment detectors in dependency order (Python → pipx → MCP binary →
config.toml → az + extensions → skills symlinked → MCP registered → `azureProfile.json` BOM-clean →
az session → subscription → workspace identity → hooks). All ✓ is the exit criterion for this step.

---

## Step 7 — Configure profiles and tools (15 min per workstation, or push via MDM)

1. `~/.config/sentinel-hunt/config.toml` (from `tools/inv-hunt-mcp/config.toml.example`):

   ```toml
   [prod]
   tenant_id       = "<ENTRA-TENANT-UUID>"
   subscription_id = "<AZURE-SUBSCRIPTION-UUID>"
   workspace_id    = "<LOG-ANALYTICS-WORKSPACE-UUID>"
   workspace_name  = "sentinel-prod"
   resource_group  = "<RESOURCE-GROUP-NAME>"
   # [staging] / [dev] blocks are opt-in; the MCP never defaults to them.
   ```

   `chmod 600` it. These are routing identifiers, not secrets, but treat the file as internal.
2. Per-tool config from each `config.toml.example` (`inv-falcon-ingest`, `inv-slack-ingest`,
   `inv-ado-ingest`): cloud region, org name, channel list; secrets via env or vault (Step 3).
3. Case workspace root: default `~/SIR/investigations/`. To relocate (for example to an encrypted
   volume) set `SIR_CASE_DIR` in the shell profile or pass `--case-dir`.
4. `skills/sentinel-hunt/references/00-overview.md` and `01-log-sources/_STATUS.json`: mark each log
   source `connected` / `planned` / `out-of-band` for **your** tenant. Hunt drafting and promotion
   refuse sources that are not connected.
5. Watchlists referenced by hunt scenarios (`approved_ips`, `approved_apps`, tier-0 assets, …): create
   them in Sentinel per `skills/sentinel-hunt/references/07-watchlists.md`; `watchlist_exists` in the
   MCP verifies presence before a hunt runs.

---

## Step 8 — Validate end to end (30 min, pilot analyst)

```bash
bash scripts/inv-doctor.sh                       # all ✓
sir-case init pilot-case --display-name "Pilot"  # creates ~/SIR/investigations/pilot-case/
cd ~/SIR/investigations/pilot-case
purview-dl probe                                 # device-code auth; lists eDiscovery cases
falcon-ingest probe                              # scope preflight; exits 8 on any write scope
slack-ingest --dry-run                           # bot-token preflight
ado-ingest probe                                 # PAT scope preflight
```

In Claude Code from inside the workspace: ask for a simple hunt ("sign-ins from new IPs in 24h"); the
`sentinel-hunt` skill drafts KQL, `kql_validate` checks it, `kql_query` runs it, results land at
`~/.local/state/sentinel-hunt/results/`. Then:

```bash
sir-case hunts coverage                          # ATT&CK coverage of the ledger
/triage-archive tools/inv-archive-triage/tests/fixtures/worked_example_sanitized.zip   # in Claude Code
python -m pytest tools/inv-case-init tools/inv-hunt-mcp -q                             # offline suites
GOWORK=off go test ./...                         # engineers, in each Go module
```

Exit criteria: doctor green, one hunt executed read-only, one triage report produced, tests green.

---

## Step 9 — Operationalize (platform owner + analysts)

1. **Scheduled jobs** (per case, on the analyst's workstation or a dedicated runner):
   `tools/inv-daily-timeline/docs/launchd.plist.example` and `scheduled-task.xml.example`; same for
   `inv-case-init` overnight runs and `inv-entra-attr-snapshot` daily snapshots. Runner identity needs
   only the RBAC in Step 1.
2. **Unattended hunt loop:** `bash scripts/hunt-loop.sh <case> --draft-only` for a first week
   (human approval per hunt), then `draft+execute` once the team trusts the approval lock. Stop with
   `touch <case>/reports/hunt-loop/STOP`.
3. **Retention:** `inv-daily-timeline` prunes to 14 days by default; align with your evidence
   retention policy, and archive closed case workspaces to controlled storage with `sir-case close-out`.
4. **Data egress review:** `tools/inv-slack-ingest/docs/data-egress.md` documents exactly what the
   classify pass sends to an LLM (redacted, PII-filtered); mirror that review for `trinity-*` skills.
5. **SOC 2 evidence cadence:** `inv-soc2-evidence collect` on a schedule; store the run directory and
   `manifest.json` where auditors can verify hashes offline.
6. **Monitoring the tools themselves:** every tool writes a run `manifest.json`; alert if a scheduled
   run's manifest is missing or its `exit_code` ≠ 0.

---

## Step 10 — Governance and maintenance

| Activity | Cadence | Reference |
|---|---|---|
| Rotate Falcon client secret, Slack bot token, ADO PAT, LLM keys | 90 days or on personnel change | `tools/inv-falcon-ingest/docs/token-rotation-runbook.md`, `tools/inv-slack-ingest/docs/token-rotation-runbook.md` |
| Offboard an analyst | Same day | Remove from `sg-soc-toolkit-analysts`; revoke PATs/tokens they held; delete `~/.cache/purview-dl/`, `~/.config/sentinel-hunt/`, `~/.claude.json` MCP entry; archive or wipe `~/SIR/investigations/` |
| Refresh MITRE ATT&CK snapshot | Quarterly | `tools/inv-hunt-mcp/docs/mitre-attack-refresh.md` |
| Update SHA-pinned actions and Go/Python deps | Monthly | `engineering/docs/SUPPLY-CHAIN-PINS.md`, Dependabot |
| Re-run the hygiene audit on the whole tree | Every release | `bash engineering/scripts/audit-no-case-data.sh --all` |
| Review read-only preflight forbidden-scope lists against vendor scope changes | Quarterly | `FORBIDDEN_*` constants in each tool's `preflight.py` / `preflight.go` |
| Verify Go binary signatures on the fleet | Every release | Release workflow `SHA256SUMS` |

---

## Step 11 — Rollout plan

1. **Week 1 — pilot:** two analysts, `prod` profile only, hunt MCP + archive triage + `sir-case`.
   Collect doctor output and friction notes.
2. **Week 2 — ingest tools:** enable Falcon / Slack / ADO ingest after integration owners confirm
   scopes and rotation ownership. Run `probe` for each on every pilot workstation.
3. **Week 3 — automation:** overnight loop in `--draft-only`, daily timeline scheduled, egress review
   signed for any LLM-backed step.
4. **Week 4 — team:** MDM-push the workstation baseline, publish the internal fork's `QUICKSTART`,
   run a 30-minute onboarding per analyst using `ANALYST-ONBOARDING.md`, hand ownership of the
   rotation calendar to the integration owners.

---

## Quick checklist

- [ ] Roles and scope recorded (Step 0)
- [ ] Entra group + Sentinel/Log Analytics Reader + Defender/Graph/Purview roles granted; CA exception scoped (Step 1)
- [ ] Falcon / Slack / ADO / GitHub clients created read-only, expiries set (Step 2)
- [ ] Vault layout populated; no secrets in config.toml (Step 3)
- [ ] Fork created; branch protection, secret scanning, CI green; banned-strings tailored (Step 4)
- [ ] Workstation baseline and egress allow-list approved (Step 5)
- [ ] `setup` + `az login` + `install-skills` + `inv-doctor` green on pilot machines (Step 6)
- [ ] `config.toml` profiles, log-source status, watchlists configured (Step 7)
- [ ] Probes pass; one hunt and one triage run end to end; tests green (Step 8)
- [ ] Schedules, retention, egress review, evidence cadence in place (Step 9)
- [ ] Rotation calendar and offboarding procedure owned (Step 10)
