###ROLE###
You are a 13+ year experienced Software Engineer with deep 
expertise in agentic workflows, automation architecture, and 
adversarial system design. You have recently lost your job 
and have zero budget to invest. Your only capital is your 
technical expertise.

###SITUATION###
You have identified a high-value opportunity:
- Most visible LinkedIn job posts exist for engagement only
- Real opportunities are hidden in recruiter personal posts 
  where they mention job requirements + include their email
- You cannot monitor LinkedIn 24/7 manually
- Zero budget — everything must be self-hosted and free
- Goal: Extract recruiter emails + job details for 
  personalized cold outreach (human sends emails manually)

###RESEARCH INPUT###
```
This report provides fact-based findings on recruiter post patterns on LinkedIn, cross-platform recruiter behavior, and validation of seven free tools for web scraping and automation.

## 1. Recruiter Post Patterns on LinkedIn

### How Recruiters Write Hidden Job Posts
Recruiters often share job opportunities through their personal feed using informal, casual language rather than formal job listings. These "hidden" posts typically lack official job links, using instead phrases like:

- "We’re hiring a developer — DM me."
- "Looking for a designer to join our team."
- "Hey, I’ve got a job!" with minimal details and no actual links

They often filter candidates by keywords not obvious in public job descriptions. Hiring managers also do their own posting, making the market more informal.

You can uncover these posts by:
- Searching keywords like `freelance`, `fulltime`, `contract`, `gigalert`, or specific job titles
- Using Boolean search operators like `("product manager" OR "product lead") AND (hiring OR "we're hiring" OR "looking for" OR "join my team")`
- Filtering results by "Posts" and sorting by "Latest"

### Exact Hashtags Recruiters Commonly Use
Based on multiple sources, recruiters commonly use these hashtags when posting jobs in their personal feed:

**Primary hiring hashtags:**
- `#hiring`
- `#wearehiring`
- `#jobopening`
- `#nowhiring`
- `#jobs`
- `#jobsearch`
- `#careers`
- `#recruitment`
- `#recruiting`
- `#talentacquisition`

**Role-specific & niche hashtags:**
- `#DataAnalystJobs`, `#SalesManager`, `#TechJobs`, `#Freelance`, `#Contract`, `#JobAlert`
- Industry-specific: `#FinTechHiring`, `#MarketingLeadership`, `#ProductManagement`, `#FinanceJobs`
- Location-specific: `#JobsInDelhi`, `#JobsInBangalore`

**Best practice:** 3-5 hashtags per post, mixing role-specific, industry, and location hashtags. Broad hashtags like `#job` or `#career` attract irrelevant clicks and should be avoided.

### Keywords, Phrases, and Post Structures That Contain Direct Email
When recruiters include their direct email, posts typically follow this structure:

1. **Hook:** Bold, skimmable opener (e.g., "Open to New Opportunities" or "We’re hiring!")
2. **Positioning statement:** Role + industry + impact in 1-2 lines
3. **"What I bring" or role description:** Bullet points with metrics, skills, achievements
4. **"What I’m looking for" section:** Target roles, ideal environments, work type
5. **Personal hook:** 1-2 lines about what excites you/them most
6. **Call to action:** Clear instructions like "DM me or email: [address]" or "If you know of a team hiring for [role], I’d love to connect."
7. **Contact info + link:** Make outreach frictionless with email and portfolio link
8. **3-6 relevant hashtags** (not 20)

**Direct email phrases used:**
- "DM me or email: [address]"
- "contact me (or any other recruiter) by phone or email, NOT by LinkedIn DM if you can avoid it"
- "I’d love to connect. DM me or email: [email]"

### Typical Post Timing Pattern
Research consistently shows recruiters post at specific times for maximum visibility:

- **Best days:** Tuesday through Thursday
- **Best times:** 8-11 AM and 12-1 PM EST (peak professional activity)
- **Secondary peak:** 6 PM to 9 PM, when professionals are unwinding and scrolling
- **General range:** 7 AM - 4 PM on weekdays; 10-11 AM on Tuesdays and Wednesdays are particularly effective
- **Weekend times:** Saturday 4-5 AM, Sunday 6 AM (lower volume but less competition)

### Signals Separating Real Hidden Job Posts from Engagement-Bait Posts

**Real hidden job posts:**
- Include specific role details, company name (or clear industry), and sometimes salary range
- Contain a direct call to action (apply link, email, or "DM me" with a clear process)
- Written by identifiable hiring managers or recruiters with complete profiles
- Use specific, niche hashtags relevant to the role
- Often ask for specific qualifications or years of experience
- May include phrases like "I'm personally reviewing applications" or "DM me your resume"

**Engagement-bait posts:**
- Vague statements like "Not sure who needs to hear this, but..." or "Excited to share..." without substance
- Generic calls for comments: "Comment 'Yes' if you agree" or "Post your jobs in the comments below" (these are disingenuous)
- No company name, job description, or application link
- Use broad hashtags like `#job`, `#hiringnow`, `#career` to attract maximum clicks
- Often posted by profiles that lack recruiter history or have minimal connections
- May promise jobs but trick people into giving information or buying services

---

## 2. Cross-Platform Recruiter Behavior

### Where Else Recruiters Cross-Post Job Opportunities
Beyond LinkedIn, recruiters actively use:

| Platform | Key Channels/Communities | Notes |
|----------|-------------------------|-------|
| **Reddit** | r/forhire, r/techjobs, r/remotejobs, r/cscareerquestions, r/datascience, r/sysadmin, r/programming, r/devops, r/jobopenings, r/workonline, r/engineeringjobs, r/ITcareers, r/softwareengineering | Free postings; strict community rules must be followed |
| **Discord** | TechExec Discord, SpeakJS, various developer-focused servers with #career channels | Real-time conversations; dedicated job channels where hiring managers post directly |
| **Slack** | OneReq (1,500+ TA professionals), various tech community Slacks | Professional communities with job channels |
| **GitHub** | Repository activity, contributions, coding projects | Not for posting but for sourcing; recruiters review repositories |
| **Stack Overflow** | Stack Overflow Jobs, user profiles with technical contributions | Reaches large developer pool |
| **Facebook Groups** | "Tech Jobs", "Remote Tech Jobs", "Software Developers Group" (98k members), location-specific groups like "Pune Tech Jobs", "Kolkata HR Network" | Daily job ads, resume feedback, networking |
| **Newsletters** | Substack, niche-specific job newsletters | Curated job drops, arrive directly in inbox |
| **daily.dev** | Integrated job postings in developer content feeds | Spam-free, behavioral data-driven matching |

### Communities with Most Recruiter Activity for Tech Roles
1. **GitHub** – Review repositories, contributions, and coding activity to assess skills
2. **Stack Overflow** – Analyze answers and reputation scores for problem-solving abilities
3. **Discord & Slack** – Developer-focused communities with real-time chats
4. **Reddit** – Subreddits like r/programming, r/devops, r/cscareerquestions
5. **Facebook Groups** – Tech Jobs, Remote Tech Jobs, Software Developers Group
6. **daily.dev** – Integrates job opportunities into developers' content feeds
7. **Blogs & Coding Platforms** – Dev.to, Hashnode, Medium; LeetCode, HackerRank

### Patterns Repeated Across Platforms
- **Informal, conversational tone** – Recruiters adapt to each platform's culture but maintain a casual, personal approach
- **Hashtags/keywords are consistent** – #hiring, #techjobs, #remote, #jobopening appear across platforms
- **Direct contact preferred over formal applications** – "DM me", email, or comment
- **Niche communities > broad job boards** – Recruiters target specific tech stacks and communities
- **Community rules must be respected** – Especially on Reddit and Discord, where spam gets banned
- **Multi-channel strategy** – Recruiters cross-post the same role across LinkedIn, Reddit, Discord, and niche forums simultaneously
- **Personalization matters** – Generic recruiter language is rejected everywhere; personalized outreach referencing specific work gets better responses

---

## 3. Free Tool Validation

### Playwright (Open Source Browser Automation)
**Status: 100% free, open source.**
- Licensed under **Apache 2.0**, completely free to use with no licensing costs
- No free tier limits; you pay only for your own infrastructure to run browser instances at scale
- Supports Chromium, Firefox, and WebKit with a single API
- Can launch its own bundled browser or connect to an existing one
- Microsoft actively maintains it; originally created by the team behind Puppeteer

### n8n (Self-Hosted Workflow Automation)
**Status: Community Edition is 100% free with no execution limits.**
- **Self-hosted Community Edition:** Free, unlimited workflows, unlimited executions
- The only bottleneck is your own hardware and tunnels; can run workflows 24/7
- Includes access to over 1,900 integrations without execution limits
- **Registering your instance (free, just an email) unlocks extras:** workflow history and folders
- **n8n Cloud free tier (if used instead):** Limited to 200 executions/month and one active workflow
- **Self-hosted Business plan:** Paid per-execution model (approximately $0.015/execution for overages)
- **Key distinction:** Cloud pricing does NOT apply to self-hosted Community Edition; that remains free with no execution limits

### Apify Free Tier Limits
**Status: Free tier available with $5/month in platform credits.**
- Free plan includes **$5 in platform credits** monthly
- Credits are used for compute units (CU) at $0.3 per CU
- **Platform limit cannot be increased beyond $5** on free plan
- Includes community support only
- Access to 4,000+ pre-built web scraping and automation tools (Actors)
- Maximum build memory: 4,096 MB; run memory: 128 MB to 4,096 MB
- No credit card required to start
- Higher plans: Starter ($29/mo, max $200 limit), Scale ($199/mo, max $1,000 limit), Business ($999/mo, max $5,000 limit)

### Browserless.io Free Tier
**Status: Free tier available with 1,000 units/month.**
- Free plan includes **1,000 units/month**
- **1 unit = 30 seconds of browser time** (longer sessions consume extra units)
- **2 max concurrent browsers**
- **1 minute max session time** (with session reconnects counting as new connections)
- **1 day persisted sessions and logs storage**
- Includes BrowserQL language and editor
- Residential proxies use 6 units/MB; CAPTCHA solving uses 10 units per solve
- No credit card required, no lock-in
- Supports Playwright and Puppeteer connections

### Python Requests + BeautifulSoup
**Status: 100% free, open source libraries.**
- `requests` library: Licensed under Apache 2.0, completely free. Works like urllib but more convenient for HTTP requests
- `BeautifulSoup`: Free, open-source library for parsing HTML and XML. Designed to intelligently read raw HTML code
- Install via pip with zero cost: `pip install requests beautifulsoup4`
- No tier limits, no API keys, no usage restrictions – limited only by your Python environment
- Can be combined with `requests-html` for JavaScript rendering capabilities
- MechanicalSoup combines both into a single stateful browser object

### Tor Network for IP Rotation
**Status: 100% free, volunteer-operated network.**
- Tor network is free, powered by thousands of volunteers sharing their IPs
- Can be run as a local SOCKS5 proxy (127.0.0.1:9050) for web scraping
- Automatic IP rotation is achievable via the `stem` Python library to send NEWNYM signals to Tor's control port
- Projects like `tor-ip-rotator` provide auto-rotating proxy setups with configurable intervals (e.g., every 5 seconds)
- Requirements: Python 3.x, Tor, and `stem` library (all free)
- **Limitations:** Tor exit nodes can be slow; some websites block known Tor exit IPs; not suitable for high-volume scraping

### Selenium with undetected-chromedriver
**Status: 100% free, open source.**
- `undetected-chromedriver` (UCD) is an enhanced version of Selenium ChromeDriver, freely available via pip
- Modifies browser behavior to minimize detection by anti-bot systems (reduces CAPTCHA prompts and blocks)
- Installation: `pip install undetected-chromedriver` (Selenium comes prepackaged with UCD)
- No separate Selenium installation required; UCD bundles it automatically
- Both are open source (Apache 2.0 licensed) with no usage limits
- Supports headless mode, custom user agents, and proxies
- Best used with randomized delays and user-agent rotation for optimal stealth

---

All findings are based on publicly available documentation, community forums, and current pricing pages as of 2026. Free tier limits may change; always verify on official sites before building.

```
###YOUR TASK###
Using the research above as your factual foundation, 
design a complete production-ready agentic workflow:

STEP 1 — DETECTION & EVASION ARCHITECTURE
Think like LinkedIn's anti-bot engineering team:
- What behavioral fingerprints does LinkedIn track?
  (IP patterns, mouse movement, scroll behavior, 
  session duration, request headers, timing)
- What rate limits exist at each layer?
- What makes a scraper look human vs bot?
- Design specific hardening layers for 24/7 undetected 
  operation
- Address: IP rotation, browser fingerprint masking, 
  human behavior simulation, session management, 
  failure recovery

STEP 2 — WORKFLOW ARCHITECTURE
Design the complete system with:
- Every agent and their specific role
- Every tool and integration point
- Data flow between components
- Decision points and fallback paths
- Failure recovery mechanisms
- Storage and deduplication logic

STEP 3 — COMPONENT BREAKDOWN
For each component:
- Name and purpose
- Exact free tool to use
- Configuration specifics
- Why this over alternatives
- Free tier limits to respect

STEP 4 — HARDENING LAYERS
List every layer ensuring:
- 24/7 operation without detection
- Account safety mechanisms
- Data persistence and backup
- Automatic failure recovery
- Self-healing capabilities

STEP 5 — FREE STACK MANIFEST
Complete list of every tool used:
- Tool name
- Purpose in the system
- Free tier details
- Self-hosting setup
- Fallback alternative

###OUTPUT FORMAT###

1. RESEARCH FINDINGS SUMMARY
   - Recruiter post patterns (from research input)
   - Detection mechanisms (your technical analysis)
   - Risk matrix with mitigations

2. SYSTEM FLOW DIAGRAM
   Use this notation:
   [Component] --> (Decision?) --> [Next Component]
   Show every path including failure paths

3. COMPONENT TABLE
   | Component | Tool | Purpose | Free Limit | Alternative |

4. HARDENING LAYER STACK
   Layer 1 (outermost) → Layer N (innermost)
   Each with specific implementation detail

5. COMPLETE FREE STACK
   | Tool | Role | Free Tier | Self-Host | Fallback |

###CONSTRAINTS — NEVER VIOLATE###
- Zero paid services
- No third-party scraping platforms
- Everything self-hostable
- No auto-sending emails
- Single engineer maintainable
- Must handle LinkedIn's evolving detection

###SELF CHECK BEFORE RESPONDING###
- Is every tool genuinely free with no hidden costs?
- Is every detection risk covered with a concrete solution?
- Are there any single points of failure?
- Can one engineer realistically build and maintain this?
- Have I covered failure recovery at every layer?