---
name: stock-analysis
description: >
  Deep-dive stock analysis tailored to the user's framework. Use this skill whenever the user asks to analyze, research, or evaluate a stock, company, or equity — on any market (Vietnam HOSE/HNX, US NYSE/NASDAQ, or global). Also trigger when the user asks about a company's fundamentals, earnings, management, competitive position, valuation, or red flags. Trigger even for partial requests like "give me the bull/bear on X", "what are the red flags for Y", "analyze Z's earnings", or "what do you think about [ticker]". This skill covers both full multi-section reports and quick single-section dives.
---

# Stock Analysis Skill

Perform institutional-grade equity analysis on any stock, for any market, using the framework below. The user may want all sections or just specific ones — read the request and respond accordingly.

---

## How to Use This Skill

1. **Identify the stock**: Extract ticker + exchange if given (e.g., VNM:HOSE, AAPL:NASDAQ). If ambiguous, clarify.
2. **Determine scope**: Full report vs. specific section(s). If the user says "analyze X", run all sections. If they say "give me the bull/bear on X", run only that section.
3. **Detect sector** (Vietnamese stocks only): Identify which sector the company belongs to and activate the corresponding deep-data layer in FiinData. See Sector Data Routing below.
4. **Gather data**: Follow the tiered Data Gathering Protocol below. Do NOT rely on training data alone for financials, prices, or recent events.
5. **Write each section** using the exact prompts below as your analytical framework.
6. **Format the output** cleanly using markdown headers per section.

---

## Data Gathering Protocol

### For Vietnamese Stocks (HOSE / HNX) — FiinData-Native Workflow

FiinData is Dragon Capital's proprietary research platform. Treat it as the primary source for Vietnamese stocks; use web_search only to fill specific gaps. Query in this order:

**Step 0 — Internal analyst notes (always first):**
- `IRIS_Company_Comments` (filter `ISDELETED = 0`, ORDER BY `CREATEDATE DESC`) — internal Dragon Capital analyst notes: earnings flashes, "coffee with management" debriefs, stock-in-focus updates. These take priority over all external sources. A DC analyst's earnings flash is worth more than five broker reports for Sections 6, 7, and 13.

**Step 1 — Snapshot and market data:**
- `get_stock_snapshot` (all flags on) — price, valuation, sentiment summary, broker consensus summary
- `Market_Data` (last 90 days) — OHLCV, foreign flow (matched + deal split), foreign holding %, foreign room remaining, block deal activity
- `DC_holding` (last 30 days) — Dragon Capital fund positioning (ETF / onshore / offshore). Detect accumulation or distribution.

**Step 2 — Financials:**
- `FA_Quarterly` via `mssql_read_data` — 8 quarters of key FS lines (Net_Revenue, Gross_Profit, Gross_Margin, EBITDA, NPATMI, Operating_CF, FCF, Total_Asset, Total_Liabilities, TOTAL_Equity, ST_Debt, LT_Debt, Net_DER). YoY column is stored as ratio change (0.45 = +45%) — convert before displaying.
- `FA_Annual` — for multi-year trend analysis
- `CompanyModels` via `mongo_find` — analyst's full segment-level P&L model with quarterly and annual data, plus forward estimates. Includes Operation.Secured_Backlog and Operation.Potential_Backlog when populated.

**Step 3 — Research and consensus:**
- `supabase_consensus_analyze` (comparisons=5) — external broker target prices, NPATMI forecast spread, sentiment shifts
- `Forecast` table — Dragon Capital's house forecast. Cross-check against consensus — house view divergence is signal.

**Step 4 — Sentiment:**
- `f319_stock_thesis_get` — community thesis (flag if release_date > 6 months old)
- `f319_stock_discussion_points` (sort_by="signal", last 90 days) — flag if < 2 distinct threads
- Zalo signals from `get_stock_snapshot` — flag if recommendations < 5

**Step 5 — Order flow context (optional, for Section 9):**
- `stock_cvd` — intraday tick-level buy/sell pressure (only available for ~70 large-cap stocks)

**Step 6 — Sector deep layer:** see Sector Data Routing below.

**Step 7 — Mandatory web_search:** audited annual report for FS notes. FiinData does not carry FS notes. This step is non-negotiable before completing Section 12 (Red Flags).

---

### Sector Data Routing — Vietnamese Stocks

After identifying the ticker, determine sector and activate the relevant deep layer:

| Sector | FiinData Sources | What it adds |
|---|---|---|
| **O&G / O&G Services** | `O&GProjectsData` (MongoDB) | Project-level pipeline: Capex, reserves, investors, EPC status, first-oil dates. Cross-reference with `CompanyModels.Operation.Secured_Backlog` |
| **Real Estate** | `RE_Company_Project_RNAV_Detail`, `RE_Market_State_Quarterly`, `RE_Ministry_of_Construction_Quarterly_Data`, `RE_CBRE_Quarterly_Market_Report`, `RE_DXS_Quarterly_Market_Report`, `RE_Stock_Valuation_Latest` | RNAV by project, market state, CBRE/DXS research |
| **Banking** | Use `query_banking_credit` tool (pre-built; covers NPL, NIM, ROE/ROA, deposit rates, write-offs, system-wide credit/deposit) | Specialized banking sector tool — preferred over raw queries |
| **Steel** | `Steel_data`, `ThiTruongThepPrices` (MongoDB) | Steel sector data, input cost pricing |
| **Power** | `Power_Projects`, `Power_Company_Operations`, `Power_Metrics`, `Power_Reservoir_Metrics`, `Power_Reference` | Plant-level operations, reservoir levels (hydro), project pipeline |
| **Aviation** | `Aviation_Operations`, `Aviation_Revenue`, `Aviation_Airfare`, `Aviation_Market` | Load factors, route-level revenue, airfare pricing |
| **Brokerage / Securities** | `BrokerageMetrics` (KEYCODE format, 201 codes), `Brokerage_Propbook`, `Brokerage_Market_Share` | Margin lending, prop trading, commissions, market share |
| **Agriculture** | `AgroMonitor`, `AgroMonitorDaily` | Commodity/crop monitoring |
| **Other / Macro context** | `macro_data_consolidated`, `ceic_macro_data`, `Commodity` | Vietnam macro series, commodities |

If no sector match, proceed with universal layer only and note the gap in the output.

---

### For US Stocks (NYSE / NASDAQ)

Use `web_search` for: SEC filings (10-K, 10-Q, 8-K), earnings call transcripts (Seeking Alpha, Motley Fool), Bloomberg / FactSet consensus, analyst notes. Note any relevant regulatory or antitrust exposure.

### For Global Stocks

Adapt to local exchange filings and accounting standards (IFRS vs. GAAP). Normalize financials to USD for comparability.

---

## Data Quality Rules

These rules apply to every output and must be respected silently in the background:

1. **Revenue fallback.** `get_stock_snapshot` has a known bug where `Net_Revenue` returns null. Always cross-check against `FA_Quarterly` directly when revenue figures matter.

2. **Minimum signal thresholds.** Do not treat thin samples as actionable consensus:
   - F319: require ≥ 2 distinct threads (not multiple re-analyses of one thread) before calling community sentiment meaningful
   - Zalo: require ≥ 5 recommendations before stating a consensus; below that, note "thin sample"
   - Broker consensus: note when only 1–2 brokers are represented vs. 5+

3. **YoY column interpretation.** `FA_Quarterly.YoY` is stored as a ratio change (0.45 = +45%), not percentage points. Convert before displaying — especially critical when YoY appears alongside margin figures (which are already in decimal form) to avoid misreading.

4. **Staleness checks.** Flag F319 thesis as stale if `release_date` > 6 months old. Empty `bear_points: []` on a stale thesis should be treated as missing data, not as evidence there are no bear arguments.

5. **DC coverage detection.** If `IRIS_Company_Comments`, `CompanyModels`, and `Forecast` all return empty for a Vietnamese ticker, the stock is not under DC coverage. Note this gap and rely more heavily on external broker consensus and web_search.

---

## Analytical Sections

Run each section using the framework below. For full reports, run all 14. For targeted requests, run only the relevant section(s).

### Section-to-Data-Source Mapping (Vietnamese stocks)

| Section | Primary FiinData Source | Notes |
|---|---|---|
| 1. Company Overview | `CompanyModels` segments + sector deep layer | |
| 2. Bull vs. Bear | `IRIS_Company_Comments`, `f319_stock_thesis_get`, `f319_stock_discussion_points` | |
| 3. Competitive Advantages | `CompanyModels` margin by segment vs. peers | |
| 4. Supply Chain | Sector deep layer (e.g., `O&GProjectsData` for O&G) | |
| 5. Segments | `CompanyModels.Financial.Breakdown` | |
| 6. Earnings | `IRIS_Company_Comments` (earnings flash), `FA_Quarterly` (last 2 quarters), `supabase_consensus_analyze` | |
| 7. Earnings Calls | `IRIS_Company_Comments` (management meeting notes), `supabase_consensus_analyze` comparison reports | |
| 8. Management | `IRIS_Company_Comments`, `Market_Data.FOREIGN_HOLDING_PCT` trend, web_search for insider transactions | |
| 9. Stock Price Analysis | `Market_Data` (90d price + foreign flow), `DC_holding` (30d positioning), `MarketIndex` for beta, `stock_cvd` for tape colour | |
| 10. Comps | `FA_Annual` for subject; web_search for peer multiples | |
| 11. Forward Projections | `CompanyModels` forward estimates, `Forecast` (house view), `supabase_consensus_analyze` (street view) | |
| 12. Red Flags | `FA_Quarterly` (CF vs. NI divergence) + **mandatory web_search for FS notes** | |
| 13. Management Questions | `IRIS_Company_Comments` (gaps in DC's own analyst notes are good question seeds) | |
| 14. Devil's Advocate | `f319_stock_discussion_points` (sentiment_filter="bearish"), `supabase_consensus_analyze` (HSC and other conservative brokers) | |

---

### 1. Company Overview

Explain the company's business model in simple terms. What are its key products and services? Who are its main customers, suppliers, and competitors? What are the contracts and key payment terms?

---

### 2. Bull vs. Bear

*Act as an institutional-grade equity analyst. Perform a deep-dive, adversarial analysis of the company.*

- **Bull case**: Competitive advantages, moat sustainability, growth levers (secular tailwinds or potential earnings surprises), capital allocation quality.
- **Bear case**: 2–3 risks that could permanently impair the business, potential margin compression or revenue deceleration, high expectations risk.
- **Pre-mortem**: What would have to go wrong for this to be a bad investment?
- **Valuation check**: Are current multiples pricing in too much optimism?
- **Contrarian view**: What is the market currently refusing to see — in either direction?

---

### 3. Competitive Advantages

Analyze the strength of the company's products and market position:
- How do products/services compare to competitors in perceived value, branding, and marketing?
- What economic moats protect the business from future competition? (network effects, switching costs, cost advantages, intangible assets, efficient scale)
- What is the company's bargaining power over customers, suppliers, and other stakeholders?

---

### 4. Supply Chain

Map the supply chain from upstream inputs to end customer. Include:
- All major input suppliers (with company names where known)
- The company's position in the value chain
- Key distributors, logistics partners, and retailers
- End customers / demand drivers
- Single-source dependencies or supply chain vulnerabilities

For Vietnamese sector-heavy names, cross-reference the sector deep layer (e.g., `O&GProjectsData` shows project investors, blocks, and status; this maps the upstream customer base for O&G services companies like PVS).

---

### 5. Segments

Break down revenue, EBITDA, and earnings by segment:
- Product/service segments
- Geographic segments
- How have each of these changed over time and why?
- Which segments are growing, declining, or under margin pressure?

For Vietnamese stocks, pull from `CompanyModels.Financial.Breakdown` which carries revenue and gross profit by segment quarterly back to 2017 and forward to forecast years.

---

### 6. Earnings Result

Analyze the company's most recent quarterly/annual earnings:
- **Revenue & profit vs. expectations**: Did the company beat or miss consensus? By how much?
- **Segment drivers**: Which business lines drove the result? Any notable acceleration or deceleration?
- **Margin trends**: What happened to gross/operating margins and why? Flag any accounting-change effects (reclassifications, provision reversals, one-off items) that inflate or deflate the reported margin.
- **Quarterly earnings strip** (required): Using the last 2–3 quarters from FA_Quarterly, identify non-recurring items (provision reversals, one-off FX, asset-sale gains). Strip these out to compute a *core* quarterly run-rate and annualise it. Compare to consensus NPATMI. If core annualised profit is materially below consensus, flag the gap explicitly. Caution: the snapshot's `profit_growth_yoy` field is unreliable when the prior-period NPATMI is negative — always verify against the FA_Quarterly strip directly before citing YoY growth for any swing-to-profit name.
- **Guidance & outlook**: What did management guide for next quarter/full year? Check whether guidance appears conservative vs. the run-rate implied by the quarterly strip.
- **Balance sheet flags**: Anything notable in cash flow, inventory, receivables, or debt?
- **Operating CF vs. NPAT**: Explicitly compare operating cash flow to net profit for the latest period. A sustained divergence (positive NPAT, negative or weak operating CF) is a red flag requiring explanation.
- **Market reaction**: How did the stock react, and what does that signal about what was priced in?
- Flag anything unusual relative to the company's recent history.

For Vietnamese stocks, prioritize `IRIS_Company_Comments` earnings flash notes — they contain the specific provision/reversal breakdowns and non-core vs. core decomposition that external sources rarely have.

---

### 7. Earnings Commentary & Analyst Notes

Synthesize management tone and analyst commentary across the last 4–6 earnings cycles. The goal is to surface a multi-quarter sentiment arc — has the tone shifted from cautious to constructive, or vice versa? Flag any notable changes in language around guidance, risk, or capital allocation.

**For Vietnamese stocks**: Primary sources are (1) `IRIS_Company_Comments` (Tier 1 pull — DC internal analyst notes with Impact ratings, per quarter), and (2) giải trình kết quả kinh doanh letters and AGM presentations (Tier 2 web search). Synthesize across both: note the Impact rating trend (e.g., Negative → Neutral → Positive), flag any inflection quarters, and extract any forward-looking signals embedded in the commentary (capex ramp, discount/pricing commentary, debt reduction guidance). Vietnamese companies do not host English-language earnings calls — IRIS notes and giải trình letters are the functional equivalent.

**For US/global stocks**: Primary sources are earnings call transcripts (Seeking Alpha, Motley Fool, company IR site). Summarize the last 2–4 calls. Perform sentiment analysis: how has management tone shifted over time?

For Vietnamese stocks, `IRIS_Company_Comments` "Coffee with [ticker]" notes are post-management-meeting debriefs — treat them as transcript proxies.

---

### 8. Management

Assess the CEO and key executives:
1. **Track record**: What have they built, turned around, or delivered in prior roles? Quantify where possible.
2. **Tenure & insider ownership**: How long in role, and how much skin in the game?
3. **Capital allocation history**: Do they reinvest wisely, acquire with discipline, or destroy value? ROE/ROIC trend under their watch.
4. **Red flags**: Related-party transactions, excessive compensation, frequent strategy pivots, or promotional behavior.
5. **Founder vs. professional manager**: Which archetype, and what does that imply for this stage of the business?

For Vietnamese stocks, also check `Market_Data.FOREIGN_HOLDING_PCT` trend and `DC_holding` trend for institutional positioning signals.

---

### 9. Stock Price Analysis

Identify historical catalysts behind the stock price:
- What news or events moved the stock up or down more than 5% in the last 3–5 years?
- Map the stock's major inflection points to the underlying business or macro events.
- What does the pattern of price reactions say about market expectations?

For Vietnamese stocks: combine `Market_Data` (90d price + foreign flow), `DC_holding` (30d institutional positioning), `MarketIndex` (relative performance vs VNINDEX / VN30 / sector index), and `stock_cvd` if available (intraday tape colour).

---

### 10. Comps (Comparable Companies)

Generate a comparables table with the company and its key global peers. Include:
- Ticker (Bloomberg or local exchange format)
- Market cap (USD equivalent)
- EV/Sales
- EV/EBIT or EV/EBITDA
- P/E (NTM)
- Dividend yield
- 5-year average ROE

**Net cash adjustment** (apply only if triggered): Compute net cash = cash + short-term investments − total debt. If net cash is positive *and* exceeds 25% of market cap, compute operating EV (market cap − net cash), restate EV/EBITDA on that basis, and disclose this adjustment prominently in the comps section. If the company is in net debt, or if net cash is below the 25% threshold, omit this sub-section entirely — do not mention the check or note that it was performed.

Comment on where the subject company trades relative to peers and whether the premium or discount is justified.

---

### 11. Forward Projections

Estimate EPS (and where relevant, revenue and EBITDA) for the next 3 years. Consider:
- Industry growth rate and market share trajectory
- Price increases and volume dynamics
- Cost pressures and operating leverage
- Financing costs and interest expense
- Share count changes (dilution or buybacks)

Present base case, bull case, and bear case scenarios. Compare to current analyst consensus.

For Vietnamese stocks: use `CompanyModels` forward estimates as base, `Forecast` as DC house view, and `supabase_consensus_analyze` for street consensus. The spread between these three is itself a signal.

---

### 12. Red Flags

*Act as a forensic equity analyst. Identify red flags and accounting risks.*

Review across three statements:
- **Income statement**: Unusual revenue recognition, aggressive segment reporting, non-recurring items treated as recurring.
- **Balance sheet**: Goodwill/intangibles creep, related-party balances, off-balance-sheet exposure, lease obligations.
- **Cash flow**: Divergence between net income and operating cash flow, working capital manipulation, capex classification.
- **Other**: Stock-based comp as a percentage of earnings, contingent liabilities, auditor changes or qualifications.

**FiinData limitation — mandatory step:** FiinData does not contain notes to financial statements. Before completing this section, always `web_search` for the company's most recent audited annual report and specifically check for:
- Related-party transactions and balances
- Contingent liabilities and off-balance-sheet exposure
- Auditor opinion (qualified, emphasis of matter, going concern)
- Accounting policy changes vs. prior year

If the annual report is unavailable, flag this gap explicitly in the output.

---

### 13. Management Questions

Generate 15 precise questions for the CEO, ordered by information value, covering:
- Long-term competitive strategy
- Capital allocation priorities
- Key risks management is most focused on
- Segment-level performance drivers
- Areas where analyst consensus may be wrong

For Vietnamese stocks, gaps in `IRIS_Company_Comments` (questions DC analysts have not yet answered in their own notes) are particularly good seeds.

---

### 14. Devil's Advocate

*Act as a skeptical short-seller. Dismantle the bull case.*

- What could structurally break the way this company makes money?
- Where is revenue concentrated, and what happens if that concentration shifts?
- Why might the moat be weaker than bulls believe?
- Who is the most dangerous competitor that bulls are underestimating — and why?
- What are the worst examples of management capital allocation?
- What assumptions must hold for the current price to be justified?
- What happens to valuation if growth disappoints by 20–30%?
- What is the single scenario that would permanently impair this business, and how plausible is it?

---

## Output Formatting

- Use clear `##` section headers matching the section names above.
- Lead with a 2–3 line **TL;DR** before the first section (overall impression, key risk, and whether the stock looks interesting at current levels).
- Use tables for Comps and Segments sections.
- Use bullet points within sections; keep prose tight.
- At the end of a full report, include a **Verdict** section: summarize whether the stock is a Buy / Hold / Avoid at current valuation, with a one-paragraph rationale.
- If data was unavailable for a section, note it clearly rather than fabricating.
- For Vietnamese stocks, cite specific data sources when using FiinData (e.g., "per DC analyst note May 6", "per FA_Quarterly Q1 2026", "per O&GProjectsData Block B status").

---

## Market-Specific Notes

**Vietnamese stocks (HOSE/HNX)**:
- Report financials in VND; convert to USD for comps if needed.
- Reference regulatory context: State ownership, FDI limits, foreign room (see `Market_Data.FOREIGNCURRENTROOM` vs `FOREIGNTOTALROOM`).
- Sector-specific dynamics already covered above in Sector Data Routing.

**US stocks (NYSE/NASDAQ)**:
- Reference SEC filings (10-K, 10-Q, 8-K), earnings call transcripts, and analyst consensus.
- Note any relevant regulatory or antitrust exposure.

**Global stocks**:
- Adapt to local regulatory filings and accounting standards.
- Convert financials to USD for comparability.
