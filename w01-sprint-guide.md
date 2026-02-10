# WEEK 1 SPRINT GUIDE
## February 10-14, 2026

---

## MISSION

Build a working automation system that demonstrates:
1. Infrastructure can run reliably 24/7
2. Real estate data flows from source to actionable output
3. Content can be published automatically
4. You can work together effectively

**The Core Principle: Links over Reports**

Every daily update must include an artifact link:
- GitHub PR or commit
- n8n workflow URL or export
- Deployed URL
- Excel file
- Published content

If there's no link, it didn't ship.

---

## THE P0/P1/P2 FRAMEWORK

### P0 - MUST COMPLETE
**Core infrastructure + basic functionality**

This is the foundation. Everything else builds on this.

**Requirements:**
- ✅ n8n + Postgres on AWS with HTTPS + authentication + persistence
- ✅ ONE real estate source scraped successfully
- ✅ Database stores leads with timestamps
- ✅ Delta detection (NEW leads vs. already seen)
- ✅ Excel export generates a working file
- ✅ Alerts configured (workflow failure + summary)

---

### P1 - CHOOSE ONE
**Add either scale OR content automation**

By Monday 10am, decide together which makes more sense:

**Option A: Content Pipeline**
- Markdown file → Beehiiv draft automation
- Header image auto-generated (consistent style)
- X + LinkedIn post stubs created
- 2 Juncture Policy articles published

**Option B: Second Real Estate Source**
- Two sources feeding unified schema
- Basic deduplication (same address = one row)
- Excel shows which sources found each lead
- Unified export with source tracking

Choose based on:
- What has fewer blockers
- What you're more excited to build
- What Walter needs first

---

### P2 - STRETCH GOALS
**Advanced features - only if ahead of schedule**

Attempt these only if P0 + P1 are done by Thursday noon:

**Advanced Lead Management:**
- Multi-tab Excel (New / Updated / Bad_Rows / Top_25)
- QA flags (BAD/REVIEW/OK with reasons)
- Priority scoring with explanations
- Changed_fields showing old/new values

**Full Stack:**
- BOTH content pipeline AND second source working
- Complete automation end-to-end

**Reality check:** P2 is ambitious. Focus on P0+P1 quality over P2 quantity.

---

## DAILY STANDUP FORMAT

**Time:** 8:30am sharp
**Duration:** 15 minutes maximum

Post in Slack using this exact format:

```
CHRISTIAN:
✅ SHIPPED: [link or "nothing"]
🎯 SHIPPING TODAY: [one specific P0/P1/P2 item]
🚧 BLOCKED: [what you need or "nothing"]
⏭️ DEPRIORITIZING: [what you're pushing to next week]

KELVIN:
[same format]
```

**The DEPRIORITIZING line matters.** It shows your thinking when managing multiple priorities.

---

## THE 30-MINUTE RULE

If you're stuck on anything for more than 30 minutes:

1. **Document what you tried:**
   - What you attempted
   - Error messages
   - What you'll try next

2. **Post in Slack:**
   - Screenshots or logs
   - Tag relevant person

3. **Switch to a different task immediately:**
   - Don't wait for a reply
   - Keep shipping other items

**Remember:** Blockers are normal. Silent blockers waste time.

---

## KELVIN'S TASK BREAKDOWN

### Your Role: Infrastructure Engineer

You own: Deployment, scraping, data pipelines, automation.

---

### MONDAY (Day 1) - 6 hours
**Objective: Prove the Stack Works**

#### Task 1: Deploy n8n + Postgres (4 hours)

**Follow this guide:**
https://docs.n8n.io/hosting/installation/docker/

**What "done" looks like:**
- n8n accessible at https://ops.nikointernational.com (or subdomain)
- Password-protected (not open to internet)
- PostgreSQL running in Docker
- Both services survive EC2 reboot

**Deliverables:**
1. **Loom video (2 min)** showing:
   - n8n login screen
   - One test workflow running
   - SQL query: `SELECT version();` proving Postgres works
2. **n8n URL** shared in Slack

**Key resources:**
- n8n Docker: https://docs.n8n.io/hosting/installation/docker/
- Let's Encrypt + Caddy: https://caddyserver.com/docs/automatic-https

---

#### Task 2: GitHub Setup (1 hour)

**Create repository structure:**
- Repository name: `kaya-automation` (or similar)
- README with basic setup instructions
- .gitignore (never commit secrets)
- /workflows folder for n8n exports

**Deliverable:** GitHub repo link in Slack

---

#### Task 3: Persistence Test (1 hour)

**Test procedure:**
1. Create test table:
```sql
CREATE TABLE leads (
  id SERIAL PRIMARY KEY,
  address TEXT,
  price NUMERIC,
  url TEXT,
  first_seen TIMESTAMP DEFAULT NOW()
);
```
2. Insert 1 test row
3. **Reboot EC2 instance**
4. Query table - row should still exist

**Deliverable:** Screenshot of query result after reboot

---

### TUESDAY (Day 2) - 6 hours
**Objective: Build the Scraper**

#### Step 1: Choose Your Source (30 min)

**Work with Christian to pick ONE source based on:**
- Publicly available (legal to scrape)
- No CAPTCHA or aggressive blocking
- Has: address, price, status, URL

**Good options (easiest first):**
1. Local Peru real estate classifieds
2. Public FSBO listing sites
3. County/municipality public records
4. Zillow (can be blocked - use carefully)

**Deliverable:** Post in Slack: "Using [source] because [reason]"

---

#### Step 2: Build Scraper Workflow (4 hours)

**n8n workflow requirements:**
- Scrape 20-50 listings
- Extract: address, price, status, url
- Save to `leads` table in Postgres
- Handle missing fields gracefully (don't crash)

**Database schema:**
```sql
CREATE TABLE leads (
  id SERIAL PRIMARY KEY,
  source TEXT,
  address_raw TEXT,
  address_normalized TEXT,
  price NUMERIC,
  status TEXT,
  url TEXT,
  first_seen TIMESTAMP DEFAULT NOW(),
  last_seen TIMESTAMP DEFAULT NOW()
);
```

**Deliverables:**
1. Exported workflow JSON (upload to GitHub)
2. Screenshot showing 20+ rows in database

**Key resources:**
- HTTP Request node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/
- Postgres node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.postgres/

---

#### Step 3: Schedule It (1.5 hours)

**Setup:**
- Workflow runs every 2 hours (or manual for testing)
- Updates `last_seen` timestamp each run

**Deliverable:** Screenshot of workflow execution history

---

### WEDNESDAY (Day 3) - 5 hours
**Objective: Add Delta Detection**

**Implement NEW Lead Detection**

**Logic:**
1. Before inserting, check if `address_normalized` already exists
2. If EXISTS → UPDATE `last_seen` timestamp only
3. If NOT EXISTS → INSERT as NEW lead

**Algorithm (pseudocode):**
```
For each scraped listing:
  normalize_address(listing.address)
  
  existing = SELECT * FROM leads 
             WHERE address_normalized = normalized_address
  
  if existing:
    UPDATE leads SET last_seen = NOW() 
    WHERE id = existing.id
  else:
    INSERT INTO leads (address_raw, address_normalized, ..., first_seen)
```

**Test procedure:**
1. Run workflow (should find ~20 NEW)
2. Run again immediately (should find 0 NEW)
3. Manually change 1 price in source
4. Run again (should detect change - for P2)

**Deliverables:**
1. Updated workflow JSON
2. Screenshot showing before/after run counts
3. 1-paragraph explanation of your logic

**Key resources:**
- Merge node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.merge/
- IF node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/

---

### THURSDAY (Day 4) - 6 hours
**Objective: Excel Export + Alerts**

#### Task 1: Excel Generator (4 hours)

**Use:** n8n Spreadsheet File node or Google Sheets

**Excel requirements (P0 version):**
- One tab: "New_Leads"
- Columns from Christian's schema:
  - address
  - price
  - status
  - url
  - first_seen
  - (other columns Christian defined)
- Only include leads where `first_seen = today`

**Deliverables:**
1. Real Excel file with 10-20 NEW leads
2. Saved to Google Drive (share link) OR downloadable from n8n

**Key resources:**
- Google Sheets: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/
- Spreadsheet File: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.spreadsheetfile/

---

#### Task 2: Alert System (2 hours)

**Setup two alerts:**

1. **Failure Alert:**
   - Trigger when workflow fails
   - Send to Slack or email
   - Include: workflow name, error message, timestamp

2. **Summary Alert:**
   - Send after each successful run
   - Format: "🏠 Found X NEW leads | Total in DB: Y | Last run: [time]"

**Test:** Intentionally break workflow, verify alert fires

**Deliverables:** Screenshots of both alerts working

**Key resources:**
- Slack node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/
- Email node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.emailsend/

---

### FRIDAY (Day 5) - 4 hours
**Objective: Documentation + Demo Prep**

#### Task 1: Write README (2 hours)

**README must include:**
1. How to start the system (`docker-compose up`)
2. How to access n8n (URL + where to find credentials)
3. How to trigger workflows manually
4. Where outputs are saved
5. What to do if something breaks

**Write for:** Non-technical business owner (Walter)

**Deliverable:** README.md in GitHub repo

---

#### Task 2: Backup Job (1 hour)

**Simple backup script:**
```bash
#!/bin/bash
docker exec postgres pg_dump -U n8n > backup_$(date +%Y%m%d).sql
```

**Requirements:**
- Script runs successfully
- Creates .sql file with data
- (Optional) Schedule via cron for nightly backups

**Deliverables:** 
- Backup script in repo
- Sample backup file proving it works

---

#### Task 3: Demo Prep (1 hour)

**Practice demonstrating:**
1. Login to n8n
2. Show workflow diagram
3. Manually trigger scraper
4. Show database filling with results
5. Download Excel file
6. Show both alerts working

**Be ready for 4pm Friday demo**

---

### P1 TASKS - If Choosing "Second Source"

#### Wednesday PM (4 hours)
- Build Source 2 scraper workflow
- Normalize into same schema as Source 1
- Test both sources run independently

#### Thursday AM (3 hours)
- Implement dedupe: matching `address_normalized`
- Add `sources_seen` column (e.g., "zillow,fsbo")
- Excel shows unified deduplicated list

**Deliverable:** Excel showing leads from both sources

---

### P2 TASKS - If Ahead of Schedule

#### Thursday-Friday (8+ hours)
- Implement hash-based change detection
- Track which fields changed (`changed_fields` column)
- Store old/new values for price, status, auction_date
- Generate multi-tab Excel:
  - New (first_seen = today)
  - Updated (changed since last run)
  - Bad_Rows (QA failures)
  - Top_25 (highest priority)
- Implement Christian's QA flags
- Implement priority scoring

**Deliverable:** Multi-tab Excel meeting specs

---

## CHRISTIAN'S TASK BREAKDOWN

### Your Role: Systems Architect

You define WHAT gets built. Kelvin implements HOW.

---

### MONDAY (Day 1) - 4 hours
**Objective: Define the Schema**

#### Task 1: Excel Schema Specification (2 hours)

**Create Google Sheet with two tabs:**

**Tab 1: "Required_Columns"**

| Column Name | Data Type | Required? | Example | Notes |
|-------------|-----------|-----------|---------|-------|
| address | Text | Yes | "123 Main St, Lima, OH 45805" | Full address |
| price | Number | Yes | 150000 | USD, no commas |
| status | Text | Yes | "FSBO" | FSBO, Pre-foreclosure, Auction, Active |
| url | URL | Yes | https://... | Link to listing |
| first_seen | Timestamp | Yes | 2026-02-10 09:30:00 | When discovered |
| last_seen | Timestamp | Yes | 2026-02-10 11:30:00 | Last seen |
| phone | Text | No | +1-555-0100 | If available |
| email | Email | No | seller@example.com | If available |
| source | Text | Yes | "zillow" | Source identifier |

**Tab 2: "Optional_Future_Columns"** (for P2)
- auction_date
- equity_percent
- motivation_score

**Deliverable:** Share Google Sheet with Kelvin by 11am Monday

---

#### Task 2: Help Choose Source (1 hour)

**Work with Kelvin to pick easiest source:**
- Research 2-3 options
- Check for anti-bot protections
- Recommend one with justification

**Deliverable:** 1-paragraph recommendation in Slack

---

#### Task 3: Define Success Criteria (1 hour)

**Answer these questions in a doc:**
1. How many leads make Friday successful? (Suggest: 20 minimum)
2. What makes a lead "good enough" to call?
3. What data quality issues should we auto-flag?

**Deliverable:** "Week 1 Success Criteria" doc (1 page)

---

### TUESDAY (Day 2) - 4 hours
**Objective: QA Rules + Scoring**

#### Task 1: Define QA Flags (2 hours)

**Create document with these rules:**

**QA_FLAG = BAD** (exclude from caller list):
1. Missing address (blank or too short)
2. Invalid address format (no numbers, gibberish)
3. Missing URL (can't verify)
4. Duplicate (same address + source + date)

**QA_FLAG = REVIEW** (visible but don't prioritize):
5. Missing contact (no phone AND no email)
6. Price = 0 or > $5M (outlier)
7. Status = "Sold" or "Pending" (too late)

**QA_FLAG = OK:**
Everything else

**Deliverable:** QA Rules v1 document

---

#### Task 2: Priority Scoring (2 hours)

**Create simple point system:**

**High Value Signals:**
- Status = "Pre-foreclosure": +40 points
- Status = "FSBO": +30 points
- Has phone number: +15 points
- Has email: +10 points
- Price < $200k: +10 points

**Deductions:**
- QA_FLAG = REVIEW: -50 points
- Missing contact: -25 points

**Priority Reasons (templates):**
- "Pre-foreclosure property with contact info"
- "FSBO with phone number available"
- "Active listing under $200k"

**Deliverable:** Scoring Rules v1 document

---

### WEDNESDAY (Day 3) - 3 hours
**Objective: Data Validation**

**Review Kelvin's Output**

Once Kelvin has 20+ leads in database:

1. **Export CSV** of all leads
2. **Manual audit:**
   - Are addresses real/valid?
   - Are prices reasonable?
   - Do URLs work?
   - Any obvious duplicates?
3. **Create tickets** for Kelvin:
   - "Fix: Some addresses missing city"
   - "Fix: Prices showing as text not numbers"

**Deliverables:**
1. Audit notes doc (5-10 issues found)
2. 3-5 GitHub issues for Kelvin

---

### THURSDAY (Day 4) - 4 hours
**Objective: Excel Testing + Refinement**

#### Task 1: Test Excel Output (2 hours)

**When Kelvin generates first Excel:**

1. **Open it as if you're a caller**
2. **Answer these questions:**
   - Can I find address easily?
   - Do I know what price to offer?
   - Do I have contact info?
   - Is URL clickable?
3. **Rate it:** "Usable as-is" or "Needs fixes"

**Create tickets for any issues**

**Deliverable:** Excel Review doc + tickets

---

#### Task 2: Refine Scoring (2 hours)

**Based on real data:**
- Are scores distributing well? (Range 0-100)
- Do priority_reasons make sense?
- Adjust thresholds if needed

**Deliverable:** Scoring Rules v2

---

### FRIDAY (Day 5) - 4 hours
**Objective: Documentation**

**Create Caller Playbook (4 hours)**

**Write 3-page guide:**

**Page 1: How to Use the Excel File**
- Which tab to look at first
- What each column means
- How to interpret priority scores

**Page 2: What Makes a Good Lead**
- Red flags to avoid
- Green flags to prioritize
- When to skip to next lead

**Page 3: Troubleshooting**
- What if phone is disconnected?
- What if URL is broken?
- Who to ask for help

**Deliverable:** Caller Playbook PDF

---

### P1 TASKS - If Choosing "Content Pipeline"

#### Wednesday (4 hours)

**Task 1: Markdown Format Standard (2 hours)**

**Create template:**
```yaml
---
site: juncturepolicy
title: "Article Title Here"
author: "Walter Guevara"
date: "2026-02-12"
tags: ["ai policy", "latam"]
image_style: "think_tank"
---

Article body in markdown...
```

**Create 3 sample files** from existing Juncture articles

**Deliverables:**
1. Content SOP v1 doc
2. 3 sample .md files in Google Drive

---

**Task 2: Image Prompt Template (2 hours)**

**Write prompt for consistent header images:**

```
Create a professional think-tank style header image for:
Title: {title}
Theme: {tags}

Requirements:
- Editorial/modern aesthetic
- High credibility look
- No text, faces, or logos
- Abstract shapes, maps, or symbolic elements
- 1200x630px dimensions
- Suitable for LinkedIn/X sharing
```

**Test with 2-3 examples** (use ChatGPT/DALL-E)

**Deliverables:**
- Image style guide
- 3 sample images

---

#### Thursday (4 hours)

**Task: Test Content Workflow**

1. Drop sample .md file in Drive trigger folder
2. **Verify outputs:**
   - Beehiiv draft created
   - Header image generated
   - X thread stub created
   - LinkedIn post stub created
3. **Review quality:**
   - Is image consistent with brand?
   - Are stubs usable?
   - Is Beehiiv formatting correct?
4. Create tickets for issues

**Deliverable:** Content QA report

---

#### Friday (4 hours)

**Task: Publish 2 Articles**

1. Select 2 existing Juncture articles
2. Run through pipeline
3. Manual review/edit if needed
4. **Publish to:**
   - Juncture Policy website
   - Beehiiv newsletter (or draft)
   - X (can be just stub for approval)
   - LinkedIn (can be just stub for approval)

**Deliverables:**
1. 2 live published articles (links)
2. Publishing SOP v2

---

### P2 TASKS - If Ahead

**Thursday-Friday:**
- Manual audit of 100 leads
- Test Top_25 tab usability
- Create operational runbook
- Document edge cases

---

## CRITICAL RESOURCES

### n8n Documentation
- Main docs: https://docs.n8n.io/
- Docker setup: https://docs.n8n.io/hosting/installation/docker/
- Workflows: https://docs.n8n.io/workflows/
- Node library: https://docs.n8n.io/integrations/builtin/

### Infrastructure
- Let's Encrypt: https://letsencrypt.org/
- Caddy server: https://caddyserver.com/
- Docker Compose: https://docs.docker.com/compose/

### APIs
- Beehiiv API: https://www.beehiiv.com/developers
- OpenAI API: https://platform.openai.com/docs/
- Google Sheets API: https://developers.google.com/sheets/api
- Slack Webhooks: https://api.slack.com/messaging/webhooks

### Translation
- DeepL: https://www.deepl.com/translator
- Use for sections you don't understand
- Keep code/column names in English

---

## FRIDAY DEMO CHECKLIST (4pm)

### Kelvin's Demo
- [ ] Login to n8n
- [ ] Show workflow running
- [ ] Trigger scraper manually
- [ ] Query database showing results
- [ ] Download/open Excel file
- [ ] Demonstrate alerts firing

### Christian's Demo
- [ ] Walk through schema decisions
- [ ] Show QA rules logic
- [ ] Explain scoring system
- [ ] Demonstrate Excel usability
- [ ] (If P1 content) Show published articles

### Together
- [ ] Explain what you'd do differently next week
- [ ] Identify 3 improvements for Week 2

---

## QUESTIONS?

**Before Monday:**
- Review this entire document
- Decide together: P1 = Content or Second Source?
- Post decision in Slack by Monday 9am

**During the week:**
- Ask questions in Slack (don't DM unless urgent)
- Tag @Walter for business context
- Tag each other for technical help

**Good luck - ship artifacts!** 🚀

