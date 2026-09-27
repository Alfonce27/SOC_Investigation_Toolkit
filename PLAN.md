# PLAN.md — Sanitize and Publish the SOC Investigation Toolkit to GitHub

**Status:** Draft v1 · **Owner:** SecOps Engineering · **Target:** public (or org-internal) GitHub repository

This plan turns the internal `InternalSecurityTooling` monorepo (18 tools, 7 skills, ~2,250 files) into a
GitHub-publishable repository with **zero** references to the originating company, its people, its tenant,
or any live case. It is written so that a second engineer can execute it without having seen the original
archive.

> **Golden rule:** the existing git history is *never* pushed. The public repo starts from a fresh
> `git init` on a sanitized working tree. Nothing in this plan changes that.

---

## 1. Objective and non-goals

| | |
|---|---|
| **Objective** | Publish the toolkit (code, skills, docs, CI, synthetic fixtures) under a neutral name with every company-specific identifier removed or replaced by a documented placeholder. |
| **Must preserve** | Tool behaviour, CLI names, config schemas, tests that run offline, the read-only safety preflights, and the repo-hygiene gate (it becomes the public repo's own guard rail). |
| **Non-goals** | Re-architecting tools; renaming console entry points (`sir-case`, `purview-dl`, `falcon-ingest`, …); sanitizing *history* — history is discarded, not rewritten. |

---

## 2. What is in the source archive (inventory)

| Area | Contents | Notes |
|---|---|---|
| `tools/` (18) | Python (Layout A script collections and Layout B pipx packages) and Go single-binary tools for acquisition, triage, hunting and evidence collection | Core value of the repo |
| `skills/` (5) + `engineering/skills/` (2) | Claude Code skills: `triage-archive`, `sentinel-hunt`, `hunt-loop`, `hypothesis-generator`, `trinity-hunt`; engineering-only `trinity-plan`, `trinity-execute` | Keep |
| `.claude/skills/writing-secops-docs/` | Internal doc-writing skill with a personal WORKLOG/LEARNING journal and a vendor-specific lessons file | Partially keep |
| `engineering/` | `inv-bootstrap` (doctor/setup engine), `go/invkit` shared Go library, parity harness with golden tapes, scripts, standards docs, **~150 sprint planning docs** (`docs/sprints/` + `drafts/`) | Sprint docs are the largest sensitivity surface |
| `reference/` | JSON schemas, MITRE ATT&CK snapshot, 1Password vault template, hygiene allowlist and banned-strings file | Keep, rewrite two files |
| `scripts/`, `docs/`, root `*.md` | Setup/quickstart/doctor shims, onboarding docs, `workflows.html` + manifest, `README`, `QUICKSTART`, `ONBOARDING`, `PROGRESS`, `WORKLOG` | Rewrite / drop |
| `tests/fixtures/` | Synthetic fixtures incl. one 8.7 MB `worked_example_sanitized.zip` | Verify before keeping |

---

## 3. Sensitivity findings (what the scan surfaced)

Values are deliberately **not** reproduced here; this file itself must be safe to commit. Counts are
approximate and come from a full-tree grep of the extracted archive.

### 3.1 Company identity — REMOVE
- Company name and legacy brand name: **~956 files**, both prose and code (test fixtures, regexes, JSON patterns, Go tests, hunt scenarios).
- Azure DevOps organization / project clone URL: 13 files (`README`, `QUICKSTART`, `ONBOARDING`, `docs/*`, `scripts/*`).
- Corporate M365 tenant domain (`<company>.onmicrosoft.com`) and corporate mail domain: ~140 lines, mostly in sprint drafts and a few test fixtures.
- Customer-support portal hostname and docs hostname (7 + 1 mentions).
- Tenant-specific Defender for Cloud Apps portal host (regex in banned-strings; parameterized in scripts already).
- MSSP-branded alert prefixes and parser names inside raw Sentinel API dumps (see 3.3).
- Log Analytics workspace *name* (3 mentions) and profile names of shape `<company>-sentinel-{prod,staging,dev}` in `config.toml.example`.
- A vendor PAM product name that doubles as the company name: used as a log-source reference filename, a hunt-scenario family (secret checkout / failed auth / first-time access / off-hours access), table names in the KQL cheat sheet, and the glossary.

### 3.2 People — REMOVE
- Primary engineer/analyst full name + macOS home-directory path (`/Users/<First.Last>`): **~140 occurrences** (sprint docs, `distributed-hunt/setup.sh`, `WORKLOG`, `config.toml.example` "ask X for the prod values", `docs/workflows*`).
- Seven additional employee UPNs (cloud-admin and standard accounts) and one victim/data-subject UPN: ~250 lines, concentrated in `tools/inv-activity-log-triage/docs/sprints/**`, `engineering/docs/sprints/**`, and two test files.
- One `C:\Users\<initials>` path in a Windows validation runbook.

### 3.3 Live case data — REMOVE ENTIRELY
- One real case-ID slug (shape `inv\d{6}`): **434 occurrences**. A second dated case slug and an "oath bundle" path appear in `engineering/scripts/distributed-hunt/setup.sh`.
- `engineering/docs/sprints/SPRINT-MCP-005-FINDINGS/`: raw `SecurityAlert` and `Heartbeat` query results, workspace touch history, SDK probe output. **These are production telemetry exports.**
- Sprint `INTENT` / `*-DRAFT` / `*-CRITIQUE` / `MERGE-NOTES` files embed: attacker and recon source IPs, SPN object GUIDs, sign-in TrackingIds, host IDs, a tenant GUID, DC / ADFS / AD CS FQDNs, timestamps of a confirmed account takeover, hunt narratives naming the subject.
- Two `s11_next_hunts.py` / `s12_exec_brief.py` scripts in `inv-activity-log-triage` still carry a hard-coded TrackingId, IPs and a "why_now" narrative from the original case in their default/example text.
- `tools/inv-salesforce-triage/tests/fixtures/cloudappevents_guest_cluster.csv` and its tests use four real-looking public IPs from the originating case.
- `tools/inv-archive-triage/reference/org_patterns.json` and `tests/fixtures/sprint011_cross_archive_match/manual.csv`: company-specific ownership/org regexes and sample rows.

### 3.4 Infrastructure identifiers — REPLACE
- ~12 non-RFC-5737 public IPv4 addresses and ~10 RFC-1918 addresses outside of fixtures already using TEST-NET ranges.
- Internal FQDNs (`corp.<company>.com`, `devint.<legacy>.com`).
- Tenant GUID and several object GUIDs. Microsoft *public* well-known GUIDs (Graph PowerShell client, Azure CLI client, Sentinel Reader role id, an ASR rule id, the Windows supportedOS GUID) are **safe** and are already allow-listed by the hygiene gate.

### 3.5 Vendor / security-stack disclosure — JUDGMENT CALL
- Integrations that *are* the tool's purpose (Microsoft Sentinel/Defender/Purview/Entra, CrowdStrike Falcon, Slack, Azure DevOps, Salesforce Experience Cloud, AWS CloudTrail) must stay.
- Peripheral stack mentions that only describe the originating environment — an MSSP, a PAM vendor, an RMM vendor, a firewall product, an observability vendor, a cloud-security vendor, a secure-access broker — reveal the org's stack when combined. Recommendation: keep the *log-source reference docs* (they are useful, generic KQL), but strip the "our environment has X" framing and the roadmap line about the deferred RMM+firewall correlation tool.

### 3.6 Secrets — NONE FOUND (verify anyway)
- All token-shaped strings (`xoxb-…`, `xoxp-…`, `sk-ant-…`, `AKIA…`) are documented fakes used by "no-secret-echo" tests. One AWS-key-shaped fixture string is not the canonical AWS example key; swap it for `AKIAIOSFODNN7EXAMPLE` so scanners stay quiet.
- `requirements.lock`, `go.sum` — fine.

### 3.7 Personal work journals — DROP
- `WORKLOG.md` (230 KB, append-only session log with approvals, PR numbers, WIP notes), `PROGRESS.md`, `.claude/skills/writing-secops-docs/{WORKLOG,LEARNING}.md`, `engineering/docs/runbooks/ado-access-followups.md`.

---

## 4. Disposition matrix

| Disposition | Paths |
|---|---|
| **DROP** (do not copy into the public tree) | `WORKLOG.md`, `PROGRESS.md`, `engineering/docs/sprints/**` (all sprint packs and drafts), `tools/*/docs/sprints/**`, `engineering/docs/sprints/SPRINT-MCP-005-FINDINGS/**`, `engineering/scripts/distributed-hunt/**`, `engineering/docs/runbooks/**`, `.claude/skills/writing-secops-docs/{WORKLOG,LEARNING,DISTRIBUTION}.md`, `.claude/skills/writing-secops-docs/references/lessons-from-<vendor>.md`, `engineering/scripts/tests/*.SUPERSEDED-*`, `tools/inv-archive-triage/tests/fixtures/sprint011_cross_archive_match/manual.csv`, `docs/EARLY-TEST.md` |
| **REWRITE** (keep file, replace content) | `README.md` (replaced by the new one shipped with this plan), `QUICKSTART.md`, `ONBOARDING.md`, `docs/ANALYST-ONBOARDING.md`, `docs/WINDOWS-QUICKSTART.md`, `docs/workflows_manifest.json` + regenerate `docs/workflows.html`, `skills/README.md`, `engineering/AUDIENCE.md`, `engineering/ONBOARDING.md`, `engineering/docs/GO-STANDARDS.md`, `engineering/docs/SUPPLY-CHAIN-PINS.md`, every `tools/*/README.md` and `QUICKSTART.md`, every `config.toml.example` / `config.example.toml`, `reference/onepassword-bootstrap-template.md`, `reference/repo-hygiene-banned-strings.txt`, `reference/repo-hygiene-allowlist.txt`, `tools/inv-archive-triage/reference/org_patterns.json` → `org_patterns.example.json`, `skills/sentinel-hunt/references/**` (overview, glossary, KQL cheat sheet, watchlists, all 86 hunt scenarios), `skills/sentinel-hunt/SKILL.md`, `skills/sentinel-hunt/QUICKSTART.md`, `tools/inv-activity-log-triage/scripts/s11_next_hunts.py`, `tools/inv-activity-log-triage/scripts/s12_exec_brief.py`, `tools/inv-salesforce-triage/**` (fixture IPs → TEST-NET, "internal use only" banner, case reference), `tools/inv-hunt-mcp/config.toml.example`, `.github/workflows/ci.yml` (only comments), test files listed in §6 |
| **RENAME** | `skills/sentinel-hunt/references/01-log-sources/<vendor>-<product>.md` (the PAM log source) → `pam-vault.md`; hunt scenarios 01–05, 07–08, 12–13, 30: retitle from the vendor product name to "PAM vault" wording and update their `log_sources:` front-matter |
| **VERIFY, then keep or drop** | `tools/inv-archive-triage/tests/fixtures/worked_example_sanitized.zip` (8.7 MB) — list every member and grep the extracted content with the sweep in §7; drop it and mark the dependent tests `skip` if anything real is inside |
| **KEEP AS-IS** | All Go source and tests under `engineering/go/invkit`, `tools/inv-{ado-enum,falcon-ingest}/go`, `tools/inv-{entra-triage,soc2-evidence,xdr-hunt}`; parity harness and golden tapes (already use `CASE-PARITY` and `example.test`); `reference/*.schema.json`, `reference/mitre_attack_techniques.json`; scripts under `scripts/` and `engineering/scripts/` other than those listed above (after a `<company>` grep) |

---

## 5. Replacement map (canonical placeholders)

Apply case-insensitively. Every replacement must be *recognisably synthetic* so the hygiene gate and future
readers can tell placeholder from data.

| Original class | Replacement |
|---|---|
| Company name / legacy brand | `<COMPANY>` in prose; `example-corp` in identifiers; `Example Corp` in titles |
| Product name that equals company name (PAM vault) | "PAM vault" / `pam-vault` |
| Corporate mail domain | `example.com` |
| M365 tenant domain | `contoso.onmicrosoft.com` |
| Internal FQDNs | `dc01.corp.example.com`, `adfs.corp.example.com`, `adcs.corp.example.com` |
| ADO clone URL | `https://github.com/<your-org>/soc-investigation-toolkit.git` |
| Repo directory name | `soc-investigation-toolkit` |
| Primary engineer name / home path | `<analyst>` / `~` or `/Users/<analyst>` |
| Other employee UPNs | `alice@example.com`, `bob.cloudadmin@contoso.onmicrosoft.com`, … (already the fixture convention) |
| Victim / subject UPN | `subject.user@example.com` |
| Case-ID slug | `inv000000` (already allow-listed as a placeholder shape) or `<case-id>` in prose |
| Public IPs | `203.0.113.x`, `198.51.100.x`, `192.0.2.x` (RFC 5737) |
| RFC-1918 IPs | `10.0.0.x` only where the test needs a private shape; otherwise TEST-NET |
| Tenant / object GUIDs | `00000000-0000-0000-0000-00000000000N` |
| Workspace name / profiles | `sentinel-prod`, `sentinel-staging`, `sentinel-dev` |
| MSSP alert prefixes / parser names | delete the files that contain them (raw dumps) |
| "ask <person> for the prod UUIDs" | "ask your SecOps lead for the routing UUIDs" |
| Support-portal hostname | `support.example.com` |
| 1Password vault default | keep `SIR-team` (documented convention, env-overridable) — or change default to `soc-team` if preferred |

---

## 6. Execution phases

### Phase 0 — Set up an isolated working area (30 min)
- [ ] Extract the archive into a scratch directory **outside** any existing git checkout.
- [ ] Do **not** run `git init` yet. Copy only the files the disposition matrix keeps into `public/` using an explicit allow-list script (`rsync --files-from`), not an exclude list — exclusion lists miss new files.
- [ ] Record the SHA-256 of the source archive in the engineering ticket for provenance.

### Phase 1 — Bulk drop (15 min)
- [ ] Delete every path in the **DROP** row of §4.
- [ ] Delete `tests/fixtures/worked_example_sanitized.zip` until Phase 4 verification clears it.
- [ ] Confirm `find public -name "SPRINT-*"` returns nothing.

### Phase 2 — Mechanical rewrite (2–3 h)
- [ ] Write `sanitize.map` with one `pattern<TAB>replacement` per §5 row; apply with a single Python pass that reports every file touched and every hit count (keep the report for review).
- [ ] Rename the PAM log-source reference file and update the `_STATUS.json` index and every hunt-scenario front-matter `log_sources:` entry that points at it.
- [ ] Replace `org_patterns.json` with `org_patterns.example.json` containing `example-corp` patterns; update the loader's default path and its tests.
- [ ] Swap the AWS-key-shaped fixture for the canonical AWS example key.
- [ ] Remove the case narrative defaults from `s11_next_hunts.py` / `s12_exec_brief.py` (leave neutral placeholder text, keep the data shape).
- [ ] Replace fixture IPs in `inv-salesforce-triage` with RFC 5737 addresses; regenerate `cloudappevents_guest_cluster.csv` via `tests/fixtures/_make_fixture.py`.
- [ ] Rewrite `reference/repo-hygiene-banned-strings.txt`: keep the structure and the public-GUID allow-list, replace the company-specific regexes with commented **examples** the adopter fills in (`re:[A-Za-z0-9._%+-]+@yourcompany\.com`).

### Phase 3 — Prose rewrite (3–4 h)
- [ ] Drop the shipped `README.md` (sanitized) into the root; port `QUICKSTART.md`, `ONBOARDING.md`, `docs/ANALYST-ONBOARDING.md`, `docs/WINDOWS-QUICKSTART.md` to the same neutral voice (clone URL, "your Sentinel workspace", "your SecOps lead").
- [ ] Rewrite `skills/sentinel-hunt/references/00-overview.md` so table→log-source mappings are presented as **an example inventory the adopter edits**, not "our tables".
- [ ] Edit each of the 86 hunt scenarios: front-matter `owner:`/`author:` fields → `soc-team`; "in our tenant" → "in your tenant"; remove any `approved_ips` / `approved_apps` example rows that were real.
- [ ] Rebuild `docs/workflows_manifest.json` and run `engineering/scripts/render-workflows.py` to regenerate `docs/workflows.html`.
- [ ] Add the shipped `docs/ENTERPRISE-SETUP.md`.
- [ ] Add `LICENSE` (choose: Apache-2.0 recommended for tooling; note the vendored `markdown2.py` and the third-party `get_SPN.ps1` carry their own licenses — keep their headers and add a `THIRD_PARTY_NOTICES.md`).
- [ ] Add `SECURITY.md` (private disclosure contact), `CONTRIBUTING.md` (hygiene gate is mandatory; no fixtures from real cases), `CODE_OF_CONDUCT.md` (optional), `.github/CODEOWNERS`, `.github/dependabot.yml`.

### Phase 4 — Verification gates (1–2 h; all must pass)
- [ ] **Gate A — company sweep:** `grep -rIil -E "<company>|<legacy-brand>|<tenant-domain>|<ado-org>" public/` → 0 files.
- [ ] **Gate B — people sweep:** grep for every UPN local-part and surname collected in Phase 0 → 0 hits.
- [ ] **Gate C — case sweep:** `grep -rIoE "\binv[0-9]{6}\b" public/ | grep -v inv000000` → 0; grep the second case slug → 0.
- [ ] **Gate D — IP sweep:** list all IPv4 literals; every non-loopback hit must be RFC 5737 / RFC 1918 fixture or a documented public DNS resolver.
- [ ] **Gate E — GUID sweep:** every GUID must be all-zero/synthetic or on the public-GUID allow-list.
- [ ] **Gate F — secrets:** run `gitleaks detect --no-git -s public/` and `trufflehog filesystem public/`; triage every hit to a documented fake.
- [ ] **Gate G — the repo's own gate:** `bash engineering/scripts/audit-no-case-data.sh --all` exits 0 with the rewritten banned-strings file.
- [ ] **Gate H — fixture zip:** `unzip -l` + extract to tmp + rerun Gates A–F on the extracted tree. Keep only if clean.
- [ ] **Gate I — tests still run:** `python -m pytest` per tool (offline subset), `go test ./...` with `GOWORK=off`, and the CI workflow on a private fork.
- [ ] **Gate J — human read-through:** two reviewers read `README.md`, `docs/**`, `skills/sentinel-hunt/references/00-overview.md`, and 10 randomly sampled hunt scenarios end-to-end.

### Phase 5 — Fresh history and publication (30 min)
- [ ] `cd public && git init -b main && git add -A && git commit -m "Initial public release (sanitized)"` — **one commit, no imported history**.
- [ ] Create the GitHub repository (`soc-investigation-toolkit`), private first.
- [ ] Enable: branch protection on `main` (PR required, CI required, no force-push), secret scanning + push protection, Dependabot alerts, signed commits (recommended).
- [ ] Push; confirm CI matrix (Ubuntu/macOS/Windows) is green.
- [ ] Run GitHub's secret scanning and a second `gitleaks` on the pushed repo.
- [ ] Flip visibility to public only after the Gate J reviewers sign off in the PR that adds `LICENSE`.

### Phase 6 — Post-publication hygiene (ongoing)
- [ ] Keep the internal repo as the upstream; sync to public via a scripted "export" branch that re-runs Phases 1–4 (never cherry-pick internal commits).
- [ ] Extend `repo-hygiene-banned-strings.txt` in the *public* repo with a `re:` line for `yourcompany`-shaped tokens so adopters do not leak their own tenant.
- [ ] Add a CI job that runs Gates A–F on every PR to the public repo.

---

## 7. Sweep commands (generic; fill in the values from Phase 0 locally, never commit them)

```bash
# Company / brand / tenant / ADO org (fill $TERMS from an untracked local file)
grep -rIil -E -f .local/sensitive-terms.txt public/ | tee /tmp/gateA.txt ; test ! -s /tmp/gateA.txt

# Case ids
grep -rIoE '\binv[0-9]{6}\b' public/ | grep -v inv000000 ; test $? -eq 1

# IPv4 literals not in RFC 5737 / loopback
grep -rIohE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' public/ \
  | grep -vE '^(192\.0\.2|198\.51\.100|203\.0\.113|127\.0\.0|0\.0\.0\.0)' | sort -u

# GUIDs not on the allow-list
grep -rIohE '[0-9a-fA-F]{8}-([0-9a-fA-F]{4}-){3}[0-9a-fA-F]{12}' public/ | sort -u \
  | grep -vE '^(0{8}-0{4}-0{4}-0{4}-0{11}[0-9]|a0{7}-|1{8}-|2{8}-|3{8}-|abcd1234-)' 

# Emails
grep -rIohE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' public/ | sort | uniq -c | sort -rn

# Home paths
grep -rIn -E '/Users/[A-Za-z.]+|C:\\Users\\[A-Za-z.]+' public/ | grep -vE '<analyst>|USERNAME|Public'

# Secrets
gitleaks detect --no-git -s public/ --report-path /tmp/gitleaks.json
```

---

## 8. Residual-risk register

| Risk | Likelihood | Mitigation |
|---|---|---|
| A hunt scenario's example KQL still encodes a real approved-IP or app-ID watchlist row | Medium | Gate D/E + Gate J sampling; treat every `approved_*` watchlist example as synthetic-by-construction |
| Fixture zip contains a real artifact despite its name | Medium | Gate H; drop if any doubt |
| Vendor-stack inference from the log-source reference set | Low | Accepted — the docs are generic KQL; remove "our environment" framing |
| Third-party script license mismatch (`get_SPN.ps1`, `markdown2.py`) | Low | `THIRD_PARTY_NOTICES.md`; keep original headers |
| Internal repo commit accidentally pushed to public remote | Low | Public repo has no shared history; branch protection; export-branch workflow only |
| Adopter commits their own tenant data | Medium | Ship the hygiene gate enabled by default; `CONTRIBUTING.md` mandates it |

---

## 9. Deliverables that accompany this plan

| File | Purpose |
|---|---|
| `PLAN.md` | This document |
| `README.md` | Sanitized, GitHub-ready front door for the repository |
| `docs/ENTERPRISE-SETUP.md` | Step-by-step configuration and engineering guide for deploying the toolkit in an enterprise environment |

**Definition of done:** all Phase 4 gates green, single-commit history, CI green on three OSes, two sign-offs
on Gate J, repository visibility flipped.
