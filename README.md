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
```

When re-uploading to Claude.ai, always delete the previous version under Customize → Skills first to avoid duplicate skill triggers.

---

## Disclaimer

This skill produces analysis for informational and educational purposes only. Output should not be construed as investment advice, a solicitation, or a recommendation to buy or sell any security. Always verify Claude's data against primary sources before acting on any analysis.

---

## License

Personal use. Not licensed for redistribution.
