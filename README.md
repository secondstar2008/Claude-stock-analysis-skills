# Stock Analysis Skill

An institutional-grade equity research framework for [Claude.ai](https://claude.ai), designed to produce deep-dive, adversarial analysis on stocks across Vietnamese (HOSE/HNX), US (NYSE/NASDAQ), and global markets.

This skill mirrors the workflow of a buy-side analyst preparing a recommendation memo: data gathering, bull/bear construction, competitive analysis, comps, forward projections, red-flag screening, and a final verdict.

---

## What this skill does

Given a ticker or company name, the skill instructs Claude to:

1. Pull current, real data from primary and secondary sources (financial filings, analyst research, earnings transcripts) rather than rely on training data.
2. Run a structured 14-section analysis — or only the sections requested.
3. Output a clean, formatted report with a **TL;DR**, body, and **Buy / Hold / Avoid verdict**.

The output is intentionally adversarial: the framework forces the analyst (Claude) to argue both sides, perform a pre-mortem, and play short-seller before reaching a conclusion.

---

## The 14-section framework

| # | Section | Purpose |
|---|---|---|
| 1 | Company Overview | Business model, customers, suppliers, contracts |
| 2 | Bull vs. Bear | Adversarial deep-dive; pre-mortem; contrarian view |
| 3 | Competitive Advantages | Moats, bargaining power, brand positioning |
| 4 | Supply Chain | Upstream to end customer; single-source risk |
| 5 | Segments | Revenue / EBITDA / earnings breakdown |
| 6 | Earnings Result | Latest quarter vs. consensus, drivers, market reaction |
| 7 | Earnings Calls | Management tone and sentiment shift over 2–4 quarters |
| 8 | Management | Track record, skin in the game, capital allocation |
| 9 | Stock Price Analysis | Historical catalysts behind major moves |
| 10 | Comps | Peer table with EV/Sales, EV/EBITDA, P/E NTM, ROE |
| 11 | Forward Projections | 3-year EPS scenarios: base, bull, bear |
| 12 | Red Flags | Forensic review of income statement, balance sheet, cash flow |
| 13 | Management Questions | 15 high-information-value questions for the CEO |
| 14 | Devil's Advocate | Short-seller dismantling of the bull case |

For full reports, all 14 run. For targeted requests ("give me the bull/bear on X", "red flags for Y"), only the relevant sections run.

---

## Markets supported

**Vietnam (HOSE / HNX)** — Sources: CafeF, VNDirect, SSI Research, HOSE/HNX disclosures, FiinGroup, Vietstock. Reports financials in VND with USD conversion for comps. Aware of sector-specific dynamics: banking (Basel compliance, NPL ratios), real estate (land bank, legal risk), energy (FiT, DPPA, QHĐ8).

**United States (NYSE / NASDAQ)** — Sources: SEC filings (10-K, 10-Q, 8-K), Seeking Alpha and Motley Fool earnings transcripts, Bloomberg, FactSet consensus.

**Global** — Adapts to local exchange filings and accounting standards (IFRS vs. GAAP where relevant). Financials normalized to USD for comparability.

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
Analyze Palantir (PLTR).
→ Triggers the full 14-section report with verdict.

Give me the bull and bear case on Vinhomes (VHM:HOSE).
→ Runs Section 2 only.

What are the red flags for Tesla?
→ Runs Section 12 only.

Forensic deep-dive on Hoa Phat Group (HPG:HOSE) — full report.
→ Triggers all sections, weighted toward Section 12.
```

---

## Methodology notes

A few design choices worth flagging for anyone using or modifying the framework:

- **Adversarial by default.** The skill forces both bull and bear construction, then runs a separate Devil's Advocate pass. This counteracts the model's tendency toward balanced-but-toothless commentary on contested names.
- **Data-first.** The Data Gathering Protocol runs *before* any section is written. Claude is instructed not to lean on training data for prices, financials, or recent events — those must come from live searches.
- **Verdict required.** Full reports end with an explicit Buy / Hold / Avoid, with a one-paragraph rationale. Hedge language is allowed within the rationale but not in place of a call.
- **Honest about gaps.** When data is unavailable (e.g., no transcripts found for a small-cap), the skill instructs Claude to note the gap rather than fabricate.

---

## Version history

Use Git commits to track meaningful methodology changes. Suggested convention for commit messages:

```
v1.0  Initial 14-section framework. Tested on PLTR (Hold/Avoid verdict).
v1.1  Added Vietnamese-specific sources to data gathering protocol.
v1.2  Tightened Section 12 (Red Flags) with three-statement structure.
```

When re-uploading to Claude.ai, delete the previous version under Customize → Skills first to avoid duplicate triggers.

---

## Disclaimer

This skill produces analysis for informational and educational purposes only. Output should not be construed as investment advice, a solicitation, or a recommendation to buy or sell any security. Always verify Claude's data against primary sources before acting on any analysis.

---

## License

Personal use. Not licensed for redistribution.
