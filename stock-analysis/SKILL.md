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
3. **Gather data**: Follow the two-tier Data Gathering Protocol below. For Vietnamese stocks, FiinData MCP tools are the primary source; `web_search` supplements. For non-Vietnamese stocks, `web_search` is the sole source. Do NOT rely on training data alone for financials, prices, or recent events.
4. **Write each section** using the exact prompts below as your analytical framework.
5. **Render the output** as a self-contained HTML file saved to `/mnt/user-data/outputs/` and surfaced via `present_files`. See Output Formatting for the HTML spec.

---

## Data Gathering Protocol

Data gathering is two-tiered. Complete **all applicable Tier 1 pulls before writing a single section**. Use Tier 2 to fill gaps Tier 1 cannot cover.

---

### Tier 1 — FiinData MCP (Vietnamese stocks only)

Run `tool_search` first to load FiinData tools, then execute the following calls in order:

1. **Snapshot** — `get_stock_snapshot` with `include_sentiment=false`, `include_broker_consensus=true`, `include_news=true`, `include_sector_metrics=true`. Captures current price, valuation multiples, latest quarterly NPATMI, broker consensus summary, and recent news. Note: `include_sector_metrics` returns meaningful data for Banking, Steel, and Power sectors only — for other sectors (e.g. Cement, Consumer, Real Estate) the field will be sparse or empty; do not expect it to add analytical value.

2. **Annual financials** — `mssql_read_data` on `FA_Annual` for the last 6 years. Pull these keycodes at minimum: `Net_Revenue`, `Gross_Profit`, `Gross_Margin`, `EBIT`, `EBIT_Margin`, `EBITDA`, `EBITDA_Margin`, `NPAT`, `NPATMI`, `NPAT_Margin`, `PBT`, `Financial_Income`, `Financial_Expense`, `Affiliate_Income`, `Other_Income`, `Operating_CF`, `Inv_CF`, `Fin_CF`, `FCF`, `Capex`, `Total_Asset`, `TOTAL_Equity`, `Total_Liabilities`, `Cash`, `Cash_Equivalent`, `Short_Investment`, `ST_Debt`, `LT_Debt`, `Net_DER`, `Account_Receivable`, `Account_Payable`, `Inventory`, `Tangible_Fixed_Asset`, `Goodwill`, `Share_Capital`, `Minority_Interest`. Query template: `SELECT KEYCODE, YEAR, VALUE, YoY FROM FA_Annual WHERE TICKER='[X]' AND YEAR >= [current-6] AND KEYCODE IN (...) ORDER BY KEYCODE, YEAR`.

3. **Quarterly financials (earnings strip)** — `mssql_read_data` on `FA_Quarterly` for the last 8 quarters. Pull: `Net_Revenue`, `Gross_Profit`, `NPAT`, `NPATMI`, `Financial_Income`, `Affiliate_Income`, `Other_Income`, `Operating_CF`. Note: the date column in `FA_Quarterly` uses a `DATE` string field in the format `"2026Q1"` — not a `QUARTER` column. Call `mssql_describe_table` on `FA_Quarterly` first if queries fail. This pull is required for Section 6's quarterly earnings quality strip — do not skip.

4. **Price history** — `mssql_read_data` on `Market_Data` for 3–5 years of daily `PX_LAST`, `VOLUME`, `MKT_CAP`, `PE`. Use this to identify ≥5% single-session moves and map them to catalysts in Section 9. Field is `TRADE_DATE` (not `TRADINGDATE`).

5. **IRIS analyst notes** — `mssql_read_data` on `IRIS_Company_Comments` for the last 6–8 entries. Query: `SELECT TOP 8 TICKER, TITLE, DESCRIPTION, CATEGORY, IMPACT, CREATEDATE FROM IRIS_Company_Comments WHERE TICKER='[X]' AND ISDELETED=0 ORDER BY CREATEDATE DESC`. These are Dragon Capital internal analyst notes tied to quarterly results, with an explicit Impact rating (Positive / Neutral / Negative). They are the primary source for Section 7 for Vietnamese names and provide a multi-quarter sentiment arc. If no records are returned for the ticker, fall through to Tier 2 web sources.

6. **Broker consensus** — `supabase_consensus_analyze` with `comparisons=5`. Captures multi-broker TP table, NPATMI estimates, recent sentiment shifts, and catalyst tracker. This is the primary source for Sections 10 and 11.

7. **Peer market data** — `mssql_read_data` on `Market_Data` for 4–6 domestic sector peers, same-day snapshot: `PX_LAST`, `MKT_CAP`, `PE`. Use for the comps table in Section 10.

**Do not call** `zalo_*`, `f319_*`, or any other sentiment/social tools. These are excluded from this workflow.

If a FiinData query fails or returns null for a critical field, note the gap and fall through to Tier 2 — do not fabricate.

---

### Tier 2 — web_search (all markets; supplement for Vietnamese stocks)

Use `web_search` for everything FiinData cannot provide:
- Management team background, tenure, track record, and any governance red flags
- Regulatory and ownership context (state ownership %, FDI room, legal disputes)
- International peer multiples (EV/EBITDA, P/E) for global comps in Section 10
- Recent news, project announcements, or earnings commentary not in the FiinData snapshot
- For Section 7 (Vietnamese names): giải trình KQKD letters and AGM presentations when IRIS notes are sparse or absent
- Vietnamese stocks: CafeF, Vietstock, theinvestor.vn, HOSE/HNX disclosures

**For US stocks (NYSE/NASDAQ)**: Tier 1 does not apply. Use `web_search` exclusively: SEC filings (10-K, 10-Q, 8-K), earnings call transcripts (Seeking Alpha, Motley Fool), Bloomberg, FactSet consensus.

**For global stocks**: Adapt `web_search` to local exchange/regulator filings and IFRS vs. GAAP context.

---

## Analytical Sections

Run each section using the framework below. For full reports, run all 14. For targeted requests, run only the relevant section(s).

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
- Identify any single-source dependencies or supply chain vulnerabilities

---

### 5. Segments

Break down revenue, EBITDA, and earnings by segment:
- Product/service segments
- Geographic segments
- How have each of these changed over time and why?
- Which segments are growing, declining, or under margin pressure?

Note: Many Vietnamese listed companies do not disclose segment revenue at line-item granularity. Where segment data is unavailable, triangulate from broker reports and management commentary, and flag estimates explicitly as such.

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

---

### 7. Earnings Commentary & Analyst Notes

Synthesize management tone and analyst commentary across the last 4–6 earnings cycles. The goal is to surface a multi-quarter sentiment arc — has the tone shifted from cautious to constructive, or vice versa? Flag any notable changes in language around guidance, risk, or capital allocation.

**For Vietnamese stocks**: Primary sources are (1) `IRIS_Company_Comments` (Tier 1 pull — DC internal analyst notes with Impact ratings, per quarter), and (2) giải trình kết quả kinh doanh letters and AGM presentations (Tier 2 web search). Synthesize across both: note the Impact rating trend (e.g., Negative → Neutral → Positive), flag any inflection quarters, and extract any forward-looking signals embedded in the commentary (capex ramp, discount/pricing commentary, debt reduction guidance). Vietnamese companies do not host English-language earnings calls — IRIS notes and giải trình letters are the functional equivalent.

**For US/global stocks**: Primary sources are earnings call transcripts (Seeking Alpha, Motley Fool, company IR site). Summarize the last 2–4 calls. Perform sentiment analysis: how has management tone shifted over time?

---

### 8. Management

Assess the CEO and key executives:
1. **Track record**: What have they built, turned around, or delivered in prior roles? Quantify where possible.
2. **Tenure & insider ownership**: How long in role, and how much skin in the game?
3. **Capital allocation history**: Do they reinvest wisely, acquire with discipline, or destroy value? ROE/ROIC trend under their watch.
4. **Red flags**: Related-party transactions, excessive compensation, frequent strategy pivots, or promotional behavior.
5. **Founder vs. professional manager**: Which archetype, and what does that imply for this stage of the business?

---

### 9. Stock Price Analysis

Identify historical catalysts behind the stock price:
- What news or events moved the stock up or down more than 5% in the last 3–5 years?
- Map the stock's major inflection points to the underlying business or macro events.
- What does the pattern of price reactions say about market expectations?

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

For commodity-linked producers (cement, steel, chemicals, fertiliser, etc.), add a **Cost Pass-Through Sensitivity** sub-section: model the NPATMI impact of a ±10% move in the key input cost (e.g. coal for cement, iron ore for steel). This is often the single most important variable for these names and should not be buried inside generic scenario narrative.

---

### 12. Red Flags

*Act as a forensic equity analyst. Identify red flags and accounting risks.*

Review across three statements:
- **Income statement**: Unusual revenue recognition, aggressive segment reporting, non-recurring items treated as recurring.
- **Balance sheet**: Goodwill/intangibles creep, related-party balances, off-balance-sheet exposure, lease obligations.
- **Cash flow**: Divergence between net income and operating cash flow, working capital manipulation, capex classification.
- **Other**: Stock-based comp as a percentage of earnings, contingent liabilities, auditor changes or qualifications.

**Notes to Financial Statements**: Raw FS notes (related-party transactions, contingent liabilities, accounting policy disclosures, pledge/collateral schedules, off-balance-sheet items) are not available in FiinData — they require reading the actual PDF filings from HOSE/HNX/SSC directly. As a partial proxy, query `ReportFileChunks` (MongoDB, IRIS database) filtered by ticker: `db.ReportFileChunks.find({ tickers: '[X]', chunkType: { $in: ['risk', 'governance', 'related_party'] } })` — DC broker reports sometimes surface FS note items explicitly. If this returns relevant material, cite it; if not, note that FS note review requires the primary filing.

---

### 13. Management Questions

Generate 15 precise questions for the CEO, ordered by information value, covering:
- Long-term competitive strategy
- Capital allocation priorities
- Key risks management is most focused on
- Segment-level performance drivers
- Any areas where analyst consensus may be wrong

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

The final deliverable is a **self-contained HTML file**, not inline markdown. Follow this process:

### Writing
Draft each section in plain text/markdown internally as you go. Do not stream the full draft to chat — keep it in working memory or bash scratch if needed.

### Rendering
Once all sections are written, render the full report into a single `.html` file saved to `/mnt/user-data/outputs/[TICKER]_equity_research_[YYYYMMDD].html`. Then call `present_files` with that path. Do not output the full report text to chat — a one-paragraph summary and the file link is sufficient.

### HTML structure and style
The file must be self-contained (no external CSS/font/JS dependencies beyond Google Fonts via `<link>`). Apply these structural and aesthetic guidelines:

- **Tone**: Institutional / editorial. Clean, high-contrast, data-dense. Not decorative.
- **Fonts**: Use a pairing from Google Fonts — a geometric or transitional serif for headings (e.g., DM Serif Display, Playfair Display, Libre Baskerville) and a legible sans-serif for body (e.g., DM Sans, IBM Plex Sans, Source Sans 3). Never use Arial, Roboto, or Inter.
- **Color**: Dark navy or near-black for primary text (`#0f172a` or similar). One strong accent color for headings, verdict callout, and section anchors (e.g., deep teal `#0f766e`, slate blue `#3b4f8c`, or amber `#b45309`). Light neutral background (`#f8fafc` or white). Use CSS variables for all color tokens.
- **Layout**:
  - Fixed left sidebar (~220px) with a table of contents linking to section anchors (`id="s1"` … `id="s14"`, `id="verdict"`, `id="skill-feedback"`).
  - Main content area with max-width ~780px, generous line-height (1.7).
  - Sticky header bar showing ticker, price, date, and verdict badge.
- **Header block** (top of main content): Ticker, company name, exchange, analysis date, price, market cap, key multiples (PE, PB, EV/EBITDA), consensus TP range. Laid out as a data grid, not prose.
- **TL;DR block**: Visually distinct panel (light accent background, left border in accent color) placed immediately below the header block, before Section 1.
- **Tables**: Styled with alternating row shading, sticky header row, borders on header only. Used for Comps (Section 10), Segments (Section 5), and forward projections (Section 11).
- **Verdict block**: Full-width callout at the end with Buy / Hold / Avoid badge (color-coded: green / amber / red), the one-paragraph rationale, and a 3-row risk/reward table (upside price / current / downside price with % moves).
- **Skill Feedback**: Rendered as a collapsible `<details>` element at the very bottom, outside the main report flow. Heading: "🔧 Skill Feedback (internal)".
- **Section headers**: `h2` with a left border in the accent color and a small section number prefix. Each `h2` has an `id` matching the sidebar TOC anchor.
- **Bullet points**: `ul` with tight spacing. Nested lists permitted but maximum 2 levels deep.
- **Flags / callouts** (🟢 🟡 🔴 in Red Flags): Render as colored inline badges, not emoji.
- **No animation**: Static report. No scroll effects, no hover transitions. The file is for reading, not demonstrating.

### Content rules (unchanged from markdown)
- Lead with TL;DR (2–3 lines: overall impression, key risk, whether stock looks interesting).
- All 14 sections for a full report; only requested sections for targeted dives.
- If data was unavailable for a section, note it clearly rather than fabricating.
- Verdict at the end with explicit Buy / Hold / Avoid and a risk/reward table.
- Skill Feedback at the very bottom inside `<details>`.

---

## Market-Specific Notes

**Vietnamese stocks (HOSE/HNX)**:
- Report financials in VND; convert to USD at ~25,000 VND/USD (update if the current rate differs materially) for comps and market cap comparisons.
- Reference relevant regulatory context: State ownership %, FDI foreign room remaining, related-party transaction disclosure requirements.
- Note sector-specific dynamics: banking (Basel II/III compliance, NPL ratios), real estate (land bank, legal risk), energy (FiT, DPPA, QHĐ8), oil & gas services (block FIDs, upstream capex cycle).
- **Sector metrics caveat**: `include_sector_metrics` on `get_stock_snapshot` returns meaningful structured data for Banking, Steel, and Power sectors only. For all other sectors, treat the field as informational at best — do not expect it to substitute for sector-specific web research.
- **SOE governance note**: For state-controlled enterprises (state ownership >50%), adjust Section 8 (Management) framing. Individual insider equity is typically negligible; focus on capital allocation outcomes, ROE/ROIC trends, and parent-group strategic direction rather than founder narrative. Flag related-party revenue concentration and non-economic optimization mandates explicitly. Where relevant, probe AGM materials from the parent group for subsidiary-level strategic guidance.
- **Management guidance sandbag check**: Vietnamese SOEs routinely guide well below consensus. In Section 6, compare management guidance to the quarterly run-rate and consensus. Flag the gap if guidance is >20% below consensus.
- **Non-core segments**: Even within commodity producers or single-product companies, probe for sidebar businesses (toll roads, real estate stakes, financial investments). These are often small by revenue but disproportionately high-margin or high-risk, and deserve explicit treatment in Section 5 rather than being rolled into an "other" line.
- Primary data sources: FiinData MCP (Tier 1 per Data Gathering Protocol above); supplementary web sources: CafeF, Vietstock, theinvestor.vn, HOSE/HNX exchange disclosures, SSI Research, VNDirect.

**US stocks (NYSE/NASDAQ)**:
- Reference SEC filings (10-K, 10-Q, 8-K), earnings call transcripts, and analyst consensus.
- Note any relevant regulatory or antitrust exposure.

**Global stocks**:
- Adapt to local regulatory filings and accounting standards (IFRS vs. GAAP where relevant).
- Convert financials to USD for comparability.
