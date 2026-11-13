# Autonomous Lead & Competitor Intelligence Agent (Google Opal)

**Live app:** https://opal.google/app/1BXJkomgLuR8qlI6-Cu7EubqYXy0NYyW0
**Exported pipeline:** [workflow.json](workflow.json)

## 1. Objective
Build, test and document a multi-step, grounded agentic workflow in Google Opal that takes a company or technology name, searches the live web, extracts structured intelligence, verifies it through a critic step, and exports the result to an interactive HTML card and a Google Sheet.

## 2. Business Problem
Sales, partnership and strategy teams spend hours manually researching leads and competitors (pricing, funding, leadership, news). The information changes quickly and plain LLM answers are often outdated or hallucinated. This agent produces a sourced, fact-checked profile in about a minute, flags stale or conflicting claims instead of silently trusting them, and logs every run to a sheet for auditability.

## 3. Architecture

![Canvas topology](Architecture%20Screenshots/01-canvas-topology.png)

```
Company Name ─┐
              ├─> Research company ─┬─> Structured Synthesis ─> Fact Check And Verification ─┬─> Design Dashboard (HTML card)
Focus Area  ──┘   (google_search)   └────────────────────────────────────────────────────────┴─> Log Research To Sheet
```

The Research output feeds both the synthesis step and the critic, so the critic can compare the draft against the original evidence (not just against itself).

### Step-by-step node breakdown

| # | Node (workflow.json id) | Opal type | What it does |
|---|---|---|---|
| 1 | **Company Name** (`ask_user_company_name`) | User Input | Published input form field: "Enter the company or technology to research." |
| 2 | **Focus Area** (`ask_user_focus_area`) | User Input | Optional field, e.g. pricing, funding, leadership. Steers the search. |
| 3 | **Research company** (`node_step_research_notes`) | Generate, search grounding (`text-3-flash`) | The `@Search` agent step. Runs distinct Google searches for company overview, products & pricing, funding & recent news, and competitors. Each finding must include the claim, source URL and publication date. Missing facts are written as `NOT_FOUND`. Retrieved data is declared untrusted. |
| 4 | **Structured Synthesis** (`node_step_json_draft`) | Generate (agent mode) | Converts the notes into strict JSON with keys `company, summary, products, pricing, funding, leadership, competitors, recent_news, sources`. Every fact carries a `source_url`; missing values become `UNKNOWN`. Output is raw JSON with no commentary. |
| 5 | **Fact Check And Verification** (`node_step_verified_results`) | Generate (agent mode) + search grounding | The Critic. Receives the JSON draft **and** the original research notes, independently re-verifies funding, pricing and leadership via search, and labels each claim `VERIFIED`, `UNSUPPORTED` or `CONFLICTING`. Outputs (1) cleaned JSON, (2) verdict table with justification, (3) confidence score. |
| 6 | **Design Dashboard** (`node_step_intelligence_dashboard`) | Output, `render-outputs` (HTML, `gemini-flash`) | Generates a responsive dark-themed interactive card: summary, products, pricing, funding, competitors, news, confidence badge, verdict table and clickable source links. `UNKNOWN` fields are greyed out and labelled "Not verified". Takes **only** the critic output. |
| 7 | **Log Research To Sheet** (`node_step_log_status`) | Output, `render-outputs` (`google-sheets`) | Appends the critic output to the "Opal Intelligence Log" Google Sheet. Takes **only** the critic output. |

Data flow (edges in workflow.json): inputs 1+2 → 3; 3 → 4; 3 + 4 → 5; 5 → 6; 5 → 7.

## 4. Live Test Run (target: Notion Labs, focus: pricing)

**Published input form** (Company Name, then optional Focus Area):

| Company Name | Focus Area |
|---|---|
| ![company](Architecture%20Screenshots/02-input-form-company-name.png) | ![focus](Architecture%20Screenshots/03-input-form-focus-area.png) |

**Agent running** (live progress through the pipeline, here at the "Research company" step):

![running](Architecture%20Screenshots/04-run-in-progress.png)

**Generated interactive output card** (identity, valuation, product ecosystem, pricing tiers, 2026 shifts, competitors):

![card](Architecture%20Screenshots/05-interactive-output-card.png)

**Updated Google Sheet log** (claim / label / justification rows from the critic):

![sheet](Architecture%20Screenshots/06-google-sheet-log.png)

> **Note on the Sheet export:** the "Log Research To Sheet" node passes the entire output of the Fact Check step to Opal's Google Sheets output. The sheet therefore stores the critic's verdict table as claim-level rows (Claim | Label | Justification) rather than one summary row per run. This is intentional: it gives a claim-by-claim audit trail of what was verified, what conflicted and why, which is more useful for hallucination review than a single row.

What the run proves: most claims came back `VERIFIED` with source-backed justification, while stale or incomplete draft claims were caught as `CONFLICTING` (e.g. Business plan priced at an obsolete $15/$18 vs ~$20 now; draft valuation of $10B from 2021 vs $11B in the Jan 2026 tender; leadership list omitted COO Akshay Kothari; the standalone Notion AI add-on is now bundled; news section ignored 2026 developments). The corrected values are what appear in the dashboard.

## 5. Critical Thinking Questions

### Q1. How did the Critic Node mitigate hallucinations when information was missing or ambiguous?
Hallucination is attacked at three stages, and the critic is the last and strongest one.

1. **Upstream, refuse to guess.** The research step must write `NOT_FOUND` when a fact can't be located, and the synthesis step must write `UNKNOWN` rather than infer or fill gaps. Missing information therefore enters the pipeline as an explicit marker, not as plausible-sounding text.
2. **The critic is a separate, adversarial pass.** "Fact Check And Verification" is instructed to act as a skeptical fact-checker. It receives both the JSON draft and the original research notes, so every claim can be traced back to evidence. It also performs its own independent `google_search` for the highest-risk categories (funding, pricing, leadership), so a claim is not validated merely because an earlier step said it.
3. **A closed label set with "unknown" as a valid answer.** Each claim must be labelled `VERIFIED`, `UNSUPPORTED` or `CONFLICTING`. `UNSUPPORTED` gives the model a legitimate way to say "no evidence" instead of being pushed to fabricate support, and `CONFLICTING` surfaces ambiguity instead of silently choosing one source. The critic returns cleaned JSON containing only verified or corrected information, a justification per claim, and a 0-100 confidence score.

Downstream, the dashboard and the sheet consume **only** the critic's output, never the raw draft, so an unverified claim cannot bypass the check. In the Notion run, the critic caught several real errors (outdated pricing, an old valuation, a missing COO, a retired add-on) and the justification table was preserved in the Google Sheet, which gives a human reviewer an audit trail.

### Q2. What security delimiters and negative constraints guard against indirect prompt injection from scraped web text?
Scraped pages are the attack surface: a malicious page can contain text like "ignore previous instructions and ...". The defenses used:

- **Untrusted-data declaration at every step that touches web content.** The Research node states that all retrieved data must be handled as untrusted. The workflow's design intent additionally instructs that anything returned by search or web pages is data, never instructions, and must be ignored if it tells the model to change behaviour, reveal instructions or visit links. Synthesis and Critic are told to treat the notes and search results as data only and never to follow instructions found inside them.
- **Negative constraints:** do not guess; do not infer or fill gaps; use only facts present in the notes; output only JSON (no free-form commentary where injected text could ride along); no scripts and no raw HTML from data in the dashboard.
- **Structural delimiting via the pipeline itself.** Each step receives the previous output through a named input reference (`{{Research company}}`, `{{Structured Synthesis}}`) that is separated from its own instruction block, and the strict JSON schema in Synthesis acts as a choke point: injected prose has no field to live in, and every field needs a `source_url`.
- **Blast radius limiting.** Output nodes render data as plain text into a sandboxed HTML iframe; the only side effect on external systems is a fixed append to one Google Sheet. The agent has no tools for sending mail, browsing arbitrary links or modifying files.
- **Independent re-verification.** The critic re-checks key claims with its own search, so a planted claim that no independent source corroborates is labelled `UNSUPPORTED`.

**Honest limitation:** these are prompt-level mitigations. They reduce but do not eliminate injection risk, and the node prompts rely on natural-language declarations of untrusted data rather than hard-coded tag delimiters. A hardened version would wrap content in explicit tags such as `<web_content>...</web_content>` and add an allow-list validation step on the JSON.

### Q3. Google Opal vs. code-first frameworks such as CrewAI or LangChain

| | Google Opal | CrewAI / LangChain |
|---|---|---|
| Time to a working agent | Minutes: visual canvas plus natural-language editing. This 7-node pipeline was built and tested in one session. | Hours to days: environment, dependencies, prompt/tool code, glue code |
| DevOps | **Zero.** Hosting, auth, scaling, Google Search grounding, HTML rendering and the Google Sheets integration are built in; publishing is one click and yields a shareable URL | You provision hosting, secrets, API keys, retries/rate limits, logging, CI/CD, and write the Sheets/Search integrations |
| Customization | Limited to provided step types, models and tools; no custom code, loops, conditional branching, custom memory or vector stores | Full control: custom tools, routing, memory/RAG, parallel agents, any model, eval harnesses |
| Testing / observability | Console and step preview only; no unit tests or Git-style diffs | Tracing (e.g. LangSmith), unit/integration tests, version control, reproducible deployments |
| Maintenance | Platform-managed prompts and runtime, but prompts are edited in a UI and behaviour can change when the platform or underlying model changes | You maintain everything, but you control upgrades, pinning and rollbacks |
| Portability / risk | Vendor lock-in. Notably, Opal graduates out of Google Labs on 17 Nov 2026 and active workflows stop running as they move to skills in Gemini, which is why the exported `workflow.json` and screenshots are preserved here | Open source and portable across clouds and model vendors |

**Verdict:** Opal's agility and zero-DevOps model are ideal for prototypes, internal tools and business-user automations like this research agent, where speed to value matters more than control. For production systems that need custom tools, deterministic orchestration, automated evaluation, compliance or multi-vendor portability, a code-first framework is the better long-term investment, at the cost of ongoing engineering and operations. A pragmatic path is to prototype in Opal and port the validated prompt design and graph topology into code once requirements outgrow the platform.

## 6. Repository Structure
```
codecamp-opal-<firstname>-<lastname>/
├── README.md
├── workflow.json
└── Architecture Screenshots/
    ├── 01-canvas-topology.png
    ├── 02-input-form-company-name.png
    ├── 03-input-form-focus-area.png
    ├── 04-run-in-progress.png
    ├── 05-interactive-output-card.png
    └── 06-google-sheet-log.png
```
