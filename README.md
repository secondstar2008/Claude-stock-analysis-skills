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
| 5 | Segments | Revenue / EBITDA / earnings breakdown |
| 6 | Earnings Result | Latest quarter vs. consensus; quarterly earnings strip; operating CF vs. NPAT check |
| 7 | Earnings Calls | Management tone and sentiment shift over 2–4 quarters |
| 8 | Management | Track record, skin in the game, capital allocation |
| 9 | Stock Price Analysis | Historical catalysts behind major moves |
| 10 | Comps | Peer table with EV/Sales, EV/EBITDA, P/E NTM, ROE; operating EV for cash-rich names |
| 11 | Forward Projections | 3-year EPS scenarios: base, bull, bear |
| 12 | Red Flags | Forensic review of income statement, balance sheet, cash flow |
| 13 | Management Questions | 15 high-information-value questions for the CEO |
| 14 | Devil's Advocate | Short-seller dismantling of the bull case |
 
For full reports, all 14 run. For targeted requests ("give me the bull/bear on X", "red flags for Y"), only the relevant sections run.
 
---
 
## Data gathering — two-tier model
 
### Tier 1 — FiinData MCP (Vietnamese stocks only)
 
For HOSE/HNX names, FiinData is the primary data source. The skill runs these calls in order before writing any section:
 
1. **`get_stock_snapshot`** — price, multiples, latest NPATMI, broker consensus summary, recent news. Sentiment tools (`include_sentiment=false`) are excluded.
2. **`FA_Annual`** — 6-year historical P&L, balance sheet, and cash flow via `mssql_read_data`. Full keycode list specified in SKILL.md.
3. **`FA_Quarterly`** — last 8 quarters of P&L and operating CF, used for the mandatory earnings quality strip in Section 6.
4. **`Market_Data`** — 3–5 years of daily price history, used to identify and map ≥5% catalyst moves for Section 9.
5. **`supabase_consensus_analyze`** — multi-broker TP table, NPATMI estimates, sentiment shifts, catalyst tracker.
6. **Peer `Market_Data`** — same-day snapshot for 4–6 domestic sector peers, used in the Section 10 comps table.
**Excluded from all runs:** `zalo_*` and `f319_*` tools are not called.
 
### Tier 2 — web_search (all markets; supplement for Vietnamese stocks)
 
Used for: management background, governance context, international peer multiples, recent news not in the FiinData snapshot, and all US/global stocks where Tier 1 does not apply.
 
---
 
## Markets supported
 
**Vietnam (HOSE / HNX)** — Primary source: FiinData MCP. Supplementary: CafeF, Vietstock, theinvestor.vn, HOSE/HNX disclosures, SSI Research, VNDirect. Reports financials in VND with USD conversion at ~25,000 VND/USD for comps. Aware of sector-specific dynamics (banking: Basel compliance, NPL ratios; real estate: land bank, legal risk; energy: FiT, DPPA, QHĐ8; O&G services: block FIDs, upstream capex cycles). Includes SOE governance framing and management guidance sandbag check.
 
**United States (NYSE / NASDAQ)** — Sources: SEC filings (10-K, 10-Q, 8-K), Seeking Alpha and Motley Fool earnings transcripts, Bloomberg, FactSet consensus. Tier 1 does not apply; `web_search` only.
 
**Global** — Adapts to local exchange filings and accounting standards (IFRS vs. GAAP where relevant). Financials normalized to USD for comparability.
 
---
 
## Output format
 
The final deliverable is a **self-contained HTML file**, not inline markdown. The file is saved to the outputs directory and shared via `present_files`. Key layout elements:
 
- **Fixed left sidebar** with a table of contents linking to all section anchors
- **Sticky header bar** showing ticker, price, analysis date, and verdict badge
- **Data grid header block** with key multiples (PE, PB, EV/EBITDA, consensus TP range)
- **TL;DR panel** — visually distinct, immediately below the header
- **Styled tables** for Segments, Comps, and Forward Projections
- **Verdict callout** — Buy / Hold / Avoid badge (color-coded), one-paragraph rationale, and a 3-row risk/reward table
- **Collapsible Skill Feedback** at the bottom (`<details>` element, outside main report flow)
Typography uses Google Fonts pairings (serif headings / sans body). Color system uses CSS variables with one strong accent color. No animation.
 
---
 
## How to use this skill in Claude.ai
 
### As an installed Skill (auto-invoked)
 
1. Download this repository as a ZIP from GitHub (green **Code** button → **Download ZIP**), or zip the `stock-analysis/` folder locally.
2. In Claude.ai, go to **Customize → Skills** → click **+** → **+ Create skill**.
3. Upload the ZIP. Toggle the skill on.
4. Claude will auto-invoke the skill whenever you ask about analyzing, evaluating, or researching a stock.
### As project knowledge (manual reference)
 
1. Connect this repo via the GitHub connector in a Claude.ai Project.
2. Select `stock-analysis/SKILL.md` as project knowledge.
3. Reference it in chat: *"Apply my stock-analysis framework to VIC:HOSE."*
The Skill route is preferred for daily use; the project-knowledge route is better while iterating on the framework itself.
 
---
 
## Example prompts
 
```
Analyze PVS:HNX.
→ Triggers the full 14-section report, rendered as an HTML file.
 
Give me the bull and bear case on Vinhomes (VHM:HOSE).
→ Runs Section 2 only.
 
What are the red flags for Tesla?
→ Runs Section 12 only.
 
Forensic deep-dive on Hoa Phat Group (HPG:HOSE) — full report.
→ Triggers all sections, weighted toward Section 12.
```
 
---
 
## Methodology notes
 
- **Adversarial by default.** The skill forces both bull and bear construction, then runs a separate Devil's Advocate pass. This counteracts the model's tendency toward balanced-but-toothless commentary on contested names.
- **Data-first.** The Data Gathering Protocol runs *before* any section is written. For Vietnamese stocks, structured FiinData pulls (annual financials, quarterly strip, price history, broker consensus) are the primary source. `web_search` supplements.
- **Earnings quality strip required.** Section 6 explicitly strips non-recurring items from the last 2–3 quarters to compute a core annualised run-rate. If the core run-rate is materially below consensus, the gap is flagged. This catches provision reversals, accounting reclassifications, and one-off FX gains that inflate reported NPATMI.
- **Operating EV for cash-rich names.** When net cash exceeds 25% of market cap, Section 10 computes and discloses the operating EV separately and restates EV/EBITDA on that basis.
- **SOE-adjusted management framing.** For Vietnamese state-controlled enterprises, the skill shifts Section 8 away from founder/incentive analysis toward capital allocation outcomes, parent-group strategic direction, and related-party revenue concentration.
- **Verdict required.** Full reports end with an explicit Buy / Hold / Avoid, a one-paragraph rationale, and a 3-price risk/reward table. Hedge language is allowed within the rationale but not in place of a call.
- **Honest about gaps.** When data is unavailable (e.g., FA_Quarterly returns null for a small-cap), the skill instructs Claude to note the gap and fall through to web_search rather than fabricate.
---
 
## Version history
 
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
```
 
When re-uploading to Claude.ai, delete the previous version under Customize → Skills first to avoid duplicate triggers.
 
---
 
## Disclaimer
 
This skill produces analysis for informational and educational purposes only. Output should not be construed as investment advice, a solicitation, or a recommendation to buy or sell any security. Always verify Claude's data against primary sources before acting on any analysis.
 
---
 
## License
 
Personal use. Not licensed for redistribution.
 







