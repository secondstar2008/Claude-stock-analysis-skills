# Stock Analysis Skill

An institutional-grade equity research framework for [Claude.ai](https://claude.ai), designed to produce deep-dive, adversarial analysis on stocks across Vietnamese (HOSE/HNX), US (NYSE/NASDAQ), and global markets.

This skill mirrors the workflow of a buy-side analyst preparing a recommendation memo: data gathering, bull/bear construction, competitive analysis, comps, forward projections, red-flag screening, and a final verdict.

---

## What this skill does

Given a ticker or company name, the skill instructs Claude to:

1. Pull current, real data from primary sources before writing — using FiinData MCP tools for Vietnamese stocks and `web_search` for all other markets.
2. Run a structured 14-section analysis — or only the sections requested.
3. Output a professionally formatted, self-contained **HTML file** with a sidebar table of contents, data grid header, TL;DR panel, styled tables, and a verdict callout with risk/reward levels.

The output is intentionally adversarial: the framework forces the analyst (Claude) to argue both sides, perform a pre-mortem, strip out one-off earnings, and play short-seller before reaching a conclusion.

---

## The 14-section framework

| # | Section | Purpose |
|---|---|---|
| 1 | Company Overview | Business model, customers, suppliers, contracts |
| 2 | Bull vs. Bear | Adversarial deep-dive; pre-mortem; contrarian view |
| 3 | Competitive Advantages | Moats, bargaining power, brand positioning |
| 4 | Supply Chain | Upstream to end customer; single-source risk |
| 5 | Segments | Revenue / EBITDA / earnings breakdown; non-core segment probe |
| 6 | Earnings Result | Latest quarter vs. consensus; quarterly earnings strip; operating CF vs. NPAT check; YoY sign-artifact caution |
| 7 | Earnings Commentary & Analyst Notes | IRIS notes + giải trình letters for Vietnamese names; earnings call transcripts for US/global |
| 8 | Management | Track record, skin in the game, capital allocation; SOE parent-group direction |
| 9 | Stock Price Analysis | Historical catalysts behind major moves |
| 10 | Comps | Peer table with EV/Sales, EV/EBITDA, P/E NTM, ROE |
| 11 | Forward Projections | 3-year EPS scenarios: base, bull, bear |
| 12 | Red Flags | Forensic review of income statement, balance sheet, cash flow; FS notes sourcing guidance |
| 13 | Management Questions | 15 high-information-value questions for the CEO |
| 14 | Devil's Advocate | Short-seller dismantling of the bull case |

For full reports, all 14 run. For targeted requests ("give me the bull/bear on X", "red flags for Y"), only the relevant sections run.

---

## Data gathering — two-tier model

### Tier 1 — FiinData MCP (Vietnamese stocks only)

For HOSE/HNX names, FiinData is the primary data source. The skill runs these calls in order before writing any section:

1. **`get_stock_snapshot`** — price, multiples, latest NPATMI, broker consensus summary, recent news. Sentiment tools (`include_sentiment=false`) are excluded. `include_sector_metrics` is called but returns structured data only for Banking, Steel, and Power sectors — for other sectors (Cement, Consumer, Real Estate, etc.) it is informational at best.

2. **`FA_Annual`** — 6-year historical P&L, balance sheet, and cash flow via `mssql_read_data`. Full keycode list specified in SKILL.md.

3. **`FA_Quarterly`** — last 8 quarters of P&L and operating CF, used for the mandatory earnings quality strip in Section 6. The date field is a `DATE` string column in the format `"2026Q1"` — not a `QUARTER` column. Describe the table first if queries fail.

4. **`Market_Data`** — 3–5 years of daily price history, used to identify and map ≥5% catalyst moves for Section 9. Field is `TRADE_DATE`.

5. **`IRIS_Company_Comments`** — last 6–8 analyst notes for the ticker (`ISDELETED=0`, `ORDER BY CREATEDATE DESC`). Dragon Capital internal per-quarter commentary with Impact ratings (Positive / Neutral / Negative). Primary source for Section 7 for Vietnamese names. If empty for the ticker, fall through to Tier 2.

6. **`supabase_consensus_analyze`** — multi-broker TP table, NPATMI estimates, sentiment shifts, catalyst tracker. Primary source for Sections 10 and 11.

7. **Peer `Market_Data`** — same-day snapshot for 4–6 domestic sector peers, used in the Section 10 comps table.

**Excluded from all runs:** `zalo_*` and `f319_*` tools are not called.

### Tier 2 — web_search (all markets; supplement for Vietnamese stocks)

Used for: management background, governance context, international peer multiples, giải trình KQKD letters and AGM presentations (Section 7 supplement), recent news not in the FiinData snapshot, and all US/global stocks where Tier 1 does not apply.

---

## Key analytical behaviours

- **Adversarial by default.** Bull and bear construction, then a separate Devil's Advocate pass. Counteracts the model's tendency toward balanced-but-toothless commentary.

- **Data-first.** All Tier 1 pulls complete before writing begins. No fabrication — gaps noted explicitly.

- **Earnings quality strip required.** Section 6 strips non-recurring items from the last 2–3 quarters to compute a core annualised run-rate. If materially below consensus, the gap is flagged. YoY growth figures from the snapshot are always verified against the FA_Quarterly strip — the snapshot's `profit_growth_yoy` field produces misleading results when the prior-period base is negative (swing-to-profit names).

- **IRIS notes as earnings call substitute.** For Vietnamese names, `IRIS_Company_Comments` is pulled as Tier 1 and used in Section 7 to construct a multi-quarter sentiment arc. Functionally equivalent to earnings call transcript analysis for US names.

- **Operating EV for cash-rich names only.** When net cash exceeds 25% of market cap, Section 10 computes and discloses the operating EV and restates EV/EBITDA. When the company is in net debt or below the threshold, the check is omitted silently — no mention in the report.

- **Cost pass-through sensitivity for commodity producers.** Section 11 includes a ±10% input cost sensitivity sub-section for cement, steel, chemicals, and similar names. This is typically the dominant earnings variable and should not be buried in generic scenario text.

- **SOE-adjusted management framing.** For Vietnamese state-controlled enterprises, Section 8 shifts from founder/incentive analysis to capital allocation outcomes, parent-group strategic direction, and related-party revenue concentration.

- **Non-core segment probe.** Section 5 explicitly looks for sidebar businesses inside commodity or single-product names (toll roads, real estate stakes, financial investments). Small by revenue but often disproportionately high-margin or high-risk.

- **FS notes sourcing.** Raw Notes to Financial Statements (related-party transactions, contingent liabilities, accounting policy disclosures, pledge/collateral schedules) are not available in FiinData. Section 12 notes this explicitly and instructs Claude to query `ReportFileChunks` (MongoDB/IRIS) as a partial proxy and to flag when primary FS filings need to be read directly.

- **Verdict required.** Full reports end with an explicit Buy / Hold / Avoid, a one-paragraph rationale, and a 3-price risk/reward table. Hedge language is allowed within the rationale but not in place of a call.

- **Honest about gaps.** When data is unavailable, the skill instructs Claude to note the gap and fall through to web_search rather than fabricate.

---

## Markets supported

**Vietnam (HOSE / HNX)** — Primary source: FiinData MCP (7-call Tier 1 protocol). Supplementary: CafeF, Vietstock, theinvestor.vn, HOSE/HNX disclosures, SSI Research, VNDirect. Reports financials in VND with USD conversion at ~25,000 VND/USD for comps. Sector-specific dynamics: banking (Basel compliance, NPL ratios); real estate (land bank, legal risk); energy (FiT, DPPA, QHĐ8); O&G services (block FIDs, upstream capex cycles). SOE governance framing and management guidance sandbag check included.

**United States (NYSE / NASDAQ)** — Sources: SEC filings (10-K, 10-Q, 8-K), Seeking Alpha and Motley Fool earnings transcripts, Bloomberg, FactSet consensus. Tier 1 does not apply; `web_search` only.

**Global** — Adapts to local exchange filings and accounting standards (IFRS vs. GAAP where relevant). Financials normalized to USD for comparability.

---

## Repository structure

```
claude-skills/
├── README.md                   ← You are here
├── CHANGELOG.md                ← Plain-English summary of each release
├── ROADMAP.md                  ← Planned improvements and backlog
├── stock-analysis/
│   └── SKILL.md                ← The skill itself (the only file you edit)
├── retrospectives/
│   └── v1.0.md                 ← Post-release lessons learned
└── case-studies/
    └── PLTR-2026-04.md         ← Sample outputs used for benchmarking
```

**The golden rule:** `SKILL.md` is the single source of truth. Claude.ai and any shared copies are downstream of this file. Always edit here first.

---

## How to use this skill in Claude.ai

### As an installed Skill (auto-invoked)

This is the preferred route for daily use. Claude auto-invokes the skill whenever you ask about a stock — no need to reference it manually in chat.

1. In this repo, click the green **Code** button → **Download ZIP**.
2. Extract the ZIP. Locate the `stock-analysis/` folder inside.
3. Re-zip just the `stock-analysis/` folder (right-click → Compress / Send to → Compressed folder).
4. In Claude.ai, go to **Customize → Skills** → click **+** → **+ Create skill** → upload the ZIP.
5. Toggle the skill on.

> **Note on the `.skill` file format:** If you have a file ending in `.skill`, it is a ZIP archive with a renamed extension — this is how Claude.ai packages skills when you download them. Rename it to `.zip` and extract normally to recover the `SKILL.md` inside. GitHub cannot edit compressed files, so always work with the extracted `SKILL.md` directly.

### As project knowledge (for development and iteration)

Use this route when actively refining the skill. The GitHub connector lets Claude always read the latest version of your file without re-uploading anything.

1. In Claude.ai, create or open a Project (e.g., *"Stock Analysis — Skill Dev"*).
2. In the project's knowledge section, click **+** → **GitHub** → authenticate → select this repo.
3. Choose `stock-analysis/SKILL.md` from the file browser → confirm.
4. Add custom instructions to the project:
   > *"This is a development environment for refining my stock-analysis skill. Follow the framework in SKILL.md from project knowledge. At the end of every analysis, flag any places where the framework was unclear, missing context, or produced weak output under a section called 'Skill Feedback'."*
5. When you push changes to GitHub, click **Sync now** in the project's knowledge panel to pull the latest version.

---

## Example prompts

```
Analyze Palantir (PLTR).
→ Full 14-section report with verdict.

Give me the bull and bear case on Vinhomes (VHM:HOSE).
→ Section 2 only.

What are the red flags for Tesla?
→ Section 12 only.

Forensic deep-dive on Hoa Phat Group (HPG:HOSE) — full report.
→ All sections, weighted toward Section 12.
```

---

## Development workflow

The skill lives in two states at any time: **dev** (where you iterate) and **stable** (what you use for real analysis). Think of it like a financial model: you maintain a working copy for testing scenarios and a locked production version for actual memos. You never edit the production version directly.

| State | Lives in | Purpose |
|---|---|---|
| **Dev** | Claude.ai Project + GitHub connector | Test changes, run benchmarks, iterate freely |
| **Stable** | Claude.ai Customize → Skills (uploaded ZIP) | Trusted version for analysis you act on |

GitHub is the source of truth that feeds both. Only tested, tagged versions get promoted from dev to stable.

---

### Phase A — One-time setup

Pick **2–3 benchmark stocks** you know well. These are your regression tests — you run the same tickers before and after every change to verify the skill improved and nothing broke.

A good benchmark mix:
- One Vietnamese name (e.g., VHM or HPG) — tests local market sourcing and VN-specific sections.
- One US name (e.g., PLTR) — tests SEC-sourced analysis and comps.
- One edge case (small-cap, bank, or conglomerate) — stresses the framework's weaker areas.

Save the initial outputs from these benchmarks in `case-studies/` (e.g., `case-studies/VHM-baseline.md`). These become your comparison point for every future change.

---

### Phase B — The iteration loop

Repeat this cycle each time you want to improve the skill.

**1. Capture the change as a GitHub Issue.**
Go to your repo → **Issues** → **New issue**. Be specific:
`[Section 12] Red Flags too generic for VN real estate — missing land bank legal risk, Decree 100 exposure`
Issues give you a prioritized backlog and a traceable history for every change you make.

**2. Create a branch.**
In GitHub Desktop: **Current Branch → New Branch**. Name it after the issue: `vn-redflags-section-12`. This isolates your edits from `main`. If the change doesn't work, you can discard the branch with no consequences to the stable version.

**3. Edit `SKILL.md`.**
See *Editing the skill* section below for tool options. Make your changes, save, commit with a descriptive message:
```
Add VN real estate red flags: land bank legal status, related-party land deals, Decree 100 exposure
```
Push the branch to GitHub.

**4. Switch the project to the dev branch.**
In Claude.ai, open your dev project → GitHub knowledge panel → branch dropdown → switch from `main` to your new branch → **Sync now**. Claude now reads the experimental version.

**5. Run benchmark tests.**
Start a new chat in the project: *"Analyze VHM:HOSE — focus on Section 12 Red Flags."* Compare against your baseline. Ask yourself: Did this add real insight? Did anything break? Is Claude correctly applying the new instructions?

**6. Merge or discard.**
- **Good:** Open a Pull Request on GitHub from your branch to `main`. Review the diff. Merge. Switch the project connector back to `main` and sync.
- **Needs work:** Make more commits to the same branch, sync the project, retest. Repeat until satisfied.
- **Didn't work:** Delete the branch. `main` is untouched.

**7. Close the Issue** with a note linking to the merged change.

---

### Phase C — Promoting to stable

Don't promote after every merge. Batch 3–5 meaningful improvements, then cut a release.

1. **Tag a release on GitHub.** Go to **Releases → Draft a new release → Create new tag** (e.g., `v1.2`). Write release notes summarizing what changed since the previous version.
2. **Download the tagged ZIP** from the release page (Source code → zip).
3. **Extract and re-zip just the `stock-analysis/` folder.** The GitHub ZIP wraps the whole repo — Claude.ai needs only the skill folder.
4. **Replace the production skill in Claude.ai.** Go to **Customize → Skills** → find the old version → **Delete** → **+ Create skill** → upload the new ZIP.
5. **Smoke-test** on your benchmark stocks in a normal chat (not the dev project) to confirm the uploaded version behaves correctly.

A reasonable cadence: iterate throughout the month, cut one production release every 4–6 weeks.

---

## Editing the skill

`SKILL.md` is a plain text file. Any tool that edits plain text works — no special software is required.

### Option 1 — GitHub web editor (no install, quick edits)
Navigate to `stock-analysis/SKILL.md` on github.com → click the **pencil icon** (top-right of the file view) → edit in the browser → commit at the bottom. Ideal for small changes (a paragraph, a bullet point); clunky for major rewrites.

### Option 2 — github.dev (no install, full editor — recommended)
On any page in your repo, press the **period key (`.`)** on your keyboard. A full editor loads in the browser — file tree on the left, find-and-replace, markdown preview. Commit and push without leaving the tab. Best option for substantial edits with no installation.

### Option 3 — Any plain text editor on your computer
After cloning the repo via GitHub Desktop, the file lives locally at `claude-skills/stock-analysis/SKILL.md`. Open it with any plain text editor (Notepad on Windows, TextEdit in plain-text mode on Mac, Notepad++ for something better). After saving, GitHub Desktop shows the change — commit and push from there.

> ⚠️ **Do not edit in Microsoft Word or Google Docs.** They inject invisible formatting characters that break markdown rendering and confuse Claude when reading the file. If you paste content from Word, run it through a plain-text intermediary (the github.com editor, or Notepad) first to strip the formatting.

### Option 4 — Claude itself (best for substantial rewrites)
The recommended approach for meaningful section improvements. Open a chat in your dev project and work conversationally:
*"Here's my current Section 12. I want to add a VN real estate overlay covering land bank legal risk and Decree 100 exposure. Show me the revised section only."*
Iterate until the output is sharp, then paste the result into `SKILL.md` via Options 1 or 2 and commit. This is how the skill should evolve long-term — using Claude to improve a Claude skill, with GitHub storing the result.

---

## Methodology notes

- **Adversarial by default.** The skill forces both bull and bear construction, then runs a separate Devil's Advocate pass. This counteracts the model's tendency toward balanced-but-toothless commentary on contested names.
- **Data-first.** The Data Gathering Protocol runs *before* any section is written. Claude is instructed not to lean on training data for prices, financials, or recent events — those must come from live searches.
- **Verdict required.** Full reports end with an explicit Buy / Hold / Avoid, with a one-paragraph rationale. Hedge language is allowed within the rationale but not in place of a call.
- **Honest about gaps.** When data is unavailable (e.g., no transcripts found for a small-cap), the skill instructs Claude to note the gap rather than fabricate.

---

## Version history

See [CHANGELOG.md](./CHANGELOG.md) for a full release history.

Suggested commit message convention:
```
v1.0  Initial 14-section framework. Tested on PLTR (Hold/Avoid verdict).
v1.1  Added Vietnamese-specific sources to data gathering protocol.
v1.2  Tightened Section 12 (Red Flags) with three-statement structure.
v1.3  Two-tier data gathering: FiinData MCP as Tier 1 for Vietnamese stocks,
      web_search as Tier 2 / sole source for US/global. Excluded F319 and
      Zalo sentiment tools. Added quarterly earnings strip to Section 6.
      Added operating EV instruction to Section 10. Added SOE governance
      framing and guidance sandbag check to Market-Specific Notes.
      Output format changed from markdown to self-contained HTML file.
v1.4  Added IRIS_Company_Comments as Tier 1 pull (Step 5 in protocol).
      Renamed Section 7 to "Earnings Commentary & Analyst Notes" with
      Vietnamese sub-protocol using IRIS notes + giải trình letters.
      Added FS notes sourcing guidance to Section 12 (ReportFileChunks
      as partial proxy; primary filing required for forensic analysis).
      Operating EV check in Section 10 now silently omitted when company
      is in net debt or below threshold — no "not applicable" language.
      Added FA_Quarterly DATE field convention note (format "2026Q1").
      Added profit_growth_yoy sign-artifact caution to Section 6.
      Added sector_metrics caveat (meaningful for Banking/Steel/Power only).
      Added cost pass-through sensitivity sub-section to Section 11.
      Added non-core segment probe to Section 5 and Market-Specific Notes.
      Added parent-group AGM probe to SOE governance note in Section 8.
```

When re-uploading to Claude.ai, always delete the previous version under Customize → Skills first to avoid duplicate skill triggers.

---

## Disclaimer

This skill produces analysis for informational and educational purposes only. Output should not be construed as investment advice, a solicitation, or a recommendation to buy or sell any security. Always verify Claude's data against primary sources before acting on any analysis.

---

## License

Personal use. Not licensed for redistribution.
