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
3. **Gather data**: Use `web_search` to fetch current, real data before writing any section. Do NOT rely on training data alone for financials, prices, or recent events.
4. **Write each section** using the exact prompts below as your analytical framework.
5. **Format the output** cleanly using markdown headers per section.

---

## Data Gathering Protocol

Before writing, search for:
- Company overview and business model (1–2 searches)
- Latest financials: revenue, EBITDA, EPS, margins (1–2 searches)
- Most recent earnings release and earnings call transcript or summary
- Analyst consensus estimates and price targets
- Recent stock price history and major catalysts
- Key competitors and their valuation multiples
- Management team background and insider ownership
- Any recent red flags: accounting changes, regulatory issues, related-party transactions

For **Vietnamese stocks** (HOSE/HNX): search CafeF, VNDirect, SSI Research, HOSE disclosures, and FiinGroup in addition to general sources.
For **US stocks**: search SEC filings (10-K, 10-Q), earnings call transcripts (Seeking Alpha, Motley Fool), Bloomberg, and FactSet consensus.
For **global stocks**: adapt sources to local exchange/regulator filings.

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

---

### 6. Earnings Result

Analyze the company's most recent quarterly/annual earnings:
- **Revenue & profit vs. expectations**: Did the company beat or miss consensus? By how much?
- **Segment drivers**: Which business lines drove the result? Any notable acceleration or deceleration?
- **Margin trends**: What happened to gross/operating margins and why?
- **Guidance & outlook**: What did management guide for next quarter/full year? Any change in tone?
- **Balance sheet flags**: Anything notable in cash flow, inventory, receivables, or debt?
- **Market reaction**: How did the stock react, and what does that signal about what was priced in?
- Flag anything unusual relative to the company's recent history.

---

### 7. Earnings Calls

Summarize the company's recent earnings calls (last 2–4 quarters):
- What themes is management focused on?
- Perform sentiment analysis: how has management tone shifted over time?
- Flag any notable changes in language around guidance, risk, or capital allocation.

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

---

### 12. Red Flags

*Act as a forensic equity analyst. Identify red flags and accounting risks.*

Review across three statements:
- **Income statement**: Unusual revenue recognition, aggressive segment reporting, non-recurring items treated as recurring.
- **Balance sheet**: Goodwill/intangibles creep, related-party balances, off-balance-sheet exposure, lease obligations.
- **Cash flow**: Divergence between net income and operating cash flow, working capital manipulation, capex classification.
- **Other**: Stock-based comp as a percentage of earnings, contingent liabilities, auditor changes or qualifications.

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

- Use clear `##` section headers matching the section names above.
- Lead with a 2–3 line **TL;DR** before the first section (overall impression, key risk, and whether the stock looks interesting at current levels).
- Use tables for Comps and Segments sections.
- Use bullet points within sections; keep prose tight.
- At the end of a full report, include a **Verdict** section: summarize whether the stock is a Buy / Hold / Avoid at current valuation, with a one-paragraph rationale.
- If data was unavailable for a section (e.g., no earnings call transcripts found), note it clearly rather than fabricating.

---

## Market-Specific Notes

**Vietnamese stocks (HOSE/HNX)**:
- Report financials in VND; convert to USD for comps if needed.
- Reference relevant regulatory context: State ownership, FDI limits, room to buy.
- Note sector-specific dynamics: banking (Basel II/III compliance, NPL ratios), real estate (land bank, legal risk), energy (FiT, DPPA, QHĐ8).
- Sources: CafeF, VNDirect, SSI Research, HOSE/HNX disclosures, FiinGroup, Vietstock.

**US stocks (NYSE/NASDAQ)**:
- Reference SEC filings (10-K, 10-Q, 8-K), earnings call transcripts, and analyst consensus.
- Note any relevant regulatory or antitrust exposure.

**Global stocks**:
- Adapt to local regulatory filings and accounting standards (IFRS vs. GAAP where relevant).
- Convert financials to USD for comparability.
