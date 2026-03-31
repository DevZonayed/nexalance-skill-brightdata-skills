# PLAN: `competitive-intel` Skill for OpenClaw / ClawHub

## Vision

A Claude Code skill that turns Bright Data's scraping infrastructure into **real-time competitive intelligence** — replacing $15K–$50K/year enterprise CI tools (Crayon, Klue, AlphaSense) with an AI-native, zero-setup alternative that costs pennies per analysis.

**Core thesis**: AI without live web data is fundamentally broken for CI (stale training data, hallucinations). Live web data without AI is just raw HTML. The combination — Bright Data for real-time extraction + Claude for strategic analysis — is the missing middle tier.

---

## Market Context

| Segment | Examples | Cost | Fatal Flaw |
|---------|----------|------|------------|
| Enterprise CI | Crayon, Klue | $15K–$50K/yr | 3-8 week setup, requires dedicated CI team, 68% of battlecards never used |
| SEO/SEM tools | Semrush, Ahrefs, SimilarWeb | $139–$500/mo | SEO-only, no product/pricing/hiring intelligence |
| DIY monitoring | Google Alerts, Visualping | Free–$50/mo | Shallow, manual, misses signals |
| AI prompts | ChatGPT/Claude raw | ~Free | Stale data, hallucinations, no live web access |
| **This skill** | competitive-intel | **~$0.05–$0.50/analysis** | **Fills the gap** |

**Target users**: Startup founders, product managers, growth/marketing teams, sales teams, indie hackers — anyone who needs real CI but can't afford enterprise tools or dedicate a person to it.

---

## Skill Architecture

### Directory Structure

```
competitive-intel/
├── SKILL.md                           # Main instructions (<500 lines)
├── references/
│   ├── analysis-frameworks.md         # SWOT, Porter's Five Forces, positioning maps
│   ├── data-source-guide.md           # Which bdata command to use for each CI need
│   ├── output-templates.md            # Report templates for each analysis type
│   └── industry-signals.md            # How to interpret hiring, pricing, feature signals
├── evals/
│   └── evals.json                     # Test cases for skill evaluation
└── PLAN.md                            # This file
```

### No Custom Scripts — CLI-First Approach

This skill uses the **Bright Data CLI (`bdata`)** directly for all data collection. No custom bash scripts needed. The CLI is frictionless:

- **One-time auth**: `bdata login` — handles OAuth, API keys, zone creation automatically
- **Three core commands** cover every CI data need:
  - `bdata search "<query>"` — Google/Bing/Yandex SERP results as structured JSON
  - `bdata scrape <url>` — Any webpage as clean markdown (with bot bypass, CAPTCHA solving, JS rendering)
  - `bdata pipelines <type> <url>` — Structured data from 40+ platforms (LinkedIn, Crunchbase, Amazon, G2, etc.)
- **Pipe-friendly**: JSON output + `jq` for chaining commands
- **Async support**: `--async` + `bdata status --wait` for heavy jobs
- **Zero config**: No env vars to set, no zones to manage, no API keys to pass after login

The SKILL.md instructs Claude to call `bdata` commands directly via Bash — the skill's value is in the **analytical intelligence layer** (what to scrape, how to interpret, how to synthesize), not in wrapper scripts.

### Why CLI over scripts

| Approach | Drawback |
|----------|----------|
| Custom bash scripts | Require `BRIGHTDATA_API_KEY` + `BRIGHTDATA_UNLOCKER_ZONE` env vars, manual zone setup, curl + jq boilerplate |
| `bdata` CLI | One-time `bdata login`, auto-created zones, built-in polling, structured output — zero friction |

The CLI absorbs all the complexity. The skill just needs to tell Claude **which commands to run and what to do with the results**.

---

## SKILL.md Design

### Frontmatter

```yaml
---
name: competitive-intel
description: >
  Real-time competitive intelligence and market research using Bright Data's
  web scraping infrastructure. Analyzes competitors' pricing, features, reviews,
  hiring patterns, content strategy, and market positioning with live web data.
  Use this skill when the user wants to analyze competitors, compare products,
  monitor pricing changes, track market trends, research a market landscape,
  build competitive battlecards, find positioning opportunities, or conduct
  any form of competitive or market research. Also use when the user mentions
  competitor analysis, market intelligence, competitive landscape, win/loss
  analysis, or wants to understand what competitors are doing.
---
```

**Key**: Description is intentionally "pushy" per skill-creator best practices — lists many trigger contexts to combat undertriggering.

### Body Structure (Target: ~400 lines)

The SKILL.md body should follow this outline:

```
# Competitive Intelligence

## What This Skill Does
Brief overview — live web data + AI analysis = real-time CI.

## Prerequisites
- Bright Data CLI installed (`curl -fsSL https://cli.brightdata.com/install.sh | bash`)
- One-time login completed (`bdata login`)
- That's it. No env vars, no zone config, no API keys to manage.

## Analysis Modules

### 1. Competitor Snapshot (Quick Intel)
When to use, what it does.
Data gathering:
  - `bdata search "[competitor name]" --json` → discover website, news
  - `bdata scrape [competitor-url]` → homepage messaging & positioning
  - `bdata scrape [competitor-url]/pricing` → pricing tiers
  - `bdata scrape [competitor-url]/about` → team, mission, history
  - `bdata pipelines crunchbase_company "[crunchbase-url]"` → funding, size
  - `bdata pipelines linkedin_company_profile "[linkedin-url]"` → employees, growth
Output: Structured competitor profile.

### 2. Pricing Intelligence
When to use, what it does.
Data gathering:
  - `bdata scrape [url]/pricing` for each competitor
  - `bdata search "[competitor] pricing" --json` for third-party breakdowns
  - `bdata pipelines amazon_product "[url]"` for e-commerce products
Output: Pricing comparison table + analysis.

### 3. Review Intelligence
When to use, what it does.
Data gathering:
  - `bdata scrape [g2-url]` → G2/Capterra review pages
  - `bdata pipelines google_maps_reviews "[url]" [days]` → Google reviews
  - `bdata pipelines google_play_store "[url]"` → app store reviews
  - `bdata pipelines apple_app_store "[url]"` → iOS reviews
  - `bdata pipelines amazon_product_reviews "[url]"` → Amazon reviews
Output: Sentiment analysis, pain point extraction, gap identification.

### 4. Hiring Signal Analysis
When to use, what it does.
Data gathering:
  - `bdata pipelines linkedin_job_listings "[company-linkedin-url]"` → open roles
  - `bdata search "[competitor] careers" --json` → find careers page
  - `bdata scrape [careers-url]` → scrape careers page directly
Output: Strategic direction inference from hiring patterns.

### 5. Content & SEO Battle
When to use, what it does.
Data gathering:
  - `bdata search "[keyword]" --json` → SERP rankings
  - `bdata search "site:[competitor.com]" --json` → indexed page count
  - `bdata scrape [competitor-url]/blog` → blog index
  - `bdata scrape [article-url]` → top competitor articles
Output: Content gap analysis, keyword positioning.

### 6. Market Landscape Map
When to use, what it does.
Data gathering:
  - `bdata search "[industry] companies" --json`
  - `bdata search "best [product category]" --json`
  - `bdata search "[category] alternatives" --json`
  - `bdata scrape [g2-category-url]` → player lists
  - `bdata scrape [each-competitor-url]` → positioning
  - `bdata pipelines crunchbase_company "[url]"` → funding/size per player
Output: Categorized market map with positioning.

## Workflow: How to Run an Analysis

Step-by-step checklist Claude follows:
1. Clarify scope (which competitors, which modules)
2. Gather data (run multiple bdata commands, parallelize where possible)
3. Analyze with appropriate framework (→ reference: analysis-frameworks.md)
4. Format output (→ reference: output-templates.md)
5. Highlight actionable insights — always end with "So what should you do?"

## Output Format
Standard report structure with sections.

## Important Guidelines
- Always scrape live data, never rely on training knowledge for competitor specifics
- Cite sources with URLs for every data point
- Flag when data might be incomplete (e.g., gated pages)
- Be cost-efficient — a snapshot should use 3-5 bdata calls, not 50
- Use `--json` flag when piping bdata output for structured processing
- Use `bdata pipelines` (structured data) over `bdata scrape` (raw markdown) whenever a pipeline exists for the target platform
```

---

## The 6 Analysis Modules — Detailed Specification

### Module 1: Competitor Snapshot

**Purpose**: Quick, comprehensive profile of a single competitor.

**Data gathering** (via `bdata` CLI):
1. `bdata search "[competitor name]" --json` → top results, news, website URL
2. `bdata scrape [url]` → homepage positioning, messaging, tagline
3. `bdata scrape [url]/about` → team size, mission, history (try /about, /about-us, /company)
4. `bdata scrape [url]/pricing` → plans, tiers, pricing model
5. `bdata pipelines crunchbase_company "[crunchbase-url]"` → funding, investors, employee count, founded date
6. `bdata pipelines linkedin_company_profile "[linkedin-url]"` → employee count, growth, locations

**Claude analysis layer**:
- Synthesize into structured profile
- Identify positioning (who they target, what they emphasize)
- Note strengths and potential weaknesses
- Compare to user's product if context available

**Output template**:
```markdown
# Competitor Snapshot: [Name]

## Overview
- **Founded**: [year] | **HQ**: [location] | **Employees**: [count]
- **Funding**: [total raised, last round, investors]
- **Positioning**: [one-line summary of how they position]

## Product & Pricing
- [Plan tiers, pricing model, key features per tier]

## Messaging Analysis
- **Tagline**: [their tagline]
- **Key claims**: [what they emphasize]
- **Target audience**: [who they're speaking to]

## Strengths & Vulnerabilities
- Strengths: [...]
- Potential gaps: [...]

## Sources
- [URLs of all scraped pages]
```

### Module 2: Pricing Intelligence

**Purpose**: Compare pricing across competitors, detect pricing model patterns.

**Data gathering** (via `bdata` CLI):
1. `bdata scrape [url]/pricing` for each competitor → pricing page as markdown
2. If e-commerce: `bdata pipelines amazon_product "[url]"` or `bdata pipelines walmart_product "[url]"`
3. `bdata search "[competitor] pricing" --json` → third-party breakdowns

**Claude analysis layer**:
- Extract plan names, prices, feature lists, limits
- Normalize into comparison table
- Identify pricing model (per-seat, usage-based, freemium, enterprise-only)
- Flag positioning signals (who's cheapest, who's premium, free tier strategy)
- Recommend pricing positioning opportunities

**Output template**:
```markdown
# Pricing Intelligence Report

## Comparison Matrix
| Feature | Competitor A | Competitor B | Competitor C | Your Product |
|---------|-------------|-------------|-------------|-------------|
| Starter Plan | $X/mo | $Y/mo | Free | ? |
| ...

## Pricing Model Analysis
- [Each competitor's model type and strategy]

## Key Insights
- [Who's cheapest at each tier]
- [Feature differentiation at each tier]
- [Positioning opportunities]

## Sources
- [URLs]
```

### Module 3: Review Intelligence

**Purpose**: Mine customer reviews to find competitor pain points, loved features, and unaddressed gaps.

**Data gathering** (via `bdata` CLI):
1. `bdata pipelines google_maps_reviews "[url]" [days]` → local/service business reviews
2. `bdata pipelines google_play_store "[url]"` or `bdata pipelines apple_app_store "[url]"` → app store reviews
3. `bdata scrape [g2-url]` or `bdata scrape [capterra-url]` → G2/Capterra/Trustpilot review pages
4. `bdata pipelines amazon_product_reviews "[url]"` → Amazon product reviews

**Claude analysis layer**:
- Sentiment distribution (positive/negative/neutral percentages)
- Top praised features (what customers love)
- Top complaints (what they hate — YOUR opportunity)
- Feature requests mentioned in reviews
- Comparison mentions ("switched from X", "better than Y")
- NPS/satisfaction trends if data allows

**Output template**:
```markdown
# Review Intelligence: [Competitor Name]

## Sentiment Overview
- **Overall**: [X/5 stars, N reviews analyzed]
- **Positive**: X% | **Neutral**: X% | **Negative**: X%

## What Customers Love
1. [Feature/aspect] — [sample quote]
2. ...

## What Customers Hate (Your Opportunity)
1. [Pain point] — [sample quote]
2. ...

## Feature Requests
1. [Requested feature] — mentioned N times

## Competitive Mentions
- "[Competitor] is better than [Other] because..."

## Sources
- [URLs, review counts per source]
```

### Module 4: Hiring Signal Analysis

**Purpose**: Infer strategic direction from job postings (new markets, tech bets, scaling).

**Data gathering** (via `bdata` CLI):
1. `bdata pipelines linkedin_job_listings "[company-linkedin-url]"` → structured job posting data
2. `bdata search "[competitor] careers" --json` → find careers page URL
3. `bdata scrape [careers-url]` → scrape careers page directly if LinkedIn data insufficient

**Claude analysis layer**:
- Job posting volume and velocity (are they scaling?)
- Role distribution (engineering-heavy? sales-heavy? expanding into new functions?)
- Technology signals (what tech stacks appear in job descriptions?)
- Geographic signals (new offices = new markets?)
- Seniority distribution (hiring leaders vs. ICs)
- Specific keyword analysis (AI, machine learning, enterprise, etc.)

**Output template**:
```markdown
# Hiring Signal Analysis: [Competitor Name]

## Hiring Velocity
- **Open roles**: [N] (as of [date])
- **Signal**: [Growing fast / Stable / Contracting]

## Department Breakdown
| Department | Open Roles | Signal |
|-----------|-----------|--------|
| Engineering | N | [Investing in product] |
| Sales | N | [Scaling revenue] |
| Marketing | N | [Brand push] |
| ...

## Strategic Signals
- **Tech bets**: [Technologies appearing in job descriptions]
- **New markets**: [Geographic expansion signals]
- **Product direction**: [What new roles suggest about roadmap]

## Sources
- [URLs]
```

### Module 5: Content & SEO Battle

**Purpose**: Understand competitors' content strategy and search positioning.

**Data gathering** (via `bdata` CLI):
1. `bdata search "[competitor topic keyword]" --json` → SERP rankings
2. `bdata scrape [competitor-url]/blog` → blog index page
3. `bdata scrape [article-url]` → top-performing competitor articles
4. `bdata search "site:[competitor.com]" --json` → indexed page count

**Claude analysis layer**:
- Content volume and frequency
- Topic clusters (what themes do they write about?)
- Top-ranking content (what are they winning on in search?)
- Content gaps (topics they don't cover = your opportunity)
- Content quality assessment (depth, originality, format)

**Output template**:
```markdown
# Content & SEO Battle Report

## Content Volume
| Competitor | Blog Posts (est.) | Publishing Frequency |
|-----------|------------------|---------------------|
| A | ~N | [Weekly/Monthly] |

## Topic Analysis
| Topic Cluster | Competitor A | Competitor B | You |
|-------------|-------------|-------------|-----|
| [Topic] | 5 articles | 2 articles | 0 |

## SERP Dominance
| Keyword | #1 | #2 | #3 |
|---------|----|----|-----|
| [keyword] | [who ranks] | ... | ... |

## Content Gaps (Opportunities)
1. [Topic nobody covers well]

## Sources
- [URLs]
```

### Module 6: Market Landscape Map

**Purpose**: Discover and categorize all players in a market space.

**Data gathering** (via `bdata` CLI):
1. `bdata search "[industry] companies" --json`, `bdata search "[industry] alternatives" --json`, `bdata search "best [product category]" --json`
2. `bdata scrape [g2-category-url]` → G2/Capterra category pages for player lists
3. `bdata scrape [competitor-url]` for each discovered competitor → homepage positioning
4. `bdata pipelines crunchbase_company "[url]"` → funding/size data per player

**Claude analysis layer**:
- Categorize players by tier (enterprise, mid-market, SMB, open-source)
- Map positioning (feature-rich vs. simple, expensive vs. cheap)
- Identify white space (underserved segments)
- Note recent entrants and trends

**Output template**:
```markdown
# Market Landscape: [Category]

## Market Map
### Enterprise Tier
- [Company] — [one-line positioning] — [funding, size]

### Mid-Market
- ...

### SMB / Startups
- ...

### Open Source / Free
- ...

## Positioning Map
[2x2 matrix description: e.g., Price vs. Feature breadth]

## White Space Opportunities
1. [Underserved segment or positioning]

## Market Trends
- [Emerging patterns, consolidation, new entrants]

## Sources
- [URLs]
```

---

## Reference Files — Detailed Specification

### `references/analysis-frameworks.md`

Provide Claude with structured analytical frameworks to apply:

- **SWOT Analysis** — template + when to use
- **Porter's Five Forces** — template + when to apply to a market
- **Positioning Matrix** — how to build 2x2 competitive positioning maps
- **Jobs-to-be-Done** — analyzing competitors through customer jobs lens
- **Blue Ocean Strategy** — identifying uncontested market space
- **Win/Loss Framework** — structuring competitive win/loss analysis

Each framework should include:
- When to use it (which module triggers it)
- Step-by-step application instructions
- Output template
- Example interpretation

### `references/data-source-guide.md`

Maps every CI need to the specific `bdata` CLI command:

```markdown
## Data Source Selection Guide

| Intelligence Need | Primary Command | Fallback |
|------------------|----------------|----------|
| Company overview | `bdata pipelines crunchbase_company "[url]"` | `bdata scrape [url]/about` |
| Pricing | `bdata scrape [url]/pricing` | `bdata search "[company] pricing" --json` |
| Product reviews (SaaS) | `bdata scrape [g2-url]` | `bdata pipelines google_maps_reviews "[url]"` |
| Product reviews (e-commerce) | `bdata pipelines amazon_product_reviews "[url]"` | `bdata pipelines google_play_store "[url]"` |
| Employee count | `bdata pipelines linkedin_company_profile "[url]"` | `bdata pipelines crunchbase_company "[url]"` |
| Job postings | `bdata pipelines linkedin_job_listings "[url]"` | `bdata scrape [careers-url]` |
| Social presence | `bdata pipelines [platform]_profiles "[url]"` | — |
| News & PR | `bdata search "[company] news" --json` | `bdata pipelines reuter_news "[url]"` |
| Financial data | `bdata pipelines yahoo_finance_business "[url]"` | `bdata pipelines crunchbase_company "[url]"` |
| App metrics | `bdata pipelines google_play_store "[url]"` | `bdata pipelines apple_app_store "[url]"` |
| Content/blog | `bdata scrape [url]/blog` | `bdata search "site:[domain]" --json` |
| Competitor discovery | `bdata search "[category] companies" --json` | `bdata scrape [g2-category-url]` |
```

**Command selection rule**: Always prefer `bdata pipelines` (structured data) over `bdata scrape` (raw markdown) when a pipeline type exists for the target platform. Pipelines return clean JSON; scraping returns markdown that Claude must parse.

Also include:
- Rate limit guidance (how many `bdata` calls per analysis type)
- Cost estimates per module
- Error handling (what to do when a page is gated, returns empty, or times out)
- Chaining patterns (e.g., `bdata search → jq → bdata scrape` pipelines)

### `references/output-templates.md`

Complete markdown templates for every report type. Claude should use these as the base and customize per analysis. Include:

- Full Competitive Battlecard template
- Quick Snapshot template
- Detailed Pricing Comparison template
- Review Intelligence Report template
- Hiring Signal Report template
- Content Battle Report template
- Market Landscape Map template
- Executive Summary template (for multi-module analyses)

### `references/industry-signals.md`

Guide for interpreting raw data as strategic signals:

- **Pricing signals**: What price increases/decreases mean, freemium vs. enterprise-only implications
- **Hiring signals**: What role distribution reveals about strategy, tech stack choices as product bets
- **Content signals**: Topic shifts as market pivots, content volume as growth stage indicator
- **Review signals**: Negative review clusters as product debt, feature requests as roadmap leaks
- **Funding signals**: Round size and investor type as strategy indicators
- **Social signals**: Engagement trends, audience growth rates, content strategy shifts

---

## Data Gathering — CLI Command Patterns

No custom scripts are needed. Claude runs `bdata` commands directly. Below are the exact command patterns for each module.

### Module 1: Competitor Snapshot — Command Sequence

```bash
# Step 1: Discover competitor's website and news
bdata search "[competitor name]" --json

# Step 2: Scrape key pages (run these in parallel via multiple Bash tool calls)
bdata scrape [competitor-url]                    # Homepage
bdata scrape [competitor-url]/pricing            # Pricing page
bdata scrape [competitor-url]/about              # About page (try /about, /about-us, /company)

# Step 3: Structured data (if URLs known)
bdata pipelines crunchbase_company "[crunchbase-url]"        # Funding, size
bdata pipelines linkedin_company_profile "[linkedin-url]"    # Employees, growth
```

### Module 2: Pricing Intelligence — Command Sequence

```bash
# Scrape pricing pages for each competitor
bdata scrape [competitor-a]/pricing
bdata scrape [competitor-b]/pricing
bdata scrape [competitor-c]/pricing

# For e-commerce products
bdata pipelines amazon_product "[url]"

# Supplementary: third-party pricing breakdowns
bdata search "[competitor] pricing review" --json
```

### Module 3: Review Intelligence — Command Sequence

```bash
# Find review pages
bdata search "[competitor] site:g2.com" --json | jq -r '.organic[0].link'
bdata search "[competitor] site:capterra.com" --json | jq -r '.organic[0].link'

# Scrape review pages
bdata scrape [g2-url]
bdata scrape [capterra-url]

# Structured review data (when URL is available)
bdata pipelines google_maps_reviews "[google-maps-url]" 30
bdata pipelines amazon_product_reviews "[amazon-url]"
bdata pipelines google_play_store "[play-store-url]"
bdata pipelines apple_app_store "[app-store-url]"
```

### Module 4: Hiring Signals — Command Sequence

```bash
# Find LinkedIn company page
bdata search "[competitor] linkedin company" --json | jq -r '.organic[0].link'

# Get job listings
bdata pipelines linkedin_job_listings "[linkedin-company-url]"

# Fallback: scrape careers page directly
bdata search "[competitor] careers" --json | jq -r '.organic[0].link'
bdata scrape [careers-url]
```

### Module 5: Content & SEO Battle — Command Sequence

```bash
# Check SERP rankings for target keywords
bdata search "[keyword 1]" --json
bdata search "[keyword 2]" --json

# Estimate indexed pages
bdata search "site:[competitor.com]" --json

# Scrape blog content
bdata scrape [competitor-url]/blog
bdata scrape [top-article-url]
```

### Module 6: Market Landscape — Command Sequence

```bash
# Discover players
bdata search "[industry] companies" --json
bdata search "best [product category] tools" --json
bdata search "[product category] alternatives" --json

# Scrape category pages
bdata scrape [g2-category-url]

# Quick snapshot of each discovered competitor
bdata scrape [competitor-1-url]
bdata scrape [competitor-2-url]
# ... for each discovered player

# Enrich with funding/size data
bdata pipelines crunchbase_company "[url]"   # for key players
```

### Parallelization Strategy

Claude should maximize parallel `bdata` calls using multiple Bash tool calls in a single response:
- **Independent scrapes** (e.g., 3 competitor pricing pages) → run all in parallel
- **Sequential dependencies** (e.g., search → extract URL → scrape) → run sequentially
- **Heavy pipelines** (LinkedIn, Crunchbase) → use `--async` flag for jobs that may take >30s, then check with `bdata status <job-id> --wait`

---

## Evaluation Plan

### Test Cases (`evals/evals.json`)

```json
{
  "skill_name": "competitive-intel",
  "evals": [
    {
      "id": 1,
      "prompt": "Analyze Notion as a competitor. I'm building a project management tool for small teams.",
      "expected_output": "Comprehensive competitor snapshot including Notion's pricing, positioning, team size, and strategic recommendations",
      "expectations": [
        "The output includes Notion's current pricing tiers with actual dollar amounts",
        "The output identifies Notion's target audience and positioning",
        "The output includes data from at least 2 different sources (website + Crunchbase/LinkedIn)",
        "The output provides actionable positioning recommendations",
        "All data points are sourced from live scraping, not training knowledge"
      ]
    },
    {
      "id": 2,
      "prompt": "Compare pricing between Slack, Microsoft Teams, and Discord for business communication.",
      "expected_output": "Pricing comparison table with plan tiers, features, and strategic analysis",
      "expectations": [
        "The output includes a comparison table with all three competitors",
        "Pricing data includes specific dollar amounts per plan tier",
        "The output identifies each competitor's pricing model (per-seat, freemium, etc.)",
        "The output highlights pricing positioning opportunities"
      ]
    },
    {
      "id": 3,
      "prompt": "What are people saying about Figma vs Canva? I want to understand which pain points I could exploit.",
      "expected_output": "Review intelligence report with sentiment analysis and pain point extraction",
      "expectations": [
        "Reviews are sourced from at least one review platform (G2, Capterra, app stores)",
        "Pain points are categorized and ranked by frequency/severity",
        "The output includes actual user quotes as evidence",
        "The output identifies exploitable gaps in both products"
      ]
    },
    {
      "id": 4,
      "prompt": "What can we learn from Stripe's hiring patterns? Are they expanding into new areas?",
      "expected_output": "Hiring signal analysis with strategic interpretation",
      "expectations": [
        "The output includes current open roles sourced from LinkedIn or careers page",
        "Roles are categorized by department",
        "The output interprets hiring patterns as strategic signals",
        "Geographic expansion signals are identified if present"
      ]
    },
    {
      "id": 5,
      "prompt": "Map out the competitive landscape for API documentation tools.",
      "expected_output": "Market landscape map with categorized players and positioning analysis",
      "expectations": [
        "At least 5 competitors are identified and categorized",
        "Players are organized by tier (enterprise, mid-market, SMB, open-source)",
        "The output identifies white space or underserved segments",
        "Each competitor has a one-line positioning summary sourced from their website"
      ]
    }
  ]
}
```

### Evaluation Process

Follow the skill-creator workflow:

1. Write SKILL.md draft → run test cases with and without skill
2. Grade results using grader agent
3. Aggregate into benchmark.json
4. Review with eval viewer
5. Iterate based on failures
6. Run description optimization (trigger eval with should/shouldn't trigger queries)
7. Package final version

### Trigger Evaluation Queries

**Should trigger** (8-10):
- "Analyze my top 3 competitors"
- "What's the competitive landscape for [category]?"
- "Compare pricing between X, Y, and Z"
- "What are people saying about [competitor] in reviews?"
- "What can we learn from [company]'s job postings?"
- "Help me build a competitive battlecard for [competitor]"
- "Research the market for [product category]"
- "What's [competitor]'s strategy based on their recent moves?"
- "I need to understand my competition before our board meeting"

**Should NOT trigger** (8-10):
- "Scrape this URL for me" (use scrape skill)
- "Search Google for [topic]" (use search skill)
- "Get LinkedIn profile for [person]" (use data-feeds)
- "Build me a web scraper for Amazon" (use scraper-builder)
- "What's the best JavaScript framework?" (general question, not CI)
- "Help me write a pricing page" (content creation, not analysis)
- "Monitor this URL for changes" (generic monitoring, not CI)
- "Analyze my website's SEO" (self-analysis, not competitive)

---

## Implementation Order

### Phase 1: Foundation (MVP)

Build the core that makes the skill functional:

1. **SKILL.md** — Write the main skill file with all 6 modules, CLI command patterns, and workflows
2. **references/data-source-guide.md** — Map CI needs to `bdata` commands with fallbacks
3. **references/output-templates.md** — Report templates for all modules
4. **evals/evals.json** — Test cases

**Exit criteria**: Can run eval IDs 1 and 2 successfully. Claude correctly uses `bdata` CLI commands (not bash scripts or training knowledge) to gather data.

### Phase 2: Depth

Add the analytical depth that differentiates from shallow AI analysis:

5. **references/analysis-frameworks.md** — Strategic frameworks (SWOT, Porter's, positioning maps)
6. **references/industry-signals.md** — Signal interpretation guide

**Exit criteria**: Can run all 5 evals successfully. Reports include strategic analysis, not just raw data.

### Phase 3: Polish

Optimize triggering, output quality, and edge cases:

7. Run trigger evaluation and optimize description
8. Run full benchmark (with vs. without skill)
9. Iterate on SKILL.md based on eval failures
10. Handle edge cases (gated pages, missing data, CLI timeouts)

**Exit criteria**: >80% assertion pass rate across all evals, clean trigger eval results.

### Phase 4: Distribution

11. Package skill with `package_skill.py`
12. Write ClawHub listing description
13. Test installation flow end-to-end

---

## Key Design Principles

1. **Live data first** — Never let Claude answer competitive questions from training knowledge alone. Always scrape first.

2. **Progressive depth** — Quick snapshot for "who is this competitor?" → deep dive for "build me a full battlecard." Don't over-scrape for simple questions.

3. **Cite everything** — Every data point must link back to its source URL. This builds trust and lets users verify.

4. **Actionable over comprehensive** — End every report with "So what? What should you do?" — not just data dumps.

5. **Respect costs** — A snapshot should use 3-5 `bdata` calls, not 50. Guide Claude to be efficient.

6. **CLI-native** — Use `bdata` CLI directly. No custom scripts, no env vars, no wrapper layers. The CLI handles auth, zones, polling, and anti-bot automatically.

7. **Prefer pipelines over scraping** — `bdata pipelines` returns structured JSON; `bdata scrape` returns raw markdown. Always use a pipeline when one exists for the target platform.

8. **Fail gracefully** — If a pricing page is gated, say so and try alternatives. If Crunchbase has no data, skip that section. Never hallucinate to fill gaps.

9. **Framework-driven analysis** — Raw data → analytical framework → strategic insight. This is what separates a skill from just running CLI commands.

---

## Competitive Differentiation (vs. existing ClawHub skills)

| What exists | What this skill adds |
|------------|---------------------|
| `bdata search` — raw SERP results | Strategic competitor discovery + market mapping |
| `bdata scrape` — raw page content | Structured CI reports with analytical frameworks |
| `bdata pipelines` — structured data from 40+ sites | Multi-source intelligence synthesis (combine LinkedIn + Crunchbase + website + reviews into one report) |
| `brightdata-cli` skill — CLI usage guide | Knows **which** commands to run for each CI use case, in what order, and how to interpret the results strategically |
| Generic AI prompts | Live web data via `bdata` eliminates hallucinations for competitor specifics |
| Enterprise CI tools ($15K+/yr) | Same intelligence at ~$0.05–$0.50 per analysis, zero setup |

**The unique value**: No other ClawHub skill combines live multi-source web data + strategic analytical frameworks + structured actionable output for competitive intelligence. The `brightdata-cli` skill teaches Claude how to use the CLI; this skill teaches Claude how to be a **competitive intelligence analyst** using the CLI as its data layer.

---

## Success Metrics

- **Eval pass rate**: >80% across all test cases
- **Trigger accuracy**: >90% correct triggering on trigger eval set
- **User value**: Reports contain data that cannot be obtained from Claude's training knowledge alone
- **Cost efficiency**: Average analysis costs <$1 in Bright Data API calls
- **ClawHub traction**: Track installs and usage after publication
