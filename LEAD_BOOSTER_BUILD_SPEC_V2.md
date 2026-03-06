# Lead Booster Pro — Full-Stack Build Spec v2.0
**ManageAI Internal · Confidential**
Version 2.0 · March 2026 · Claude-Native Edition

---

## WHY CLAUDE INSTEAD OF OPENAI — READ THIS FIRST

Every AI call in this system uses **Claude Sonnet** (`claude-sonnet-4-20250514`). Here is why this is the right call for a contact extraction and enrichment pipeline:

**1. Structured Output Adherence**
Claude follows complex JSON schemas more reliably than GPT-4o. In a pipeline where a malformed JSON response breaks downstream Supabase inserts, this matters enormously. Claude returns clean, parseable arrays — consistently.

**2. Instruction Density**
The extraction prompt in WF-03 has 15+ hard rules (include this, exclude that, only if explicitly present, confidence scoring, classification, subpage detection). Claude handles high-density instruction sets without degrading on rules buried at the bottom. GPT-4o drifts on long system prompts.

**3. Classification Accuracy**
The `complete / partial / subpages / no_people` classification that routes the pipeline is a judgment call requiring contextual reading of HTML. Claude's reading comprehension on semi-structured messy web content is measurably better.

**4. One API, Full Stack**
You already have the Anthropic key. Using Claude for extraction (WF-03), ICP scoring (WF-08), news intelligence (WF-09), and outreach personalization (WF-10) means one credential, one billing relationship, one model family, no OpenAI dependency anywhere in your stack.

**5. Cost**
Claude Sonnet is cheaper per token than GPT-4o at this workload. At 2,000 leads/month the savings are ~$30–40/mo.

**6. Hallucination Profile**
For contact data extraction, hallucinating a person who doesn't exist is a hard failure. Claude hallucinates less on grounded extraction tasks (tasks where the answer must come from the provided text).

---

## TABLE OF CONTENTS

1. Project Overview
2. Full System Architecture
3. Technology Stack
4. Free & Paid Data Sources (Full Waterfall)
5. Supabase Schema v2
6. Apify Actors
7. n8n Workflows 01–10
8. Supabase Edge Functions
9. Claude Prompt Library
10. Credentials & API Endpoints
11. Claude Code Swarm Instructions
12. GitHub Repo Structure
13. Smoke Test & Validation Scripts
14. Seed Data
15. Post-Swarm Checklist

---

## 1. PROJECT OVERVIEW

### What This Builds

A fully autonomous, Claude-native **contact intelligence platform** — not just a scraper. The system:

- Accepts company/domain targets via REST webhook from any caller (Marla, AI personas, CRM triggers, manual)
- Discovers company web presence via Apify multi-actor strategy
- Extracts structured people data from HTML using Claude (not GPT)
- Runs a 14-step free-first enrichment waterfall
- Scores every contact against your ICP using Claude
- Monitors news and trigger events for target companies
- Detects technographic profile (what software they run)
- Consumes intent signals (companies actively researching)
- Finds LinkedIn connection paths through your team
- Auto-generates personalized outreach drafts via Claude
- Queues verified, scored, personalized contacts for human approval
- Routes output by vertical to downstream systems

### v2.0 vs v1.0

| Capability | v1.0 | v2.0 |
|---|---|---|
| Core scrape + enrichment | ✅ | ✅ |
| Free-first waterfall | ✅ | ✅ |
| All 5 paid enrichment tools | ✅ | ✅ |
| AI model | ~~OpenAI GPT-4~~ | ✅ Claude Sonnet |
| ICP Scoring Engine | ❌ | ✅ WF-08 |
| News & Trigger Monitoring | ❌ | ✅ WF-09 |
| Automated Outreach Handoff | ❌ | ✅ WF-10 |
| Technographic Enrichment | ❌ | ✅ WF-05 step |
| Intent Data Layer | ❌ | ✅ WF-09 |
| Email Validation Waterfall (3-step) | ❌ | ✅ WF-05 |
| Company Lookalike Expansion | ❌ | ✅ WF-08 |
| LinkedIn Connection Intelligence | ❌ | ✅ WF-05 |
| Feedback Loop (win data → ICP) | ❌ | ✅ WF-08 |
| Duplicate-Across-Clients Guard | ❌ | ✅ WF-04 |
| Enrichment Cache TTL (90-day refresh) | ❌ | ✅ WF-07 |
| Vertical-Specific Claude Prompts | ❌ | ✅ WF-03 |
| Workflows total | 7 | 10 |

---

## 2. FULL SYSTEM ARCHITECTURE

```
═══════════════════════════════════════════════════════════════════
                        INTAKE LAYER
═══════════════════════════════════════════════════════════════════

  External callers:
  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │  Marla   │  │ AI Persona│  │   CRM   │  │  Manual  │
  │(Teams bot)│  │(Rebecka  │  │ Webhook │  │  POST   │
  └────┬─────┘  └────┬─────┘  └────┬────┘  └────┬─────┘
       └─────────────┴──────────────┴─────────────┘
                           │
                           ▼
          POST /functions/v1/enrich-webhook
          { company, domain, vertical, source, priority }
                           │
                           ▼
               ┌─────────────────────┐
               │  WF-01: Orchestrator│
               │  • Validate input   │
               │  • Dedup check      │
               │  • Queue in Supabase│
               │  • Route to WF-02   │
               └──────────┬──────────┘

═══════════════════════════════════════════════════════════════════
                      DISCOVERY LAYER
═══════════════════════════════════════════════════════════════════

               ┌──────────┴──────────┐
               │  WF-02: Discovery   │
               │  • Apify A1 Website │
               │  • Apify A2 LinkedIn│
               │  • Apify A3 Google  │
               │  • Aggregate pages  │
               │  → lb_pages table   │
               └──────────┬──────────┘
               ┌──────────┴──────────┐
               │  WF-09: Triggers    │  ← runs in parallel
               │  • Google News RSS  │
               │  • LinkedIn company │
               │  • Press releases   │
               │  • Intent signals   │
               │  • Bombora/G2       │
               │  → lb_triggers table│
               └──────────┬──────────┘

═══════════════════════════════════════════════════════════════════
                      EXTRACTION LAYER
═══════════════════════════════════════════════════════════════════

               ┌──────────┴──────────┐
               │  WF-03: Extract     │
               │  • HTTP fetch HTML  │
               │  • JS normalizer    │
               │  • CLAUDE: extract  │  ← vertical-tuned prompt
               │    people + classify│
               │  • Subpage recurse  │
               │  → lb_contacts_raw  │
               └──────────┬──────────┘

═══════════════════════════════════════════════════════════════════
                     ENRICHMENT LAYER
═══════════════════════════════════════════════════════════════════

               ┌──────────┴──────────┐
               │  WF-04: Enrich Orch │
               │  • Dedup fingerprint│
               │  • Cross-client guard│
               │  • Batch fan-out    │
               └──────────┬──────────┘
                           │
               ┌──────────┴──────────┐
               │  WF-05: Waterfall   │
               │  FREE TIER FIRST:   │
               │  1. GitHub emails   │
               │  2. WHOIS registrant│
               │  3. Google Places   │
               │  4. ProPublica/EDGAR│
               │  5. FullContact     │
               │  6. Clearbit        │
               │  PAID TIER:         │
               │  7. Hunter.io       │
               │  8. Apollo.io       │
               │  9. PDL             │
               │  10. Snov.io        │
               │  11. RocketReach    │
               │  VALIDATION:        │
               │  12. ZeroBounce     │
               │  13. MillionVerifier│
               │  14. NeverBounce    │
               │  AUGMENTATION:      │
               │  15. BuiltWith tech │
               │  16. Wappalyzer     │
               │  17. LinkedIn connect│
               │  → lb_contacts table│
               └──────────┬──────────┘

═══════════════════════════════════════════════════════════════════
                    INTELLIGENCE LAYER
═══════════════════════════════════════════════════════════════════

               ┌──────────┴──────────┐
               │  WF-08: ICP Scorer  │
               │  • CLAUDE: score    │
               │    0-100 vs ICP     │
               │  • Seniority weight │
               │  • Tech stack match │
               │  • Intent multiplier│
               │  • Trigger bonus    │
               │  • Lookalike expand │
               │  • Win feedback loop│
               │  → lb_contacts score│
               └──────────┬──────────┘

═══════════════════════════════════════════════════════════════════
                      OUTPUT LAYER
═══════════════════════════════════════════════════════════════════

               ┌──────────┴──────────┐
               │  WF-10: Outreach    │  80+ score only
               │  • CLAUDE: write    │
               │    personalized     │
               │    email 1 of 3     │
               │  • Instantly.ai API │
               │  • Hold for approval│
               │  → lb_outreach table│
               └──────────┬──────────┘
               ┌──────────┴──────────┐
               │  WF-06: Output Route│
               │  • Vertical switch  │
               │  • CRM push         │
               │  • Sheets fallback  │
               │  • Dave notify      │
               └──────────┬──────────┘
               ┌──────────┴──────────┐
               │  WF-07: Retry/Maint │
               │  • Stuck job retry  │
               │  • 90-day refresh   │
               │  • Error alerts     │
               └─────────────────────┘
```

---

## 3. TECHNOLOGY STACK

| Component | Technology | Version / Tier | Purpose |
|---|---|---|---|
| Workflow engine | n8n (self-hosted, Coolify) | Latest | Orchestration |
| Scraping | Apify Cloud | Starter $49/mo | Headless browser |
| Database | Supabase (Postgres) | Pro $25/mo | Queue, contacts, state |
| Edge Functions | Supabase / Deno | Included | Webhooks, intake |
| **AI — ALL calls** | **Claude Sonnet** | **claude-sonnet-4-20250514** | **Extraction, scoring, writing** |
| Enrichment 1 | Hunter.io | Starter $49/mo | Email lookup |
| Enrichment 2 | Apollo.io | Basic $49/mo | Person + company |
| Enrichment 3 | People Data Labs | Pay-as-go | Person data |
| Enrichment 4 | Snov.io | Starter $39/mo | Email finder |
| Enrichment 5 | RocketReach | Essential $80/mo | LinkedIn enrichment |
| Validation 1 | ZeroBounce | Pay-as-go | Email validation |
| Validation 2 | MillionVerifier | Pay-as-go | Validation fallback |
| Technographics | BuiltWith | Free tier | Tech stack detection |
| Intent | Bombora / G2 | Trial | Buying intent signals |
| Outreach | Instantly.ai | Growth $47/mo | Sequence automation |
| Version control | GitHub | Free | Source of truth |

**Total estimated monthly (2,000 leads):** ~$380/mo → ~$0.19/lead all-in

---

## 4. FREE & PAID DATA SOURCES — FULL WATERFALL

### Waterfall Logic
```
for each contact:
  for each step in waterfall:
    run step
    if verified_email found AND confidence > 0.8:
      break  ← short-circuit, skip all remaining paid steps
    log result to enrichment_log[]
  calculate enrichment_score
  write to lb_contacts
```

### Step-by-Step Waterfall

| Step | Source | Cost | What You Get | API / Method |
|---|---|---|---|---|
| 1 | **Company website** | FREE (Apify compute) | Names, titles, emails, phones from HTML | Apify Actor A1 |
| 2 | **Google Search** | FREE (SerpAPI 100/mo) | Team page URLs | SerpAPI / direct Google |
| 3 | **LinkedIn public** | FREE (Apify) | Name, title, company, location | Apify Actor A2 |
| 4 | **GitHub commit emails** | FREE (5000 req/hr) | Developer emails from public commits | `api.github.com/search/commits` |
| 5 | **WHOIS registrant** | FREE | Domain owner name/email (SMBs) | `whois-json.com/api/{domain}` |
| 6 | **Google Places** | FREE under 1K/mo | Phone, business hours, website | Places API text search |
| 7 | **ProPublica Nonprofit** | FREE | Leadership of 501(c)s | `projects.propublica.org/nonprofits/api/v2` |
| 8 | **SEC EDGAR** | FREE | Exec names + titles for public cos | `efts.sec.gov/LATEST/search-index` |
| 9 | **OpenCorporates** | FREE (basic) | Officers/directors of registered cos | `api.opencorporates.com/v0.4` |
| 10 | **FullContact** | FREE (100/mo) | Person + company enrichment | `api.fullcontact.com/v3/person.enrich` |
| 11 | **Clearbit** | FREE (basic tier) | Company + person basics | `person.clearbit.com/v2/combined` |
| 12 | **Hunter.io** | PAID ($49/mo) | Named email lookup + verification | `api.hunter.io/v2/email-finder` |
| 13 | **Apollo.io** | PAID ($49/mo) | Full person + company enrichment | `api.apollo.io/v1/people/match` |
| 14 | **PDL** | PAID (pay-as-go) | Person data by name/domain | `api.peopledatalabs.com/v5/person/enrich` |
| 15 | **Snov.io** | PAID ($39/mo) | Email from LinkedIn URL | `api.snov.io/v2/get-emails-from-url` |
| 16 | **RocketReach** | PAID ($80/mo) | LinkedIn profile enrichment | `api.rocketreach.co/v2/api/search` |
| 17 | **ZeroBounce** | PAID ($0.003/ea) | Email deliverability validation | `api.zerobounce.net/v2/validate` |
| 18 | **MillionVerifier** | PAID ($0.002/ea) | Second validation pass | `api.millionverifier.com/api/v3` |
| 19 | **NeverBounce** | PAID ($0.008/ea) | Third validation pass (hard confirm) | `api.neverbounce.com/v4.2/single/check` |
| 20 | **BuiltWith** | FREE (limited) | What software the company runs | `api.builtwith.com/free1/api.json` |
| 21 | **Wappalyzer** | FREE (Apify) | Tech stack detection | Apify `wappalyzer-scraper` actor |
| 22 | **LinkedIn connect check** | CUSTOM | 1st/2nd degree paths via your team | Internal lb_team_connections table |

### Enrichment Score Calculation

```javascript
// WF-05 Function node: calculate_enrichment_score
function calculateScore(contact) {
  let score = 0;
  if (contact.email)              score += 15;
  if (contact.email_verified === 'hard') score += 20;
  if (contact.email_verified === 'risky') score += 5;
  if (contact.phone)              score += 10;
  if (contact.linkedin_url)       score += 10;
  if (contact.title)              score += 5;
  if (contact.location_city)      score += 5;
  if (contact.company_size)       score += 5;
  if (contact.tech_stack?.length) score += 10;
  if (contact.linkedin_connection_degree <= 2) score += 15;
  // Cap at 100
  return Math.min(score, 100);
}
```

---

## 5. SUPABASE SCHEMA v2

### Run These Migrations in Order

```sql
-- ═══════════════════════════════════════
-- MIGRATION 001: Core queue and contacts
-- ═══════════════════════════════════════

CREATE TABLE lb_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  company TEXT NOT NULL,
  domain TEXT NOT NULL,
  vertical TEXT DEFAULT 'general',
  source TEXT DEFAULT 'manual',
  priority INT DEFAULT 5,
  status TEXT DEFAULT 'pending' CHECK (
    status IN ('pending','in_progress','complete','failed','retry','stale')
  ),
  retry_count INT DEFAULT 0,
  error_message TEXT,
  last_enriched_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  completed_at TIMESTAMPTZ,
  UNIQUE(domain)
);

CREATE TABLE lb_pages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  queue_id UUID REFERENCES lb_queue(id) ON DELETE CASCADE,
  url TEXT NOT NULL,
  page_type TEXT DEFAULT 'unknown',
  scraped BOOLEAN DEFAULT FALSE,
  scrape_status TEXT DEFAULT 'pending',
  html_length INT,
  depth INT DEFAULT 0,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE lb_contacts_raw (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  queue_id UUID REFERENCES lb_queue(id) ON DELETE CASCADE,
  page_id UUID REFERENCES lb_pages(id),
  company TEXT,
  domain TEXT,
  full_name TEXT,
  first_name TEXT,
  last_name TEXT,
  title TEXT,
  email_raw TEXT,
  phone_raw TEXT,
  linkedin_url TEXT,
  source_url TEXT,
  extraction_confidence FLOAT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE lb_contacts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  queue_id UUID REFERENCES lb_queue(id),
  raw_contact_id UUID REFERENCES lb_contacts_raw(id),
  company TEXT,
  domain TEXT,
  full_name TEXT NOT NULL,
  first_name TEXT,
  last_name TEXT,
  title TEXT,
  seniority TEXT,
  department TEXT,
  -- Email fields
  email TEXT,
  email_verified TEXT DEFAULT 'unverified' CHECK (
    email_verified IN ('hard','risky','unverified','invalid','unknown')
  ),
  email_source TEXT,
  -- Contact fields
  phone TEXT,
  phone_source TEXT,
  linkedin_url TEXT,
  linkedin_source TEXT,
  twitter_url TEXT,
  github_url TEXT,
  -- Location
  location_city TEXT,
  location_state TEXT,
  location_country TEXT,
  -- Company data
  company_size TEXT,
  company_industry TEXT,
  company_linkedin TEXT,
  company_founded_year INT,
  company_revenue_range TEXT,
  -- Technographics
  tech_stack JSONB DEFAULT '[]',
  tech_stack_source TEXT,
  -- Scoring
  enrichment_score INT DEFAULT 0,
  icp_score INT DEFAULT 0,
  icp_score_reason TEXT,
  priority_score INT DEFAULT 0,
  -- Connection intelligence
  linkedin_connection_degree INT,
  linkedin_connection_path TEXT,
  linkedin_connection_via TEXT,
  -- Enrichment metadata
  enrichment_sources JSONB DEFAULT '[]',
  enrichment_log JSONB DEFAULT '[]',
  -- Outreach
  outreach_status TEXT DEFAULT 'pending' CHECK (
    outreach_status IN ('pending','draft_ready','approved','sent','replied','bounced','unsubscribed')
  ),
  outreach_draft TEXT,
  outreach_sequence_id TEXT,
  -- Lifecycle
  status TEXT DEFAULT 'enriched',
  exported_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Deduplication
CREATE TABLE lb_dedup (
  fingerprint TEXT PRIMARY KEY,
  contact_id UUID REFERENCES lb_contacts(id),
  domain TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Cross-client dedup guard
CREATE TABLE lb_global_contacts (
  email TEXT PRIMARY KEY,
  first_seen_queue_id UUID REFERENCES lb_queue(id),
  clients_targeting TEXT[] DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════
-- MIGRATION 002: Intelligence tables
-- ═══════════════════════════════════════

-- News and trigger events per company
CREATE TABLE lb_triggers (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  queue_id UUID REFERENCES lb_queue(id) ON DELETE CASCADE,
  domain TEXT,
  trigger_type TEXT CHECK (
    trigger_type IN ('new_hire','leadership_change','funding','expansion',
                     'press_release','award','intent_signal','linkedin_post')
  ),
  trigger_title TEXT,
  trigger_url TEXT,
  trigger_date TIMESTAMPTZ,
  trigger_score INT DEFAULT 0,
  processed BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Intent signals
CREATE TABLE lb_intent (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  domain TEXT,
  intent_topic TEXT,
  intent_source TEXT,
  intent_strength INT,
  recorded_at TIMESTAMPTZ DEFAULT NOW()
);

-- ICP definition (your Ideal Customer Profile)
CREATE TABLE lb_icp_definition (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  vertical TEXT NOT NULL,
  title_keywords TEXT[] DEFAULT '{}',
  seniority_levels TEXT[] DEFAULT '{}',
  company_size_range TEXT,
  tech_stack_required TEXT[] DEFAULT '{}',
  tech_stack_bonus TEXT[] DEFAULT '{}',
  geography TEXT[] DEFAULT '{}',
  disqualifiers TEXT[] DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Win/loss feedback for ICP refinement
CREATE TABLE lb_feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contact_id UUID REFERENCES lb_contacts(id),
  outcome TEXT CHECK (outcome IN ('won','lost','no_response','unqualified')),
  deal_value NUMERIC,
  feedback_notes TEXT,
  recorded_at TIMESTAMPTZ DEFAULT NOW()
);

-- Team LinkedIn connections (for connection intelligence)
CREATE TABLE lb_team_connections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  team_member TEXT,
  team_member_linkedin TEXT,
  connection_name TEXT,
  connection_linkedin TEXT,
  connection_company TEXT,
  degree INT DEFAULT 1,
  last_synced TIMESTAMPTZ DEFAULT NOW()
);

-- Outreach queue
CREATE TABLE lb_outreach (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contact_id UUID REFERENCES lb_contacts(id),
  queue_id UUID REFERENCES lb_queue(id),
  icp_score INT,
  subject_line TEXT,
  email_body TEXT,
  personalization_notes TEXT,
  sequence_name TEXT,
  approval_status TEXT DEFAULT 'pending' CHECK (
    approval_status IN ('pending','approved','rejected','sent')
  ),
  approved_by TEXT,
  approved_at TIMESTAMPTZ,
  instantly_sequence_id TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Lookalike expansion queue
CREATE TABLE lb_lookalike_queue (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  source_domain TEXT,
  source_company TEXT,
  lookalike_domain TEXT,
  lookalike_company TEXT,
  similarity_reason TEXT,
  status TEXT DEFAULT 'pending',
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- ═══════════════════════════════════════
-- MIGRATION 003: Indexes
-- ═══════════════════════════════════════

CREATE INDEX idx_queue_status ON lb_queue(status);
CREATE INDEX idx_queue_domain ON lb_queue(domain);
CREATE INDEX idx_queue_vertical ON lb_queue(vertical);
CREATE INDEX idx_contacts_domain ON lb_contacts(domain);
CREATE INDEX idx_contacts_email ON lb_contacts(email);
CREATE INDEX idx_contacts_icp ON lb_contacts(icp_score DESC);
CREATE INDEX idx_contacts_outreach ON lb_contacts(outreach_status);
CREATE INDEX idx_raw_queue ON lb_contacts_raw(queue_id);
CREATE INDEX idx_pages_queue ON lb_pages(queue_id);
CREATE INDEX idx_triggers_domain ON lb_triggers(domain);
CREATE INDEX idx_intent_domain ON lb_intent(domain);
CREATE INDEX idx_dedup_domain ON lb_dedup(domain);

-- ═══════════════════════════════════════
-- MIGRATION 004: Triggers and functions
-- ═══════════════════════════════════════

CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = NOW(); RETURN NEW; END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_queue_updated BEFORE UPDATE ON lb_queue
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();
CREATE TRIGGER trg_contacts_updated BEFORE UPDATE ON lb_contacts
  FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Priority score: combines icp_score + enrichment_score + trigger bonus + intent bonus
CREATE OR REPLACE FUNCTION calculate_priority_score(
  p_icp_score INT,
  p_enrichment_score INT,
  p_has_trigger BOOLEAN,
  p_has_intent BOOLEAN,
  p_connection_degree INT
) RETURNS INT AS $$
BEGIN
  RETURN LEAST(100,
    (p_icp_score * 0.5) +
    (p_enrichment_score * 0.2) +
    (CASE WHEN p_has_trigger THEN 15 ELSE 0 END) +
    (CASE WHEN p_has_intent THEN 20 ELSE 0 END) +
    (CASE WHEN p_connection_degree = 1 THEN 15
          WHEN p_connection_degree = 2 THEN 8 ELSE 0 END)
  );
END;
$$ LANGUAGE plpgsql;

-- ═══════════════════════════════════════
-- MIGRATION 005: ICP seed data
-- ═══════════════════════════════════════

INSERT INTO lb_icp_definition (vertical, title_keywords, seniority_levels, company_size_range, tech_stack_required, tech_stack_bonus, geography, disqualifiers) VALUES
(
  'behavioral_health',
  ARRAY['Medical Director','Clinical Director','Director of Admissions','CEO','COO','Chief Operating','VP Operations','Compliance Officer','Director of Operations','Utilization Review'],
  ARRAY['C-Suite','VP','Director','Manager'],
  '10-500',
  ARRAY[]::TEXT[],
  ARRAY['Kipu','Salesforce','Office Ally','TheraNest','SimplePractice'],
  ARRAY['Arizona','California','Texas','Florida','Nevada'],
  ARRAY['student','intern','volunteer']
),
(
  'construction',
  ARRAY['Project Manager','Estimator','VP Operations','Chief Operating','CEO','Superintendent','Director','Business Development'],
  ARRAY['C-Suite','VP','Director','Manager'],
  '20-1000',
  ARRAY[]::TEXT[],
  ARRAY['Procore','Buildertrend','PlanGrid','Bluebeam','Sage 300'],
  ARRAY['Arizona','California','Nevada','Texas'],
  ARRAY['subcontractor','laborer']
),
(
  'medical_transport',
  ARRAY['Operations Director','Fleet Manager','Director of Transport','CEO','COO','Dispatch Manager','Medical Director','Compliance'],
  ARRAY['C-Suite','VP','Director','Manager'],
  '20-500',
  ARRAY[]::TEXT[],
  ARRAY['Zoll','ESO','ImageTrend','Samsara','Verizon Connect'],
  ARRAY['Arizona','California','Nevada'],
  ARRAY['driver','EMT basic']
);
```

---

## 6. APIFY ACTORS

### Actor A1: `lb-website-scraper`
**Runtime:** Crawlee + Playwright (Node.js)

```javascript
// actors/lb-website-scraper/src/main.js
import { PlaywrightCrawler, Dataset } from 'crawlee';

const HIGH_VALUE_PATHS = ['/team','/about','/leadership','/management',
  '/people','/staff','/contact','/who-we-are','/our-team','/executives'];
const SKIP_PATHS = ['/blog','/news','/careers','/jobs','/press',
  '/resources','/insights','/events','/login','/signup'];

const { domain, maxPages = 20, maxDepth = 2 } = await Actor.getInput();

const scorer = (url) => {
  const path = new URL(url).pathname.toLowerCase();
  if (SKIP_PATHS.some(p => path.startsWith(p))) return -99;
  if (HIGH_VALUE_PATHS.some(p => path.startsWith(p))) return 10;
  return 1;
};

const crawler = new PlaywrightCrawler({
  maxRequestsPerCrawl: maxPages,
  async requestHandler({ page, request, enqueueLinks }) {
    const html = await page.content();
    const depth = request.userData.depth || 0;
    const pageType = HIGH_VALUE_PATHS.find(p =>
      new URL(request.url).pathname.toLowerCase().startsWith(p)
    ) ? 'team' : 'unknown';
    await Dataset.pushData({ url: request.url, html, statusCode: 200, pageType, depth });
    if (depth < maxDepth) {
      await enqueueLinks({
        transformRequestFunction: (req) => {
          req.userData = { depth: depth + 1 };
          req.userData.score = scorer(req.url);
          return req.userData.score > 0 ? req : false;
        }
      });
    }
  }
});

const startUrls = [`https://${domain}`, `https://www.${domain}`];
for (const hv of HIGH_VALUE_PATHS) {
  startUrls.push(`https://${domain}${hv}`, `https://www.${domain}${hv}`);
}
await crawler.run(startUrls.map(url => ({ url, userData: { depth: 0 } })));
```

---

### Actor A2: `lb-linkedin-company-scraper`
Use Apify's built-in `apify/linkedin-companies-scraper`. Wrap with config:

```json
{
  "startUrls": [{ "url": "https://www.linkedin.com/company/{slug}/people/" }],
  "maxItems": 50,
  "proxyConfiguration": { "useApifyProxy": true }
}
```

**Output mapping:** Extract `{ name, title, linkedinUrl, location }` per person.

---

### Actor A3: `lb-google-search-scraper`
Use Apify's built-in `apify/google-search-scraper` with pre-built query templates:

```json
{
  "queries": [
    "site:{domain} intitle:team OR leadership OR management OR staff",
    "site:{domain} intitle:about OR people OR executives OR directors",
    "\"{company}\" leadership site:linkedin.com"
  ],
  "resultsPerPage": 10,
  "maxPagesPerQuery": 1
}
```

---

## 7. n8n WORKFLOWS 01–10

---

### WF-01: Master Orchestrator
**File:** `workflows/01-master-orchestrator.json`
**Trigger:** Webhook POST `/webhook/lead-booster/enrich`

```
Nodes:
[Webhook]
  → [Function: Validate]
      // Required: company (string), domain (string)
      // Optional: vertical (default 'general'), source, priority (1-10)
      // Normalize domain: strip https://, www., trailing slash
  → [Supabase: Check Dedup]
      // SELECT id, status, last_enriched_at FROM lb_queue WHERE domain = $domain
  → [If: Exists AND not stale?]
      EXISTS & fresh → [Respond 409 Already Processing/Complete]
      EXISTS & stale → [Supabase: Reset to pending] → trigger WF-02
      NOT EXISTS    → [Supabase: Insert lb_queue] → trigger WF-02
  → [HTTP: POST to WF-02 Webhook]
  → [Respond 202 Accepted]
      { status: 'queued', queue_id, company, domain, estimated_minutes: 5 }
```

---

### WF-02: Company Discovery
**File:** `workflows/02-company-discovery.json`
**Trigger:** Webhook (called by WF-01)
**Input:** `{ queue_id, company, domain, vertical }`

```
[Webhook]
  → [Supabase: Update status = in_progress]
  → [Parallel - Split In Batches node]:
      Branch A: [Apify: Run Actor A3 - Google Search]
                { domain, company } → returns [{ url, title }]
      Branch B: [Apify: Run Actor A1 - Website Scraper]
                { domain } → returns [{ url, html, pageType }]
      Branch C: [Function: Build Pattern URLs]
                // Generate /team, /about, /leadership etc from domain
      Branch D: [HTTP: LinkedIn Company Search]
                // GET company LinkedIn URL via Google: "{company}" site:linkedin.com/company
  → [Merge All Branches]
  → [Function: Deduplicate + Score URLs]
      // Remove duplicates, score by path relevance
      // Store HTML blobs separately in lb_pages_html (Supabase storage bucket)
  → [Supabase: Insert lb_pages] (one row per URL)
  → [HTTP: POST to WF-09 Webhook] (parallel trigger for news monitoring)
  → [HTTP: POST to WF-03 Webhook] (with queue_id + page list)
```

---

### WF-03: Scrape & AI Extract (Claude-Native)
**File:** `workflows/03-scrape-and-extract.json`
**Trigger:** Webhook (called by WF-02)
**Input:** `{ queue_id, vertical, pages: [{page_id, url, html?}] }`

```
[Webhook]
  → [Split In Batches: 5 pages]
  → [If: HTML already fetched by Apify?]
      YES → skip fetch
      NO  → [HTTP Request: GET URL, timeout: 15s, follow redirects: 5]
  → [Supabase: Update lb_pages scraped=true]
  → [Function: JS HTML Normalizer]
      // See §9 Claude Prompt Library for full normalizer code
  → [Claude API: Extract People]
      // Model: claude-sonnet-4-20250514
      // System: vertical-specific prompt from lb_icp_definition
      // User: normalized HTML
      // Expected response: JSON { classification, people[], subpage_urls[] }
  → [Function: Parse Claude Response]
      // Parse JSON, validate schema
      // Handle: complete / partial / subpages / no_people
  → [If: classification = subpages?]
      YES → [HTTP: POST back to WF-02] (subpage_urls, depth + 1, depth guard: max 3)
  → [Function: Split people array]
  → [Supabase: Insert lb_contacts_raw] (one row per person)
  → [HTTP: POST to WF-04] (queue_id, raw_contact_ids[])
```

**Claude node config:**
```json
{
  "model": "claude-sonnet-4-20250514",
  "max_tokens": 2000,
  "system": "{{ $node['Supabase - Get ICP'].data.claude_extraction_prompt }}",
  "messages": [{
    "role": "user",
    "content": "Company: {{ $json.company }}\nURL: {{ $json.url }}\n\nHTML:\n{{ $json.normalized_html }}"
  }]
}
```

---

### WF-04: Enrichment Orchestrator
**File:** `workflows/04-enrichment-orchestrator.json`
**Trigger:** Webhook (called by WF-03)
**Input:** `{ queue_id, raw_contact_ids[] }`

```
[Webhook]
  → [Supabase: Fetch Raw Contacts]
      SELECT * FROM lb_contacts_raw WHERE id = ANY($raw_contact_ids)
  → [Function: Deduplicate]
      // fingerprint = lower(full_name).replace(/\s+/g,'.') + '@' + domain
      // Check lb_dedup — skip if exists
  → [Function: Cross-Client Guard]
      // Check lb_global_contacts — flag if email already in another vertical
      // If conflict: set priority_score -= 20, add note
  → [Split In Batches: 10 contacts]
  → [HTTP: POST to WF-05] (one contact per call, parallel execution)
  → [Wait: Poll lb_contacts until all IDs present OR timeout 10min]
  → [HTTP: POST to WF-08] (queue_id for ICP scoring)
```

---

### WF-05: Enrichment Waterfall
**File:** `workflows/05-enrichment-waterfall.json`
**Trigger:** Webhook (called by WF-04 per contact)
**Input:** `{ raw_contact_id, full_name, first_name, last_name, domain, title, email_raw, linkedin_url }`

```
[Webhook]
  → [Initialize: enrichment_log = [], verified_email = null]

  ── FREE SOURCES ──────────────────────────────────────────────────

  [Step 1: GitHub Email Search]
    GET https://api.github.com/search/commits
      ?q=author-name:{first_name}+{last_name}+{domain}
    Authorization: Bearer $GITHUB_TOKEN
    Extract: commit.author.email if not noreply
    → IF found: set email candidate, source = 'github'

  [If: verified_email found?] → jump to VALIDATION

  [Step 2: WHOIS Registrant]
    GET https://whois-json.com/api/{domain}
    Extract: registrant_email, registrant_name
    → IF registrant_email AND name_match > 0.6: set email candidate

  [If: verified_email found?] → jump to VALIDATION

  [Step 3: Google Places]
    GET https://maps.googleapis.com/maps/api/place/textsearch/json
      ?query={company}+{city}&key=$GOOGLE_PLACES_KEY
    Extract: phone, website, hours
    → Augments contact.phone if missing (not email source)

  [Step 4: ProPublica / EDGAR / OpenCorporates]
    [If: vertical = behavioral_health]:
      GET https://projects.propublica.org/nonprofits/api/v2/search.json?q={company}
      Extract: executives[].name + title match
    [If: vertical = general AND company_size > 500]:
      GET https://efts.sec.gov/LATEST/search-index?q={company}&dateRange=custom&startdt=2023-01-01
      Extract: filer contact persons

  [Step 5: FullContact]
    POST https://api.fullcontact.com/v3/person.enrich
    { "email": "{email_raw}", "person": { "name": { "full": "{full_name}" } } }
    Authorization: Bearer $FULLCONTACT_KEY
    → Extract: emails[], phones[], social[], location, employment

  [If: verified_email found?] → jump to VALIDATION

  [Step 6: Clearbit]
    GET https://person.clearbit.com/v2/combined/find?email={email_raw}&name={full_name}&company_domain={domain}
    Authorization: Bearer $CLEARBIT_KEY
    → Extract: person.email, person.employment.title, company.*

  [If: verified_email found?] → jump to VALIDATION

  ── PAID SOURCES ──────────────────────────────────────────────────

  [Step 7: Hunter.io Email Finder]
    GET https://api.hunter.io/v2/email-finder
      ?domain={domain}&first_name={first_name}&last_name={last_name}&api_key=$HUNTER_KEY
    → IF score > 70: set email candidate, source = 'hunter'

  [If: verified_email found?] → jump to VALIDATION

  [Step 8: Apollo.io Person Match]
    POST https://api.apollo.io/v1/people/match
    { "first_name": ..., "last_name": ..., "domain": ..., "linkedin_url": ... }
    x-api-key: $APOLLO_KEY
    → Extract: email, phone, title, seniority, department, linkedin, company.*

  [If: verified_email found?] → jump to VALIDATION

  [Step 9: PDL Person Enrich]
    POST https://api.peopledatalabs.com/v5/person/enrich
    { "params": { "name": "{full_name}", "company": "{company}", "location": "..." } }
    x-api-key: $PDL_KEY
    → Extract: emails[], phone_numbers[], job_title, location, company.*

  [If: verified_email found?] → jump to VALIDATION

  [Step 10: Snov.io Email from LinkedIn]
    GET https://api.snov.io/v2/get-emails-from-url?url={linkedin_url}
    Authorization: Bearer $SNOVIO_TOKEN
    → Extract: emails[0].email if confidence > 70

  [Step 11: RocketReach]
    POST https://api.rocketreach.co/v2/api/search
    { "name": "{full_name}", "current_employer": "{company}", "linkedin": "{linkedin_url}" }
    api_key: $ROCKETREACH_KEY
    → Extract: emails[], phones[], linkedin_url

  ── EMAIL VALIDATION ──────────────────────────────────────────────

  [Step 12: ZeroBounce]
    GET https://api.zerobounce.net/v2/validate
      ?api_key=$ZEROBOUNCE_KEY&email={email_candidate}
    status: 'valid' → continue; 'invalid' → clear email, try next source

  [Step 13: MillionVerifier] (if ZeroBounce says risky)
    GET https://api.millionverifier.com/api/v3/?api=$MILLIONVERIFIER_KEY&email={email}
    result: 'ok' → upgrade to risky_verified

  [Step 14: NeverBounce] (if email still unconfirmed after 12+13)
    GET https://api.neverbounce.com/v4.2/single/check
      ?api_key=$NEVERBOUNCE_KEY&email={email}
    result: 'valid' → set email_verified = 'hard'
    result: 'catchall' → set email_verified = 'risky'
    result: 'invalid' → clear, mark invalid

  ── AUGMENTATION ──────────────────────────────────────────────────

  [Step 15: BuiltWith Tech Stack]
    GET https://api.builtwith.com/free1/api.json?KEY=$BUILTWITH_KEY&LOOKUP={domain}
    → Extract: Results[].Paths[].Technologies[].Name → tech_stack[]

  [Step 16: Wappalyzer]
    Apify actor: wappalyzer-scraper
    Input: { urls: ["https://{domain}"] }
    → Extract: technologies[] → merge with tech_stack[]

  [Step 17: LinkedIn Connection Check]
    SELECT * FROM lb_team_connections
    WHERE connection_company ILIKE '%{company}%'
       OR connection_linkedin = '{linkedin_url}'
    → IF found: set linkedin_connection_degree, linkedin_connection_via

  ── WRITE RESULT ──────────────────────────────────────────────────

  [Function: Build Final Contact]
    // Merge all enrichment data (best value wins strategy)
    // Calculate enrichment_score using score function
    // Set email_verified classification

  [Supabase: Insert lb_contacts]
  [Supabase: Insert lb_dedup] (fingerprint)
  [Supabase: Update lb_global_contacts] (email → clients array)
  [Respond 200]
```

---

### WF-06: Output Router
**File:** `workflows/06-output-router.json`
**Trigger:** Webhook (called by WF-08 after ICP scoring)
**Input:** `{ queue_id }`

```
[Webhook]
  → [Supabase: Fetch Queue + Contacts]
  → [Switch: vertical]
      'behavioral_health' → POST to Sanctuary/New Freedom/Cholla webhook
      'construction'      → POST to Cornerstone CRM webhook
      'medical_transport' → POST to Sunstate workflow webhook
      'default'           → [Google Sheets: Append Rows] + [Supabase Storage: Export CSV]
  → [Supabase: Update lb_queue] (status = complete, completed_at = NOW())
  → [Email: Notify Dave + Brian]
      Subject: "Lead Booster Complete: {company} — {contact_count} contacts, avg ICP {avg_score}"
      Body: top 10 contacts by priority_score as HTML table
```

---

### WF-07: Retry, Maintenance & Refresh
**File:** `workflows/07-retry-maintenance.json`
**Trigger:** Cron — every 10 minutes

```
[Cron]
  → [Supabase: Find Stuck Jobs]
      WHERE status = 'in_progress' AND updated_at < NOW() - INTERVAL '30 min'
      OR status = 'failed' AND retry_count < 3
  → [Loop: each stuck job]
      → [Supabase: Increment retry_count]
      → [HTTP: Re-trigger WF-01]
      → [If: retry_count >= 3]
           → [Email: Alert brian@manageai.io]

  → [Supabase: Find Stale Contacts] (last_enriched_at < NOW() - INTERVAL '90 days')
  → [Loop: each stale domain]
      → [Supabase: Reset queue status = 'pending']
      → [HTTP: Re-trigger WF-01] (source = 'auto_refresh')

  → [Supabase: Find Expired Dedup]
      DELETE FROM lb_dedup WHERE created_at < NOW() - INTERVAL '90 days'
```

---

### WF-08: ICP Scoring Engine + Lookalike Expansion
**File:** `workflows/08-icp-scorer.json`
**Trigger:** Webhook (called by WF-04 after enrichment)
**Input:** `{ queue_id }`

```
[Webhook]
  → [Supabase: Fetch Contacts] (all enriched for queue_id, status != scored)
  → [Supabase: Fetch ICP Definition] (by vertical)
  → [Supabase: Fetch Win Feedback] (last 90 days for vertical)
  → [Split In Batches: 20 contacts]
  → [Claude API: ICP Score Batch]
      // Model: claude-sonnet-4-20250514
      // System: ICP scoring prompt (see §9)
      // Input: contact[] + icp_definition + win_examples[]
      // Output: [{ contact_id, icp_score, score_reason, seniority, department }]
  → [Function: Apply Multipliers]
      // intent_multiplier: +20 if domain in lb_intent (within 30 days)
      // trigger_bonus: +15 if domain in lb_triggers (within 14 days)
      // connection_bonus: already in enrichment_score
  → [Function: Calculate Priority Score]
      // SQL function calculate_priority_score()
  → [Supabase: Update lb_contacts]
      SET icp_score, icp_score_reason, priority_score, seniority, department
  → [Filter: icp_score >= 80 AND email_verified IN ('hard','risky')]
  → [HTTP: POST to WF-10] (high-score contacts for outreach draft)
  → [HTTP: POST to WF-06] (all contacts for output routing)

  ── LOOKALIKE EXPANSION ───────────────────────────────────────────

  → [Supabase: Find High-Score Companies] (avg icp_score > 70 for this queue)
  → [Apollo API: Company Search]
      // Find similar companies by industry, size, tech_stack, location
      POST https://api.apollo.io/v1/companies/search
      { filters: { industry, employee_count, technologies, locations } }
  → [Function: Filter Already Queued]
  → [Supabase: Insert lb_lookalike_queue]
  → [If: lookalike_count > 0]
      → [Email: Notify Brian]
           "Found {n} lookalike companies from {company}. Approve to queue:"
           [Link to approval UI]
```

---

### WF-09: News & Trigger Intelligence
**File:** `workflows/09-news-triggers.json`
**Trigger:** Webhook (called by WF-02) + Cron daily at 7am

```
[Webhook OR Cron]
  → [Supabase: Get Active Domains]
      SELECT DISTINCT domain, company, vertical FROM lb_queue WHERE status != 'pending'
  → [Split: each domain]
  → [Parallel]:

    Branch A: Google News RSS
      GET https://news.google.com/rss/search?q={company}&hl=en-US&gl=US&ceid=US:en
      Parse RSS → filter items in last 14 days
      → Classify title with Claude: trigger_type from allowed list
      → score by type: funding=25, leadership_change=20, expansion=15, press=10

    Branch B: LinkedIn Company Posts
      Apify: apify/linkedin-post-scraper
      Input: { companyUrl: "{linkedin_url}", maxPosts: 10 }
      → Claude: classify post intent: 'hiring' / 'expansion' / 'milestone' / 'other'

    Branch C: Intent Signals (Bombora / G2)
      GET https://api.bombora.com/topics
        // If you have Bombora access
      OR
      GET https://data.g2.com/api/v1/buyer-activity
        // If you have G2 access
      → Filter: topics matching your vertical keywords
      → Insert into lb_intent table

  → [Supabase: Upsert lb_triggers] (domain, trigger_type, title, url, date, score)
  → [Supabase: Update lb_contacts.priority_score] for contacts in triggered domain
```

---

### WF-10: Automated Outreach Handoff
**File:** `workflows/10-outreach-handoff.json`
**Trigger:** Webhook (called by WF-08 for icp_score >= 80)
**Input:** `{ contacts: [{id, full_name, title, company, icp_score_reason, tech_stack, trigger_events}] }`

```
[Webhook]
  → [Supabase: Fetch Full Contact + Company + Trigger Data]
  → [Split: each contact]
  → [Claude API: Write Personalized Email]
      // Model: claude-sonnet-4-20250514
      // System: outreach writing prompt (see §9)
      // Input: contact profile + ICP reason + trigger events + tech stack
      // Output: { subject, body, personalization_notes, sequence_name }
  → [Function: Validate Email]
      // Ensure <200 words, no spam triggers, has CTA
      // Validate subject line < 60 chars
  → [Supabase: Insert lb_outreach]
      { contact_id, icp_score, subject, email_body, approval_status: 'pending' }
  → [Supabase: Update lb_contacts] (outreach_status = 'draft_ready')
  → [Email: Notify Brian]
      Subject: "{n} outreach drafts ready for approval"
      Body: card per contact — name, title, company, ICP score, email preview
      [Approve All] button → POST to approval webhook
      [Review Each] link → Supabase table link

  ── APPROVAL WEBHOOK (separate endpoint) ──────────────────────────

  [Webhook: /webhook/outreach/approve]
  Input: { outreach_id, action: 'approve'|'reject' }
  → [If: approve]
      → [Instantly.ai API: Add Lead to Sequence]
          POST https://api.instantly.ai/api/v1/lead/add
          { api_key, campaign_id, email, first_name, last_name,
            personalization: subject + body }
      → [Supabase: Update lb_outreach] (approval_status = 'approved', instantly_sequence_id)
      → [Supabase: Update lb_contacts] (outreach_status = 'sent')
  → [If: reject]
      → [Supabase: Update lb_outreach] (approval_status = 'rejected')
```

---

## 8. SUPABASE EDGE FUNCTIONS

### Function 1: `enrich-webhook` (Public intake)

```typescript
// supabase/functions/enrich-webhook/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import { createClient } from "https://esm.sh/@supabase/supabase-js@2"

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type, x-api-key",
}

serve(async (req) => {
  if (req.method === "OPTIONS") return new Response("ok", { headers: corsHeaders })

  const apiKey = req.headers.get("x-api-key")
  if (apiKey !== Deno.env.get("INTAKE_API_KEY")) {
    return new Response(JSON.stringify({ error: "Unauthorized" }), {
      status: 401, headers: { ...corsHeaders, "Content-Type": "application/json" }
    })
  }

  const { company, domain, vertical = "general", source = "api", priority = 5 } = await req.json()

  if (!company || !domain) {
    return new Response(JSON.stringify({ error: "company and domain required" }), {
      status: 400, headers: { ...corsHeaders, "Content-Type": "application/json" }
    })
  }

  const cleanDomain = domain.replace(/^https?:\/\//,'').replace(/^www\./,'').replace(/\/.*$/,'').toLowerCase()

  const supabase = createClient(
    Deno.env.get("SUPABASE_URL")!,
    Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!
  )

  const { data: existing } = await supabase
    .from("lb_queue").select("id, status, last_enriched_at").eq("domain", cleanDomain).single()

  if (existing && existing.status !== 'stale') {
    return new Response(JSON.stringify({
      status: "exists", queue_id: existing.id, queue_status: existing.status
    }), { status: 409, headers: { ...corsHeaders, "Content-Type": "application/json" } })
  }

  let queue
  if (existing) {
    const { data } = await supabase.from("lb_queue")
      .update({ status: 'pending', retry_count: 0, source })
      .eq("id", existing.id).select().single()
    queue = data
  } else {
    const { data } = await supabase.from("lb_queue")
      .insert({ company, domain: cleanDomain, vertical, source, priority })
      .select().single()
    queue = data
  }

  await fetch(Deno.env.get("N8N_ORCHESTRATOR_WEBHOOK")!, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ queue_id: queue.id, company, domain: cleanDomain, vertical, priority })
  })

  return new Response(JSON.stringify({
    status: "queued", queue_id: queue.id, estimated_minutes: 5
  }), { status: 202, headers: { ...corsHeaders, "Content-Type": "application/json" } })
})
```

---

### Function 2: `contacts-api` (Read contacts)

**Endpoints:**
- `GET ?domain=acmecorp.com` → contacts for domain, sorted by priority_score DESC
- `GET ?queue_id=uuid` → contacts for job
- `GET ?vertical=behavioral_health&min_icp=70&limit=100` → filtered list
- `GET ?outreach_status=draft_ready` → pending approval queue

---

### Function 3: `outreach-approve` (Approval webhook)

Simple webhook that receives `{ outreach_id, action }` and calls the WF-10 approval branch.

---

## 9. CLAUDE PROMPT LIBRARY

All prompts stored in Supabase `lb_icp_definition.claude_extraction_prompt` and overridden per vertical.

---

### Prompt 1: HTML People Extractor (Vertical-Tuned)

**Base System Prompt:**
```
You are a structured contact extraction engine operating in a production lead enrichment pipeline. Your output is consumed by automated downstream systems. Every deviation from the output schema breaks the pipeline.

ABSOLUTE RULES:
1. Return ONLY valid JSON. Zero preamble, zero markdown, zero explanation.
2. Include a person ONLY if they have BOTH a first name AND a last name (minimum 2 distinct tokens).
3. Include a person ONLY if they have a job title explicitly associated with them on the page.
4. NEVER invent or infer email, phone, or LinkedIn not explicitly present on the page.
5. If a field is not explicitly present for that person, omit the field entirely (do not null it).
6. Confidence: float 0.0–1.0 representing how certain you are the data is correctly attributed.

CLASSIFICATION (required):
- "complete": people found with sufficient data
- "partial": people found but key fields missing
- "subpages": page contains links to individual team member subpages (return urls)
- "no_people": no qualifying people found

OUTPUT SCHEMA:
{
  "classification": "complete|partial|subpages|no_people",
  "people": [
    {
      "full_name": "Jane Smith",
      "first_name": "Jane",
      "last_name": "Smith",
      "title": "Director of Clinical Services",
      "email": "jane@company.com",
      "phone": "+1-555-123-4567",
      "linkedin_url": "https://linkedin.com/in/janesmith",
      "confidence": 0.95
    }
  ],
  "subpage_urls": []
}
```

**Behavioral Health Vertical Addition:**
```
PRIORITY TITLES for this vertical (extract even if not leadership):
Medical Director, Clinical Director, Director of Admissions, Director of Operations,
Chief Clinical Officer, CEO, COO, VP Operations, Compliance Officer, Utilization Review Manager,
Business Development Director, Marketing Director, Case Manager Supervisor

SKIP: volunteers, interns, students, contractors listed without title
```

**Construction Vertical Addition:**
```
PRIORITY TITLES for this vertical:
Project Manager, Senior Project Manager, Estimator, Chief Estimator,
VP Operations, VP Business Development, CEO, COO, Superintendent,
Director of Pre-Construction, Safety Director, Field Operations Manager
```

**Medical Transport Vertical Addition:**
```
PRIORITY TITLES for this vertical:
Operations Director, Fleet Operations Manager, Director of Transportation,
CEO, COO, Dispatch Supervisor, Medical Director, Compliance Manager,
Business Development Manager, Billing Manager
```

---

### Prompt 2: ICP Scorer

**System Prompt:**
```
You are an ICP (Ideal Customer Profile) scoring engine for a B2B AI automation agency.
ManageAI builds custom workflow automation for SMBs in behavioral health, construction, and medical transport.

You will receive:
- A batch of enriched contacts as JSON
- The ICP definition for their vertical
- Recent won deals as examples

Score each contact 0–100 where:
100 = Perfect match (decision-maker, right vertical, right size, known pain points)
70+ = Strong match (pursue immediately)
50–69 = Moderate match (worth nurturing)
<50 = Weak match (deprioritize)

Scoring factors:
- Title seniority and decision-making authority: 0–35 points
- Vertical and sub-vertical match: 0–20 points
- Company size appropriate for our deal size ($5K–$50K): 0–15 points
- Tech stack signals (running software we integrate with): 0–15 points
- Geographic relevance (Southwest US primary): 0–10 points
- Role in operations/process ownership (vs. purely clinical): 0–5 points

Return ONLY valid JSON array:
[
  {
    "contact_id": "uuid",
    "icp_score": 87,
    "score_reason": "Clinical Director at 85-bed behavioral health facility in AZ running Kipu — direct decision-maker for operational automation",
    "seniority": "director",
    "department": "clinical_operations"
  }
]
```

---

### Prompt 3: Outreach Email Writer

**System Prompt:**
```
You are an expert B2B outreach copywriter for ManageAI, an AI automation agency.
ManageAI builds custom Make.com, n8n, and AI workflow systems for behavioral health, construction, and medical transport companies.

Write a cold outreach email that:
- Is 100–160 words maximum (not one word more)
- Has a subject line under 55 characters
- Opens with a specific, research-backed observation (NOT "I noticed your website")
- Names a specific pain point relevant to their role and vertical
- Mentions ONE specific ManageAI capability relevant to that pain
- Has a single, low-friction CTA (15-minute call, not "let me know if you're interested")
- Sounds like it was written by a real person who did their homework
- Never mentions AI, automation, or software generically — be specific

Do NOT:
- Use "I hope this email finds you well"
- Use "I wanted to reach out"
- Use "synergy", "cutting-edge", "innovative", "streamline", "leverage"
- Include more than one CTA
- Be longer than 160 words

Return ONLY valid JSON:
{
  "subject": "...",
  "body": "...",
  "personalization_notes": "Key hooks used: trigger event, tech stack, role inference",
  "sequence_name": "behavioral_health_director|construction_pm|transport_ops"
}
```

---

### Prompt 4: News Trigger Classifier

**System Prompt:**
```
You are a business intelligence classifier. Classify the following news headline/snippet for a B2B sales team.

Trigger types:
- "new_hire": New executive or senior leader announced
- "leadership_change": CEO/COO/Director departed or replaced
- "funding": Investment, grant, or funding round received
- "expansion": New location, new service line, acquisition
- "press_release": General announcement (product, partnership, award)
- "irrelevant": No sales relevance

Score urgency 0–25:
25 = Act within 24 hours (new CEO/COO, major funding)
15 = Act within 1 week (expansion, new hire)
10 = Act within 2 weeks (award, press release)
0 = No urgency

Return ONLY: { "trigger_type": "...", "score": 0, "reason": "one sentence" }
```

---

### Shared: JS HTML Normalizer

```javascript
// WF-03 Function Node: normalize_html
// Strips junk while preserving people cards and JSON-LD

const company = $input.first().json.company || "";
const url = $input.first().json.url || "";
const html = $input.first().json.html || "";

function safeUrl(u) { try { return new URL(u); } catch { return null; } }

// Remove script tags (except JSON-LD), style, nav, footer, header, aside
let cleaned = html
  .replace(/<script(?![^>]*type=["']application\/ld\+json["'])[^>]*>[\s\S]*?<\/script>/gi, '')
  .replace(/<style[^>]*>[\s\S]*?<\/style>/gi, '')
  .replace(/<nav[^>]*>[\s\S]*?<\/nav>/gi, '')
  .replace(/<footer[^>]*>[\s\S]*?<\/footer>/gi, '')
  .replace(/<header[^>]*>[\s\S]*?<\/header>/gi, '')
  .replace(/<aside[^>]*>[\s\S]*?<\/aside>/gi, '');

// Remove cookie banners, chat widgets, ad containers
cleaned = cleaned.replace(/<div[^>]*(?:cookie|consent|gdpr|chat-widget|intercom)[^>]*>[\s\S]*?<\/div>/gi, '');

// Extract JSON-LD blocks and prepend
const jsonLdBlocks = [];
const jsonLdRegex = /<script[^>]*type=["']application\/ld\+json["'][^>]*>([\s\S]*?)<\/script>/gi;
let match;
while ((match = jsonLdRegex.exec(html)) !== null) {
  jsonLdBlocks.push(match[1].trim());
}

// Strip remaining HTML tags, normalize whitespace
let text = cleaned
  .replace(/<[^>]+>/g, ' ')
  .replace(/&nbsp;/g, ' ')
  .replace(/&amp;/g, '&')
  .replace(/&lt;/g, '<')
  .replace(/&gt;/g, '>')
  .replace(/\s{3,}/g, '\n\n')
  .trim();

// Truncate if massive (>50K chars) — keep first and last thirds (people cards often at bottom)
if (text.length > 50000) {
  const third = Math.floor(text.length / 3);
  text = text.slice(0, third * 1.5) + '\n\n[...TRUNCATED...]\n\n' + text.slice(-8000);
}

const result = {
  company,
  url,
  normalized_html: (jsonLdBlocks.length ? '=== JSON-LD ===\n' + jsonLdBlocks.join('\n') + '\n\n=== PAGE TEXT ===\n' : '') + text,
  original_length: html.length,
  normalized_length: text.length
};

return [{ json: result }];
```

---

## 10. CREDENTIALS & API ENDPOINTS

> ⚠️ Fill ALL values in `.env.build` before launching swarm. Gitignored — never commit.

```env
# ═══════════════════════════════
# SUPABASE
# ═══════════════════════════════
SUPABASE_URL=https://[ref].supabase.co
SUPABASE_ANON_KEY=[anon_key]
SUPABASE_SERVICE_ROLE_KEY=[service_role_key]
SUPABASE_DB_URL=postgresql://postgres:[password]@db.[ref].supabase.co:5432/postgres

# ═══════════════════════════════
# N8N
# ═══════════════════════════════
N8N_BASE_URL=https://n8n.[your-domain].com
N8N_API_KEY=[n8n_api_key]
N8N_ORCHESTRATOR_WEBHOOK=${N8N_BASE_URL}/webhook/lead-booster/enrich
N8N_DISCOVERY_WEBHOOK=${N8N_BASE_URL}/webhook/lb-discovery
N8N_EXTRACT_WEBHOOK=${N8N_BASE_URL}/webhook/lb-extract
N8N_ENRICH_ORCH_WEBHOOK=${N8N_BASE_URL}/webhook/lb-enrich-orch
N8N_WATERFALL_WEBHOOK=${N8N_BASE_URL}/webhook/lb-waterfall
N8N_ICP_WEBHOOK=${N8N_BASE_URL}/webhook/lb-icp
N8N_OUTREACH_WEBHOOK=${N8N_BASE_URL}/webhook/lb-outreach
N8N_OUTREACH_APPROVE_WEBHOOK=${N8N_BASE_URL}/webhook/outreach/approve

# ═══════════════════════════════
# APIFY
# ═══════════════════════════════
APIFY_API_TOKEN=[apify_token]
APIFY_ACTOR_WEBSITE=manageai~lb-website-scraper
APIFY_ACTOR_LINKEDIN=apify~linkedin-companies-scraper
APIFY_ACTOR_GOOGLE=apify~google-search-scraper
APIFY_ACTOR_WAPPALYZER=apify~website-content-crawler

# ═══════════════════════════════
# AI — CLAUDE (no OpenAI)
# ═══════════════════════════════
ANTHROPIC_API_KEY=[claude_api_key]
CLAUDE_MODEL=claude-sonnet-4-20250514

# ═══════════════════════════════
# ENRICHMENT — FREE TIER
# ═══════════════════════════════
GITHUB_TOKEN=[github_pat]
GOOGLE_PLACES_API_KEY=[places_key]
FULLCONTACT_API_KEY=[fullcontact_key]
CLEARBIT_API_KEY=[clearbit_key]
BUILTWITH_API_KEY=[builtwith_key]
SERPAPI_KEY=[serpapi_key]

# ═══════════════════════════════
# ENRICHMENT — PAID
# ═══════════════════════════════
HUNTER_API_KEY=[hunter_key]
APOLLO_API_KEY=[apollo_key]
PDL_API_KEY=[pdl_key]
SNOVIO_CLIENT_ID=[snov_id]
SNOVIO_CLIENT_SECRET=[snov_secret]
ROCKETREACH_API_KEY=[rr_key]

# ═══════════════════════════════
# EMAIL VALIDATION
# ═══════════════════════════════
ZEROBOUNCE_API_KEY=[zb_key]
MILLIONVERIFIER_API_KEY=[mv_key]
NEVERBOUNCE_API_KEY=[nb_key]

# ═══════════════════════════════
# OUTREACH
# ═══════════════════════════════
INSTANTLY_API_KEY=[instantly_key]
INSTANTLY_CAMPAIGN_BEHAVIORAL_HEALTH=[campaign_id]
INSTANTLY_CAMPAIGN_CONSTRUCTION=[campaign_id]
INSTANTLY_CAMPAIGN_TRANSPORT=[campaign_id]

# ═══════════════════════════════
# INTENT DATA (optional)
# ═══════════════════════════════
BOMBORA_API_KEY=[bombora_key]
G2_API_KEY=[g2_key]

# ═══════════════════════════════
# NOTIFICATIONS
# ═══════════════════════════════
INTAKE_API_KEY=[random_32_char_string]
DAVE_EMAIL=dave@manageai.io
BRIAN_EMAIL=brian@manageai.io
```

---

## 11. CLAUDE CODE SWARM INSTRUCTIONS

### Pre-Swarm (Manual — 30 min)

```bash
# 1. Create GitHub repo
gh repo create manageai/lead-booster-pro --private --clone
cd lead-booster-pro
cp LEAD_BOOSTER_BUILD_SPEC_V2.md ./
cp .env.example .env.build  # Fill in ALL values

# 2. Supabase — run all 5 migrations in SQL Editor
# 3. Verify n8n is live
curl ${N8N_BASE_URL}/healthz
# 4. Apify — create account, get token
# 5. Get all API keys from the credentials section
```

---

### Terminal 1: Supabase Edge Functions

```
Read LEAD_BOOSTER_BUILD_SPEC_V2.md in full.
Your job: Build ONLY the 3 Supabase Edge Functions.

1. supabase/functions/enrich-webhook/index.ts — full code from spec §8
2. supabase/functions/contacts-api/index.ts — read-only query endpoint (domain, queue_id, vertical filters)
3. supabase/functions/outreach-approve/index.ts — approval webhook

Set all secrets:
  supabase secrets set INTAKE_API_KEY=... SUPABASE_SERVICE_ROLE_KEY=... N8N_ORCHESTRATOR_WEBHOOK=...

Deploy:
  supabase functions deploy enrich-webhook
  supabase functions deploy contacts-api
  supabase functions deploy outreach-approve

Test enrich-webhook with a curl POST for domain "test.com".
Log all function URLs and responses to CLAUDE_CODE_LOG.md.
Commit to GitHub.
```

---

### Terminal 2: Apify Actors

```
Read LEAD_BOOSTER_BUILD_SPEC_V2.md in full.
Your job: Build ONLY Actor A1 (lb-website-scraper). Actors A2 and A3 use existing Apify actors.

1. Create actors/lb-website-scraper/ using Crawlee + Playwright
   Exact input schema and output schema from spec §6
   High-value path scoring, skip list, depth guard at maxDepth
   Handle SPAs (Playwright, not just HTTP fetch)

2. Push to Apify:
   cd actors/lb-website-scraper && apify push

3. Test actor:
   apify call manageai/lb-website-scraper --input='{"domain":"manageai.io","maxPages":5}'

4. Verify Actor A2 (apify/linkedin-companies-scraper) exists and is callable
5. Verify Actor A3 (apify/google-search-scraper) exists and is callable

Log all actor IDs to CLAUDE_CODE_LOG.md. Commit.
```

---

### Terminal 3: n8n Workflows 01–05

```
Read LEAD_BOOSTER_BUILD_SPEC_V2.md in full.
Your job: Build n8n workflows WF-01 through WF-05.

n8n base URL: [N8N_BASE_URL]
n8n API key: [N8N_API_KEY]
Import command: POST {N8N_BASE_URL}/api/v1/workflows
  Headers: Authorization: Bearer [N8N_API_KEY]
  Body: { "name": "...", "nodes": [...], "connections": {...}, "active": true }

Build each workflow as complete n8n JSON:
- WF-01: Master Orchestrator (webhook trigger)
- WF-02: Company Discovery (3 Apify actors in parallel)
- WF-03: Scrape & Extract (HTTP fetch + JS normalizer + Claude API call)
  IMPORTANT: Use Claude claude-sonnet-4-20250514, NOT OpenAI
  Use the JS normalizer from spec §9 in a Function node
  Load vertical-specific prompt from Supabase lb_icp_definition table
- WF-04: Enrichment Orchestrator (dedup + cross-client guard + fan-out)
- WF-05: Enrichment Waterfall (all 22 steps, short-circuit logic)
  Steps 1-11 enrichment, 12-14 validation, 15-17 augmentation

After importing each workflow:
1. Note the workflow ID and active webhook URL
2. Update previous workflows to point to correct webhook URLs
3. Test chain: WF-01 → WF-02 → WF-03 → WF-04 → WF-05

Log all workflow IDs and webhook URLs to CLAUDE_CODE_LOG.md. Commit.
```

---

### Terminal 4: n8n Workflows 06–10 + Scripts

```
Read LEAD_BOOSTER_BUILD_SPEC_V2.md in full.
Your job: Build n8n workflows WF-06 through WF-10 and all scripts.

n8n base URL: [N8N_BASE_URL]
n8n API key: [N8N_API_KEY]

Build and import:
- WF-06: Output Router (vertical switch + notify Dave)
- WF-07: Retry + 90-day refresh (cron every 10 min)
- WF-08: ICP Scorer + Lookalike Expansion (Claude API, see prompt in spec §9)
  IMPORTANT: Claude only — claude-sonnet-4-20250514
  Batch 20 contacts per Claude call for efficiency
  Include Apollo company search for lookalike expansion
- WF-09: News & Trigger Intelligence (Google News RSS + LinkedIn posts)
  Claude classifies each trigger — see Prompt 4 in spec §9
- WF-10: Outreach Handoff (Claude writes email, Instantly.ai API)
  Claude email writer prompt from spec §9
  Include approval webhook handler

Create all scripts:
- scripts/test_pipeline.sh (full end-to-end smoke test, 5min timeout)
- scripts/seed_data.sql (5 test companies, 3 verticals)
- scripts/import_workflows.sh (import all 10 workflows via n8n API in order)
- scripts/reset_test_data.sql (clean lb_queue/contacts for re-testing)

Log all workflow IDs and webhook URLs to CLAUDE_CODE_LOG.md.
Update CLAUDE_CODE_LOG.md with complete system URL map.
Commit and push.
```

---

## 12. GITHUB REPO STRUCTURE

```
lead-booster-pro/
├── LEAD_BOOSTER_BUILD_SPEC_V2.md      ← this file (agents read first)
├── CLAUDE_CODE_LOG.md                 ← agents write decisions + IDs here
├── .env.example                       ← template, no real values
├── .env.build                         ← real credentials (gitignored)
├── .gitignore
│
├── supabase/
│   ├── migrations/
│   │   ├── 001_core_tables.sql
│   │   ├── 002_intelligence_tables.sql
│   │   ├── 003_indexes.sql
│   │   ├── 004_triggers_functions.sql
│   │   └── 005_icp_seed_data.sql
│   └── functions/
│       ├── enrich-webhook/index.ts
│       ├── contacts-api/index.ts
│       └── outreach-approve/index.ts
│
├── actors/
│   └── lb-website-scraper/
│       ├── src/main.js
│       ├── .actor/actor.json
│       ├── .actor/input_schema.json
│       └── package.json
│
├── workflows/
│   ├── 01-master-orchestrator.json
│   ├── 02-company-discovery.json
│   ├── 03-scrape-and-extract.json
│   ├── 04-enrichment-orchestrator.json
│   ├── 05-enrichment-waterfall.json
│   ├── 06-output-router.json
│   ├── 07-retry-maintenance.json
│   ├── 08-icp-scorer.json
│   ├── 09-news-triggers.json
│   └── 10-outreach-handoff.json
│
├── prompts/
│   ├── extraction-base.txt
│   ├── extraction-behavioral-health.txt
│   ├── extraction-construction.txt
│   ├── extraction-medical-transport.txt
│   ├── icp-scorer.txt
│   ├── outreach-writer.txt
│   └── trigger-classifier.txt
│
└── scripts/
    ├── import_workflows.sh
    ├── test_pipeline.sh
    ├── seed_data.sql
    └── reset_test_data.sql
```

---

## 13. SMOKE TEST SCRIPT

```bash
#!/bin/bash
# scripts/test_pipeline.sh
set -e
source .env.build

echo "════════════════════════════════════════"
echo "  Lead Booster Pro — Smoke Test v2.0"
echo "════════════════════════════════════════"

COMPANY="ManageAI Test BH"
DOMAIN="test-bh-$(date +%s).manageai.io"
TIMEOUT=360

echo "1️⃣  Intake: $COMPANY / $DOMAIN"
RESP=$(curl -s -X POST \
  "${SUPABASE_URL}/functions/v1/enrich-webhook" \
  -H "x-api-key: ${INTAKE_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{\"company\":\"$COMPANY\",\"domain\":\"$DOMAIN\",\"vertical\":\"behavioral_health\",\"source\":\"smoke_test\"}")
QUEUE_ID=$(echo $RESP | python3 -c "import sys,json;print(json.load(sys.stdin)['queue_id'])")
echo "   Queue ID: $QUEUE_ID"

echo "2️⃣  Polling for completion..."
ELAPSED=0
while [ $ELAPSED -lt $TIMEOUT ]; do
  STATUS=$(curl -s "${SUPABASE_URL}/rest/v1/lb_queue?id=eq.${QUEUE_ID}&select=status" \
    -H "apikey: ${SUPABASE_ANON_KEY}" \
    -H "Authorization: Bearer ${SUPABASE_ANON_KEY}" | python3 -c \
    "import sys,json;d=json.load(sys.stdin);print(d[0]['status'] if d else 'not_found')")
  echo "   [${ELAPSED}s] Status: $STATUS"
  [ "$STATUS" = "complete" ] && break
  [ "$STATUS" = "failed" ] && { echo "❌ FAILED"; exit 1; }
  sleep 15; ELAPSED=$((ELAPSED+15))
done
[ $ELAPSED -ge $TIMEOUT ] && { echo "❌ TIMEOUT after ${TIMEOUT}s"; exit 1; }

echo "3️⃣  Checking contacts..."
COUNT=$(curl -s "${SUPABASE_URL}/rest/v1/lb_contacts?queue_id=eq.${QUEUE_ID}&select=id,icp_score,email_verified" \
  -H "apikey: ${SUPABASE_ANON_KEY}" -H "Authorization: Bearer ${SUPABASE_ANON_KEY}" | python3 -c \
  "import sys,json;d=json.load(sys.stdin);print(len(d))")
echo "   Total contacts: $COUNT"

echo "4️⃣  Checking ICP scores..."
ICP_SCORED=$(curl -s "${SUPABASE_URL}/rest/v1/lb_contacts?queue_id=eq.${QUEUE_ID}&icp_score=gt.0&select=id" \
  -H "apikey: ${SUPABASE_ANON_KEY}" -H "Authorization: Bearer ${SUPABASE_ANON_KEY}" | python3 -c \
  "import sys,json;print(len(json.load(sys.stdin)))")
echo "   ICP-scored contacts: $ICP_SCORED"

echo "5️⃣  Checking outreach drafts..."
DRAFTS=$(curl -s "${SUPABASE_URL}/rest/v1/lb_outreach?queue_id=eq.${QUEUE_ID}&select=id" \
  -H "apikey: ${SUPABASE_ANON_KEY}" -H "Authorization: Bearer ${SUPABASE_ANON_KEY}" | python3 -c \
  "import sys,json;print(len(json.load(sys.stdin)))")
echo "   Outreach drafts generated: $DRAFTS"

echo ""
echo "════════════════════════════════════════"
echo "✅ SMOKE TEST PASSED"
echo "   Queue: $QUEUE_ID"
echo "   Contacts enriched: $COUNT"
echo "   ICP scored: $ICP_SCORED"
echo "   Outreach drafts: $DRAFTS"
echo "════════════════════════════════════════"
```

---

## 14. SEED DATA

```sql
-- scripts/seed_data.sql

INSERT INTO lb_queue (company, domain, vertical, source, priority) VALUES
  ('Sanctuary Recovery Centers', 'sanctuaryrecoverycenters.com', 'behavioral_health', 'seed', 9),
  ('Cholla Behavioral Health', 'chollabh.com', 'behavioral_health', 'seed', 8),
  ('Sunstate Medical Transport', 'sunstatemedical.com', 'medical_transport', 'seed', 9),
  ('Cornerstone General Contractors', 'cornerstonegc.com', 'construction', 'seed', 8),
  ('ManageAI', 'manageai.io', 'general', 'seed', 5);

-- Seed team connections for LinkedIn intelligence
INSERT INTO lb_team_connections (team_member, team_member_linkedin, connection_name, connection_company, degree)
VALUES
  ('Brian Reinhart', 'linkedin.com/in/brianreinhart', '', '', 1),
  ('Tony [Sales]', 'linkedin.com/in/tony-manageai', '', '', 1),
  ('Chad [Product]', 'linkedin.com/in/chad-manageai', '', '', 1);
```

---

## 15. POST-SWARM CHECKLIST

### n8n UI — Add Credentials (Header Auth type)

- [ ] Anthropic (Claude) — API Key header: `x-api-key`
- [ ] Hunter.io — query param: `api_key`
- [ ] Apollo.io — header: `x-api-key`
- [ ] PDL — header: `x-api-key`
- [ ] Snov.io — OAuth2 (client_id + client_secret)
- [ ] RocketReach — header: `api_key`
- [ ] ZeroBounce — query param: `api_key`
- [ ] MillionVerifier — query param: `api`
- [ ] NeverBounce — query param: `api_key`
- [ ] GitHub — header: `Authorization: Bearer`
- [ ] Google Places — query param: `key`
- [ ] FullContact — header: `Authorization: Bearer`
- [ ] Apify — header: `Authorization: Bearer`
- [ ] Instantly.ai — query param: `api_key`

### Assign credentials in n8n

- [ ] WF-03: Claude node → Anthropic credential
- [ ] WF-05: Each enrichment HTTP node → corresponding credential
- [ ] WF-08: Claude node → Anthropic credential
- [ ] WF-10: Claude node → Anthropic credential; Instantly node → Instantly credential

### Database & Deploy

- [ ] Run all 5 migrations in Supabase SQL Editor
- [ ] Deploy Edge Functions: `supabase functions deploy --all`
- [ ] Load seed data: `psql $SUPABASE_DB_URL < scripts/seed_data.sql`
- [ ] Verify ICP definitions loaded: SELECT * FROM lb_icp_definition;

### Validation

- [ ] Run smoke test: `bash scripts/test_pipeline.sh`
- [ ] Manually trigger seed queue for Sanctuary via n8n
- [ ] Verify contacts appear in lb_contacts with icp_score > 0
- [ ] Verify at least one outreach draft in lb_outreach
- [ ] Confirm Dave + Brian both receive completion email
- [ ] Test approval webhook with a draft outreach
- [ ] Test Marla → intake webhook chain (Teams message → POST → queued)

### Go-Live

- [ ] Activate WF-07 (retry/maintenance cron)
- [ ] Activate WF-09 (news monitoring cron)
- [ ] Set WF-01 webhook URL as Marla's "enrich" intent endpoint
- [ ] Update ManageAI AI personas with intake webhook URL

---

*ManageAI · Lead Booster Pro · Build Spec v2.0 · March 2026 · Claude-Native Edition · CONFIDENTIAL*
*All AI calls use Anthropic Claude Sonnet (claude-sonnet-4-20250514) — no OpenAI dependency.*
