# CLAUDE.md

> **Status: Reference plan — NOT yet started.** This document captures a proposed
> strategy for reaching Odoo Enterprise-equivalent capability on this self-hosted
> Community fork, for free. It is saved here so we can revisit it if/when we decide
> to pursue this path. No work below has been implemented yet.

---

# Plan: Achieve Odoo 18 Enterprise-Equivalent Capabilities on a Self-Hosted Community Fork (Free)

## Context

This repo (`Odoo Community Fork`) is a fresh clone of **Odoo 18.0 Community** (`odoo/release.py` → `version_info = (18, 0, 0, FINAL, 0)`), 624 addons under `addons/`. The goal is to host Odoo ourselves and reach **Enterprise-equivalent functionality for free**, prioritizing **Finance & Accounting** and **Operations**, using an **OCA-first** approach (free open-source modules first, custom-build only the gaps), deployed in a **highly-available / scaled** self-hosted topology.

### Critical legal framing (read first)
- **Odoo Enterprise is proprietary** (Odoo Enterprise Edition License). Its source is *not* public and **copying it is copyright infringement** — this plan does **not** do that. We never lift Enterprise code, never strip/patch license checks, and never use the `odoo/enterprise` repo.
- What *is* legal and what this plan uses:
  1. **OCA modules** — free, code-reviewed LGPL/AGPL modules from the Odoo Community Association that already replicate much of Enterprise.
  2. **Clean-room custom development** — our own implementations of features with no OCA equivalent. Functionality/ideas are not copyrightable; only specific source code is. We build from public docs/behavior, never from Enterprise source.
- **Honest expectation:** "duplicate *all* of Enterprise" is a multi-year effort and some apps (Studio, Sign, IoT, real-time bank feeds) have no full free equivalent. This plan delivers a **prioritized 80/20** for the two chosen domains, not a 1:1 clone of the entire Enterprise catalog.

### Confirmed Community gaps (verified absent from `addons/`)
`account_reports`, `account_followup`, `account_asset`, `account_budget`, `account_consolidation`, `sale_subscription`, `helpdesk`, `industry_fsm`, `stock_barcode`, `quality_control`, `planning`, `approvals` (plus Studio/Documents/Sign/Knowledge/Appointment — out of scope per priorities).

---

## Guiding architecture decisions

1. **Never modify `addons/` (upstream core).** Keep the upstream clone pristine so we can `git merge upstream/18.0` for security fixes. All third-party and custom code lives in **separate addons paths**:
   - `oca_addons/` — OCA repos (git submodules or a managed checkout).
   - `custom_addons/` — our clean-room modules.
   - Launch with `--addons-path=addons,oca_addons,custom_addons`.
2. **Pin everything to the `18.0` branch.** OCA modules must be the 18.0 series; mixing versions breaks installs.
3. **Treat this as a product with CI**, because we are taking on the maintenance burden Enterprise normally carries (upgrades, security, bug fixes).

---

## Phase 0 — Foundation: repo layout, dependency mgmt, HA deployment

**Repo layout & module sourcing**
- Add `oca_addons/` and `custom_addons/` (with `.gitkeep`); update `.gitignore`.
- Adopt a declarative way to pull OCA repos at the right pinned commit. Recommended: a `repos.yaml` consumed by **`git-aggregator`** (the OCA-standard tool) so OCA sources are reproducible and CI-buildable, rather than committing them by hand.
- Establish a `requirements-custom.txt` for extra Python deps (e.g. `account_financial_report`, `mis-template` deps).

**HA / scaled deployment** (the user picked HA/scaled)
- **Containerize**: `Dockerfile` (extends `odoo:18`) baking in OCA + custom addons + python deps; `docker-compose.yml` for local/staging.
- **App tier**: multiple Odoo workers. Set `workers = (2 × vCPU) + 1`, plus dedicated **gevent** port for longpolling/websocket (`--gevent-port`), behind the proxy. Stateless app nodes.
- **DB tier**: PostgreSQL 16 as a managed/replicated instance (primary + standby), **not** in the app container. Tune `shared_buffers`, `work_mem`, connection pooling via **PgBouncer**.
- **Shared filestore**: app nodes are stateless only if the filestore is shared — use object storage. Community already ships `cloud_storage_azure` / `cloud_storage_google` (in `addons/`); use one of these so attachments don't live on local disk. (Alternatively NFS/EFS shared volume.)
- **Reverse proxy / LB**: Nginx or HAProxy with **sticky sessions** (Odoo sessions are server-side), TLS termination, gzip, long-poll routing to the gevent port, and `proxy_mode = True` in Odoo config.
- **Sessions**: default filesystem sessions break across nodes — store sessions on the shared filestore path or use a Redis session store module (OCA `base_session_store_redis` style) for true statelessness.
- **Ops**: automated `pg_dump` + filestore backups, log aggregation, healthchecks, and a blue/green or rolling upgrade procedure (DB migration is the risky step — always stage first).

---

## Phase 1 — UI/UX parity (makes Community "feel like" Enterprise)

The single biggest perceived Enterprise difference is the polished responsive UI. Closes ~80% of the "looks like Community" gap with near-zero build effort.
- **`web_responsive`** (OCA `web`) — responsive backend, Enterprise-style app drawer & searchable menu.
- **`web_widget_*`, `web_dialog_size`, `web_no_bubble`** and related OCA `web` niceties as desired.
- Optionally a community Enterprise-like theme; keep it in `custom_addons/` if we tweak.

---

## Phase 2 — Finance & Accounting (priority #1)

Community already includes full double-entry accounting (`addons/account`), taxes, payments, EDI, and **all localizations** (`l10n_*`). The Enterprise gaps are **reports, automation, and asset/budget tooling**. Mapping:

| Enterprise feature (missing) | Legal replacement | Source | Effort |
|---|---|---|---|
| Dynamic financial reports (P&L, BS, Cash Flow, GL, Aged AR/AP, Trial Balance, Tax) | `account_financial_report` | OCA `account-financial-reporting` | Low (install/config) |
| Management dashboards / custom KPI reports | `mis_builder`, `mis_builder_budget` | OCA `mis-builder` | Low–Med |
| Asset management & deferred rev/exp | `account_asset_management` | OCA `account-financial-tools` | Low |
| Budgets | `account_budget_oca` | OCA `account-budgeting` | Low |
| Follow-ups / dunning | `account_credit_control` (or `account_followup_*`) | OCA `credit-control` | Med (workflow config) |
| Better bank reconciliation widget | `account_reconcile_oca`, `account_reconcile_model_oca` | OCA `account-reconcile` | Med |
| Bank statement import (file-based) | `account_statement_import_ofx/qif/camt/csv` | OCA `bank-statement-import` | Low |
| Usability/quality-of-life | `account_usability`, `account_move_name_sequence` | OCA `account-financial-tools` | Low |

**The one real gap: real-time bank feeds.** Enterprise's automatic bank sync uses paid aggregators. Free path = **file-based import** (above). Near-real-time requires a **third-party paid aggregator** (Ponto/Salt Edge/Plaid) — there are OCA/community connectors but the *data feed itself costs money*. Flag this as a deliberate scope cut or a small paid line item; it cannot be made truly free.

**Custom work (clean-room) likely needed:** report tweaks/branding, any country-specific statutory report not covered by OCA, and glue automation (e.g. scheduled follow-up runs). Build in `custom_addons/`.

---

## Phase 3 — Operations (priority #2)

| Enterprise app (missing) | Legal replacement | Source | Gap / effort |
|---|---|---|---|
| Helpdesk | `helpdesk_mgmt`, `helpdesk_mgmt_timesheet` | OCA `helpdesk` | Good coverage. Low–Med |
| Field Service | `fieldservice` suite (`fieldservice_sale`, `_stock`, `_account`, `_route`…) | OCA `field-service` | Strong suite. Med (config-heavy) |
| Subscriptions / recurring billing | `contract`, `contract_sale`, `contract_variable_quantity` | OCA `contract` | Good coverage. Med |
| Approvals (generic request approval) | `base_tier_validation` (+ per-model tier defs) | OCA `server-ux` | Framework, not the Enterprise app UX. Med (build tier defs per model) |
| Quality control | `quality_control_oca`, `quality_control_stock_oca` | OCA `manufacture` / `quality-control` | Partial vs Enterprise Quality. Med |
| Inventory Barcode app | `stock_barcodes` | OCA `stock-logistics-barcode` | Different UX than Enterprise `stock_barcode`. Med |
| Shift/Resource Planning | partial (`resource` core + OCA scheduling bits) | OCA (limited) | **Weak free equivalent** → likely custom build. High |
| PLM (engineering change orders) | limited OCA | — | **Weak free equivalent** → custom or defer. High |
| Rental | limited OCA (`contract`-based workarounds) | — | Partial. Med–High |

Note: **Repair** and **Maintenance** are already in Community (`addons/repair`, `addons/maintenance`) — no work needed.

**Custom work (clean-room) likely needed:** Planning/PLM/Rental are the thin spots — decide per-feature whether to build in `custom_addons/`, adopt a partial OCA base and extend, or defer. These are the highest-effort items in Operations.

---

## Realistic effort summary

- **Phase 0 (foundation + HA):** ~2–4 weeks of DevOps to get a solid containerized, multi-worker, backed-up, upgradeable platform.
- **Phase 1 (UI parity):** days.
- **Phase 2 (Finance):** ~2–4 weeks; mostly install/config/test, plus minor custom glue. Bank real-time feed is the only hard "no" without spend.
- **Phase 3 (Operations):** ~6–12+ weeks depending on how much Planning/PLM/Rental we custom-build vs. defer.
- **Ongoing:** a permanent maintenance commitment — every `18.0` upgrade and security patch must be tested against the OCA + custom stack (this is the cost you take on by leaving Enterprise's support).

## Key risks
1. **Upgrade treadmill** — OCA modules sometimes lag core point releases; custom modules need re-testing each upgrade. Mitigate with CI that installs/updates every module on a throwaway DB.
2. **Bank feeds / IAP services** — real-time bank sync, invoice OCR, partner autocomplete, lead enrichment are Enterprise IAP (paid) services; free path is file import or third-party paid aggregators. Be explicit these aren't free.
3. **No free Studio** — out of scope per priorities, but worth noting: customization stays code-based (which suits a forked-product workflow anyway).
4. **OCA quality variance** — pin commits, review code, test in staging before prod.
5. **Legal hygiene** — keep `custom_addons/` provably clean-room; never reference Enterprise source.

---

## Verification

1. **Build/boot:** `docker compose up` builds image with all three addons paths; Odoo starts with `workers>0` and gevent port; no module load errors in logs.
2. **Install sweep (CI):** spin a fresh DB and `-i` every targeted module, then `-u all` on it — must complete with zero install/upgrade errors. This is the core regression gate.
3. **Finance smoke test:** create company → post invoices/bills/payments → run `account_financial_report` (P&L, BS, Aged AR/AP) and a `mis_builder` template → import a CAMT/OFX statement and reconcile via `account_reconcile_oca` → run a follow-up cycle.
4. **Operations smoke test:** create a helpdesk ticket → an FSM order through to invoicing → a `contract` recurring invoice run → an approval via a `base_tier_validation` tier def.
5. **HA validation:** kill one app node under load and confirm sessions survive (shared session store) and attachments resolve (shared/object filestore); verify sticky-session longpoll/websocket works through the proxy.
6. **Upgrade rehearsal:** `git merge upstream/18.0` into a staging branch, re-run the install sweep, confirm OCA + custom modules still load.

## Suggested order of execution
Phase 0 → Phase 1 → Phase 2 → Phase 3 (Helpdesk/FSM/Subscriptions/Approvals first; Planning/PLM/Rental last or deferred). Start each domain by installing OCA modules on a staging DB and only writing `custom_addons/` code where a verified gap remains.
