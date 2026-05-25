# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a single-page HTML tool built for **CRC Energy** — a client report generator for the **Fondul pentru Modernizare (FM) 2026 — Stocare Stand-Alone** financing program in Romania. It helps sales reps quickly analyze a company's eligibility and simulate grant calculations by entering a CUI (Romanian company identifier).

The tool is designed to run **embedded inside a Claude AI conversation** (the `sendPrompt()` call passes a summary back into chat), but also works as a standalone HTML file.

## Architecture

Everything lives in a single file: `cui_report_generator.html`.

**No build system, no dependencies to install.** The only external dependency is jsPDF loaded from a CDN at runtime (`cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js`).

### Data Flow

1. User enters a CUI → `fetchCUI()` calls `https://api.openapi.ro/api/companies/{cui}` with header `x-api-key: test`
2. If the API call fails (network error, timeout, non-OK response), `buildMockData()` returns demo data marked with `_demo: true`
3. `renderFirm()` populates the company card and eligibility checks, then calls `recalc()`
4. `recalc()` reacts to all parameter inputs (MW, MWh, EUR/MWh, eligible costs, non-eligible costs, ATR status) and updates three result panels in real time

### Key Business Logic

- **Eligibility**: requires CAEN code starting with `35` (energy sector), active company, positive share capital, not insolvent
- **Grant cap**: `min(eurMWh × MWh, eligibleCosts, 15,000,000 EUR)`
- **Technical requirement**: MWh/MW ratio must be ≥ 2 (minimum 2-hour storage duration)
- **Scoring simulation**: score = `(1 − (eurMWh − 35000) / (69000 − 35000)) × 100` — lower EUR/MWh request scores higher
- **Pre-financing**: 20% of grant
- **Max EUR/MWh**: 69,000; minimum project size: 1 MW

### UI Sections (in order)

1. CUI search input + loading state
2. Company card (name, meta, status badges)
3. Company data + eligibility checklist (grid 2-col)
4. Project parameters inputs (MW, MWh, EUR/MWh, eligible costs, non-eligible costs, ATR status)
5. Dimensioning / grant calculation / scoring simulation (grid 3-col)
6. Document checklist (opis) with progress bar
7. Action buttons: PDF export + send to chat

### PDF Generation (`generatePDF`)

Uses jsPDF directly (no HTML-to-canvas). Draws header, colored section bars, two-column rows, progress bars, and footer manually with coordinates. Saves as `CRC_Raport_{CUI}_{date}.pdf`.

### Claude Chat Integration (`sendToChat`)

Calls `sendPrompt()` (injected by the Claude environment) with a pre-filled Romanian prompt asking Claude to generate a client email. This function only works when the tool is embedded in a Claude conversation artifact.

## Planned Integration: demoanaf.ro

The current company data source (`api.openapi.ro`) should be replaced or supplemented with **demoanaf.ro** API. The MCP tools available for this are prefixed `mcp__claude_ai_demoanaf_ro__*`. Relevant tools:

- `search_company` / `get_company` — look up company by CUI or name
- `get_company_financials` — balance sheet data (replaces the stub capital social field)
- `list_supplier_issues` — insolvency / fiscal issues check
- `list_company_tenders` / `check_company_contracts` — procurement history (useful for scoring context)

When integrating, keep the `buildMockData()` fallback for offline/demo use, and mark live demoanaf data with `_demo: false`.

## Supporting Documents

- `CRC_Energy_Brief_Sales_Stocare.docx` — sales brief for the FM 2026 program
- `CRC_Energy_Stocare_OnePager.docx` — one-pager for client-facing use

These are reference documents, not part of the app build.

## Styling Conventions

The CSS uses CSS custom properties (`var(--color-background-primary)`, `var(--color-border-secondary)`, etc.) sourced from the Claude conversation host environment. Brand colors used directly:
- Primary blue: `#005B7F`
- Accent cyan: `#009BD1`
- Success green: `#1E8C4E`
- Warning amber: `#F0A500`
- Error red: `#C0392B`

Icons come from Tabler Icons CDN (`ti ti-*` classes), also injected by the host environment.
