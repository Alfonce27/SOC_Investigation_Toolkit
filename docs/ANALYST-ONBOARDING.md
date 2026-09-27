# Analyst Onboarding (30-minute walkthrough)

First-run guide for a new SIR / SOC analyst. Goal: in under 30 minutes from `git clone`, you should
have `purview-dl` installed and authenticated, the `triage-archive` and `sentinel-hunt` Claude Code
skills loaded, the read-only hunt MCP registered, and one worked example run end-to-end.

> **Skim the visual map first.** Open `workflows.html` in a browser (no install, no server) for a
> one-page reference covering every tool, skill and the canonical case flow. Pair it with this
> walkthrough if you learn better from diagrams than prose.
>
> **Setting this up for a whole team?** That is [`ENTERPRISE-SETUP.md`](ENTERPRISE-SETUP.md). This
> page assumes the RBAC, API clients and vault layout described there already exist.

**macOS-first.** Linux is identical except `~/Library/Caches/` → `~/.cache/`. Windows analysts: the
`.ps1` / `.cmd` shims run natively (no WSL required); see
[`WINDOWS-QUICKSTART.md`](WINDOWS-QUICKSTART.md) for the winget / Developer Mode / BOM specifics.
Inline notes flag where commands differ.

---

## 0 · Prerequisites (3 min)

You need:

- **macOS** (Apple Silicon or Intel), a **Linux** box, or **Windows 10/11**.
- **Python ≥ 3.11.** Check: `python3 --version`. `scripts/setup.sh` installs one via brew / winget
  if missing.
- **pipx.** `setup.sh` bootstraps it; manual install if you prefer:
  - macOS: `brew install pipx && pipx ensurepath`
  - Linux: `python3 -m pip install --user pipx && python3 -m pipx ensurepath`
  - Windows: `py -m pip install --user pipx && py -m pipx ensurepath` (restart your terminal after).
- **Azure CLI** (`az`), signed in with an account that holds **Microsoft Sentinel Reader** and
  **Log Analytics Reader** on your workspace.
- **Claude Code CLI** installed and signed in.
- **Microsoft 365 E5** or the eDiscovery Premium add-on on the tenant, and the signed-in user must
  hold the **eDiscovery Manager** or **eDiscovery Administrator** Purview role (only needed for
  `purview-dl`).
- Read access to this repository.
- From your SecOps lead: the four routing UUIDs for `config.toml` (tenant, subscription, workspace,
  resource group). They are not secrets, but you need them.

---

## 1 · Clone the repo (1 min)

```bash
# Convention: keep this repo (and any sibling tooling) under ~/repos/.
# install-skills.sh resolves the clone path dynamically, so the symlinks work
# wherever you clone -- but every example in this repo assumes ~/repos/SOC_Investigation_Toolkit.
mkdir -p ~/repos && cd ~/repos
git clone https://github.com/Alfonce27/SOC_Investigation_Toolkit.git
cd ~/repos/SOC_Investigation_Toolkit
```

Verify your clone is on `main`:

```bash
git branch --show-current   # should be `main` (or `feat/...` if you're tracking a feature branch).
```

---

## 2 · Run setup (5 min)

```bash
bash scripts/setup.sh          # macOS / Linux
# .\scripts\setup.ps1          # Windows PowerShell
# scripts\setup.cmd            # Windows cmd.exe
```

Five idempotent phases: prereqs → pipx-install every `tools/inv-*` package → `config.toml` wizard
(paste the four UUIDs from your lead; pre-filled from `az account show` if you're already logged in)
→ register the hunt MCP in `~/.claude.json` → doctor. Every ✗ prints its own fix command.

Then the two things the script deliberately leaves to you:

```bash
az login --tenant <YOUR-TENANT-UUID>   # interactive, MFA-respecting
bash scripts/inv-doctor.sh --fix       # applies the safe auto-fixes; re-check until all ✓
```

`--check` previews setup without touching anything; `--yes` accepts `az`-sourced defaults silently.

---

## 3 · Install the vendored Claude skills (2 min)

```bash
bash scripts/install-skills.sh
```

What it does:

1. Symlinks every `skills/<name>/` into `~/.claude/skills/<name>`. If a real directory already exists
   there (e.g. from an older tarball install), it is moved aside with a timestamp suffix and the
   symlink takes its place. Reversible.
2. Rewrites the `__REPO_ROOT__` placeholder in each `SKILL.md` to your actual clone path. Idempotent —
   re-runs are a no-op.

Verify:

```bash
ls -la ~/.claude/skills/triage-archive ~/.claude/skills/sentinel-hunt   # both symlinks into the repo
```

The `sentinel-hunt` skill drafts KQL against **your** Sentinel workspace and ships an editable inventory
of which tables hold which log source (`skills/sentinel-hunt/references/00-overview.md` and
`01-log-sources/_STATUS.json`). Your SecOps lead maintains that inventory; if a source is marked
`planned` or `not-yet-connected`, the skill refuses to draft against it.

Engineering planning skills (`trinity-plan` / `trinity-execute`) are **not** part of the analyst
install path. They live under `engineering/skills/` and engineers install them via
`bash engineering/scripts/install-engineering-skills.sh`.

Restart Claude Code (Cmd+Q / Ctrl+Q + reopen) so it sees the new skills and the MCP entry.

---

## 4 · Confirm `purview-dl` is installed (1 min)

`setup.sh` already pipx-installed it. Check:

```bash
purview-dl --help    # should list probe / tags / download subcommands.
```

If it is missing: `pipx install ./tools/inv-purview-downloader`. If `pipx install` fails with
"already installed", use `pipx reinstall purview-dl`.

---

## 5 · Authenticate against Purview (3 min, first time only)

Device-code flow — browser-based, MFA-respecting, no Entra app registration required:

```bash
purview-dl probe
```

The tool prints a `https://microsoft.com/devicelogin` URL and a code. Open the URL in any browser
signed in with your work account, enter the code, approve the consent prompt, and switch back to your
terminal. `probe` then lists every eDiscovery case visible to your user.

Refresh tokens are cached at `~/.cache/purview-dl/msal_cache.bin` (`0600`) on macOS/Linux and
`%LOCALAPPDATA%\purview-dl\msal_cache.bin` on Windows. Subsequent runs reuse the cache silently for
~90 days of inactivity. Delete the file to force re-auth.

If `probe` returns 0 cases: your user lacks the eDiscovery Manager / Administrator role. Ask Identity
to add it.

---

## 6 · Run the worked example (5 min)

The repo ships a synthetic test fixture (`tools/inv-archive-triage/tests/fixtures/worked_example_sanitized.zip`)
designed to exercise every triage code path with shape-preserving fakes. Use it as a smoke test for
the whole pipeline.

```bash
cd ~/repos/SOC_Investigation_Toolkit   # adjust if you cloned elsewhere.

python3 tools/inv-archive-triage/scripts/run_all.py \
    --input tools/inv-archive-triage/tests/fixtures/worked_example_sanitized.zip
```

Wall-clock: ~30 s. Outside a case workspace, outputs land under
`tools/inv-archive-triage/reports/run_YYYYMMDD-HHMMSS/` with a stable `reports/latest/` pointer. Open:

```bash
open tools/inv-archive-triage/reports/latest/00_SECURITY_TRIAGE_REPORT.md
```

(Linux: `xdg-open`. Windows: `explorer.exe`.)

You should see a TL;DR, critical/high findings, and `00_NEXT_HUNTS.md` guidance. Cleartext secrets
never appear in any output file.

---

## 7 · Drive the same workflow from Claude Code (5 min)

```bash
cd ~/repos/SOC_Investigation_Toolkit
claude    # open a Claude Code session.
```

In the session, say:

> Triage this zip: tools/inv-archive-triage/tests/fixtures/worked_example_sanitized.zip

The `/triage-archive` skill auto-loads (matched on the natural-language phrase), shells out to the
pipeline, surfaces the TL;DR, and offers to walk you through the next hunts.

To draft — and, with the MCP registered, run — the KQL for one of the next-hunt items, in the same
session:

> Use sentinel-hunt to write me a KQL query that <copy a next-hunt bullet here>.

The MCP executes read-only; results land at `~/.local/state/sentinel-hunt/results/`.

---

## 8 · Real workflow (when an investigation kicks off)

Once the worked example passes, the real loop is:

1. **Bootstrap the case workspace** (once per investigation, off-repo):

   ```bash
   sir-case init <case-id> --display-name "<short name>"
   cd ~/SIR/investigations/<case-id>/
   ```

   From inside the workspace every tool defaults its `--output` automatically.

2. **Download from Purview.** Build a `names.txt` (one filename per line) for the files you need:

   ```bash
   purview-dl download \
       --case <case-name> \
       --reviewset <reviewset-name> \
       --names-file names.txt \
       --remember            # stashes the Purview ids in case.toml for next time
   ```

3. **Acquire other evidence as needed** — `slack-ingest`, `falcon-ingest`, `ado-ingest`,
   `inv-entra-triage`, `inv-xdr-hunt`. Each runs a read-only scope preflight first; if it exits with
   code `8`, the token you were given carries a write scope — send it back.

4. **Triage.** In Claude Code:

   > /triage-archive ./downloads/<archive>.zip

   (Or invoke the Python pipelines directly via `run_all.py`.) Outputs land at
   `<case>/reports/<tool>/run_<ts>/`.

5. **Hunt.** Walk `00_NEXT_HUNTS.md` with `sentinel-hunt`; `sir-case hunts merge / bundle / approve /
   overnight` runs the approved set unattended. `/hypothesis-generator` seeds new hunts from outside
   intel; `/trinity-hunt` and `/hunt-loop` extend from case findings (these send case context to
   third-party LLM APIs — only if your team has approved that).

6. **Hand off.** `inv-timeline-render` for the audience-facing timeline; drop a `00_HANDOFF.md` in the
   case workspace using the template in the top-level README.

---

## 9 · Turn on the hygiene gate (recommended)

The repo ships an audit gate that fails commits introducing case-data references or evidence-shaped
files (`*.csv`, `*.zip`, …) outside `tests/fixtures/`. Two install paths — pick one:

```bash
# Path A (lightweight): git tracks the hook directly.
bash scripts/install-hooks.sh

# Path B (pre-commit framework): if you already use pre-commit.
pip install pre-commit && pre-commit install
```

Run on demand against the whole tree:

```bash
bash engineering/scripts/audit-no-case-data.sh --all
```

Without the hook installed the gate doesn't run locally — CI still enforces it on every PR.

---

## 10 · Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `purview-dl probe` returns 0 cases | Your user lacks the eDiscovery Manager / Administrator Purview role. Ask Identity to grant it. |
| `AADSTS65001: consent required` | A tenant admin needs to consent `eDiscovery.Read.All` for the Microsoft Graph PowerShell public client (`14d82eec-204b-4c2f-b7e8-296a70dab67e`, a Microsoft-published constant). |
| `Multiple cases named 'X'` | Switch to `--case-id <guid>` (run `probe` to discover the GUID). |
| Purview export stays `pending` forever | Big review sets can take 30+ min. Bump `--timeout-seconds 5400`. |
| `pipx install` fails with "already installed" | `pipx reinstall purview-dl` (or `pipx uninstall purview-dl` then re-install). |
| `inv-doctor` shows ✗ on the workspace probe | Confirm `Microsoft Sentinel Reader` + `Log Analytics Reader` on the workspace, and that `config.toml`'s four UUIDs match what your lead gave you. |
| Claude doesn't discover the `triage-archive` skill | Re-run `bash scripts/install-skills.sh`. Confirm `readlink ~/.claude/skills/triage-archive` resolves to your repo clone. Restart Claude Code. |
| Claude doesn't discover `trinity-plan` / `trinity-execute` | Engineering-only skills: `bash engineering/scripts/install-engineering-skills.sh`. |
| `triage-archive` invokes the pipeline but the path doesn't exist | The `__REPO_ROOT__` placeholder didn't get rewritten. Re-run `bash scripts/install-skills.sh`. |
| An ingest tool exits with code `8` | Its token carries a write / manage / admin scope. The tool refuses by design — request a read-only token per the tool's install runbook. |
| pytest fails in `tools/inv-archive-triage/` with `test_no_vendor_drift` | A vendored module under `scripts/lib/` was edited but the SHA pin in `_VENDORED.json` wasn't updated. Re-pin via `dev/check_vendor_drift.py` (it prints the new SHA). |
| Audit gate fails on a legitimate file | Add a glob to `reference/repo-hygiene-allowlist.txt` with a `# reason:` comment. |

---

## What NOT to do

- **Never commit case data.** No `*.csv`, `*.xlsx`, `*.zip`, no `reports/` outputs, no per-case
  `00_HANDOFF.md`. The audit gate is defense in depth — the primary control is "don't add the file in
  the first place". Per-case work lives in `~/SIR/investigations/<case>/`, off-repo.
- **Never commit your MSAL token cache** or `~/.config/sentinel-hunt/` config. Both are in the
  repo-wide `.gitignore`, but be defensive if you ever `cp -r ~/.cache/` for any reason.
- **Never put tokens in `config.toml`.** It holds routing UUIDs only; secrets come from your vault or
  environment variables (see `ENTERPRISE-SETUP.md`).
- **Never push to `main` directly.** Open a PR; CI runs the cross-OS tests and the hygiene gate.
- **Don't modify vendored library files** (`tools/inv-archive-triage/scripts/lib/<vendored>.py`)
  without also updating the SHA pin in `_VENDORED.json`. `test_no_vendor_drift` enforces parity with
  the upstream copy in `tools/inv-purview-edisco-triage/scripts/lib/`.
- **Don't weaken a read-only preflight** to make a token "work". It is a safety property, not a
  nuisance.
