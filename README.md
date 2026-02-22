# 🤖 n8n Workflow Collection

A collection of production-ready **n8n automation workflows** for lead management, AI-powered content creation, job hunting, and more.

---

## 📋 Workflows Overview

| # | Workflow | Trigger | Key Integrations |
|---|---------|---------|-----------------|
| 01 | [Lead Intake → Validation → Routing → CRM Upsert](#01-lead-intake--validation--routing--crm-upsert) | Webhook (POST) | Google Sheets |
| 03a | [AI Email Autoresponder with Approval](#03a-ai-email-autoresponder-with-approval) | Email (IMAP) | OpenAI, Qdrant, Gmail, SMTP |
| 03b | [Qdrant Knowledge Base Upsert](#03b-qdrant-knowledge-base-upsert) | Manual | Qdrant, OpenAI Embeddings |
| 04 | [YouTube Automation](#04-youtube-automation) | Manual / Schedule | Reddit, Supabase, YouTube, OpenAI |
| 05 | [LinkedIn Automation From X Posts](#05-linkedin-automation-from-x-posts) | Schedule (Hourly) | X (Twitter), OpenAI, Gmail, LinkedIn |
| 06 | [Job Application Assistant (India)](#06-job-application-assistant-india) | Schedule | JSearch API, OpenAI, Gmail, Google Sheets |
| 07 | [LinkedIn Job Lead Finder](#07-linkedin-job-lead-finder) | Chat Trigger | Apify, LinkFinder AI, Google Sheets |
| — | [AI Receptionist With UI](#ai-receptionist-with-ui) | Full-stack App | Next.js, Convex, Vapi, n8n |

---

## 01. Lead Intake → Validation → Routing → CRM Upsert

**File:** `01_Lead Intake → Validation → Routing → CRM Upsert (1).json`

**Purpose:** Receives incoming leads via a webhook, validates the data (email format, phone existence), routes them by interest to the appropriate team, and upserts into a Google Sheets CRM.

**Flow:**
```
Webhook (POST /lead-intake)
  → Validate Lead (email + phone)
    → Route Lead by Interest
      ├── AI Automation → Set priority: high, owner: AI Sales Team
      ├── Web Development → Set priority: medium, owner: Web Sales Team
      └── Others → Set priority: low, owner: General
    → Merge → Find Lead by Email
      → Lead Exists?
        ├── Yes → Update Lead
        └── No → Create Lead
      → Respond to Webhook
```

**Setup:**
- Configure the **Google Sheets** credential with your sheet
- The webhook endpoint is `POST /lead-intake`
- Send JSON body with: `name`, `email`, `phone`, `interest`, `source`

---

## 03a. AI Email Autoresponder with Approval

**File:** `03_AI Email processing autoresponder with approval (Yes_No).json`

**Purpose:** Monitors an email inbox via IMAP, summarizes incoming emails using GPT, drafts a professional reply using RAG (Qdrant knowledge base), and sends the draft for human approval before sending.

**Flow:**
```
Email Trigger (IMAP)
  → Convert HTML to Markdown
    → Summarize Email (GPT-4.1)
      → Set Email Content
        → AI Agent: Write Reply (GPT-4o-mini + Qdrant RAG)
          → Send Draft for Approval (Gmail)
            → Approved?
              ├── Yes → Send Email (SMTP)
              └── No → Loop back to rewrite
```

**Setup:**
- Configure **IMAP** credentials for incoming email monitoring
- Configure **SMTP** credentials for sending replies
- Configure **Gmail OAuth2** for approval emails
- Configure **OpenAI API** credentials
- Configure **Qdrant** credentials (requires the `company_knowledge` collection — see workflow 03b)
- Set the approval email address in the "Send Draft" node

---

## 03b. Qdrant Knowledge Base Upsert

**File:** `03_Qdrant_Upsert.json`

**Purpose:** Populates the Qdrant vector database with company knowledge used by the AI Email Autoresponder (03a). Contains company profile, services, pricing, operating hours, contact info, and refund policy.

**Flow:**
```
Manual Trigger
  → Set Demo Data (company knowledge text)
    → Qdrant Upsert (collection: company_knowledge)
      ├── OpenAI Embeddings
      └── Default Data Loader + Recursive Text Splitter
```

**Setup:**
- Configure **Qdrant** credentials
- Configure **OpenAI API** credentials for embeddings
- Modify the text in "Set Demo Data" with your actual company information
- Run manually once to populate the knowledge base

---

## 04. YouTube Automation

**File:** `04_Youtube_Automation.json`

**Purpose:** End-to-end YouTube content pipeline — scrapes Reddit revenge stories, classifies them with AI, creates fictional characters, generates narrated videos, and uploads to YouTube.

**Flow:**
```
PART 1: Story Ingestion
  Manual Trigger
    → Fetch top Reddit posts (/r/revengestories)
      → Filter (text posts only)
        → Loop: Check for duplicates in Supabase
          → Classify (story / advice / rant)
            ├── Story → Save to Supabase (status: queued)
            └── Not a story → Skip

PART 2: Video Creation
  Manual Trigger
    → Get next queued story from Supabase
      → Create character (name, age, location, sex)
        → Rewrite story with AI
          → Clean up text
            → Send to video API
              → Poll video status
                ├── Completed → Download → Upload to YouTube
                ├── Processing → Wait 10s → Poll again
                └── Failed → Set status: failed
```

**Setup:**
- Configure **Supabase** credentials and create `revenge_stories` table with columns: `reddit_id`, `title`, `content`, `permalink`, `status`
- Configure **OpenAI API** credentials
- Configure **YouTube** credentials for uploads
- Set `server_url` in the "Configuration for story writing" node (video generation API)
- Set character image URLs and background video URL

---

## 05. LinkedIn Automation From X Posts

**File:** `05_LinkedIn_Automation_From_X_Posts.json`

**Purpose:** Searches for trending automation-related tweets on X (Twitter), repurposes them into professional LinkedIn posts using AI, sends for email approval, and publishes approved posts to LinkedIn.

**Flow:**
```
Schedule Trigger (Every Hour)
  → Search X for automation tweets
    → Set Tweet Fields
      → Filter short tweets (< 50 chars)
        → AI Agent: Repurpose for LinkedIn (GPT-4o-mini)
          → Set LinkedIn Fields
            → Send Draft for Approval (Gmail)
              → Approved?
                ├── Yes → Post to LinkedIn → Log to Google Sheet
                └── No → Discard
```

**Setup:**
- Configure **X (Twitter) OAuth2** credentials
- Configure **OpenAI API** credentials
- Configure **Gmail OAuth2** credentials
- Configure **LinkedIn OAuth2** credentials
- Create a **Google Sheet** with columns: `tweet_id`, `author`, `original_tweet_url`, `linkedin_post`, `status`, `posted_at`
- Set your LinkedIn person URN in the "Post to LinkedIn" node
- Replace `YOUR_GOOGLE_SHEET_ID` in the "Log to Google Sheet" node

---

## 06. Job Application Assistant (India)

**File:** `06_Job_Application_Assistant_India.json`

**Purpose:** Automated job hunting assistant for the Indian job market. Scrapes your resume, generates a search strategy, finds matching jobs via the JSearch API, selects the best matches, generates cover letters, and applies on your behalf.

**Flow:**
```
Schedule Trigger
  → Load Configuration (location, salary, API keys)
    → Scrape Resume (Jina AI)
      → AI Agent: Generate Profile & Search Strategy
        → AI Agent: Generate JSearch URLs (4-10 URLs)
          → Loop: Fetch jobs from JSearch API
            → Aggregate all jobs
              → Check Google Sheets for already-processed jobs
                → Remove duplicates
                  → AI Agent: Select best matches
                    → Loop per selected job:
                      → AI Agent: Generate cover letter
                        → Send via Gmail
                          → Log to Google Sheets
```

**Setup:**
- Configure **OpenAI API** credentials
- Configure **Google Sheets OAuth2** credentials
- Configure **Gmail OAuth2** credentials
- Get a **JSearch API** key from RapidAPI
- Get a **Jina AI** API key for resume scraping
- Update the Configuration node with:
  - Your resume URL
  - Target location, salary range, and remote preference
  - API keys

---

## 07. LinkedIn Job Lead Finder

**File:** `07_LinkedIn_Job_Lead_Finder.json`

**Purpose:** Scrapes LinkedIn job listings using Apify, enriches company data and finds decision-makers using LinkFinder AI, then saves everything to Google Sheets for lead generation.

**Flow:**
```
Chat Trigger (paste LinkedIn search URL)
  → Linkedin Job Scraper (Apify actor: 2rJKkhh7vjpX7pvjg)
    → Loop Over Items:
      → Enrich Company (LinkFinder AI: linkedin_company_to_linkedin_info)
        → Find Decision Maker (LinkFinder AI: linkedin_company_to_employees)
          → Filter by Job Title (founder, recruiter, HR, director, manager, CEO, owner)
            → Enrich Decision Maker (LinkFinder AI: linkedin_profile_to_linkedin_info)
              → Save to Google Sheets
                → Wait 25s → Next item
```

**Setup:**
- Get an **Apify API token** and enter it in the "Linkedin Job Scraper" node URL
- Get a **LinkFinder AI** API key from [linkfinderai.com](https://linkfinderai.com) and enter in the Authorization header of all 3 LinkFinder nodes
- Configure **Google Sheets OAuth2** credentials
- Create a Google Sheet with columns: `Name`, `Title`, `Company name`, `Email`, `Linkedin_job`, `Website`
- Input: paste a LinkedIn job search URL like:
  `https://www.linkedin.com/jobs/search/?f_TPR=r604800&geoId=100025096&keywords=data+scientist`

> **Note:** The `linkedin_company_to_employees` endpoint requires sufficient LinkFinder AI credits. Set **On Error → Continue On Fail** on the "Find decision maker" node to handle credit exhaustion gracefully.

---

## AI Receptionist With UI

**Directory:** `AI_Receiptionist_With_UI/ai-receptionist-app/`

**Purpose:** A full-stack AI Receptionist application with a modern web interface. Handles inbound/outbound calls using Vapi AI, manages leads and call records in Convex, and integrates with n8n workflows for call processing.

**Tech Stack:**
- **Frontend:** Next.js + React + TypeScript
- **Database:** Convex (real-time backend)
- **Voice AI:** Vapi
- **Automation:** n8n workflows
- **Styling:** Tailwind CSS + shadcn/ui

**Key Features:**
- Dashboard with call analytics
- Lead management system
- Call history with transcripts
- Outbound call campaigns
- Real-time webhook integration with n8n

**Setup:** See `AI_Receiptionist_With_UI/ai-receptionist-app/README.md` for detailed setup instructions.

---

## 🔑 Common Credentials Required

| Service | Used In | How to Get |
|---------|---------|-----------|
| OpenAI API | 03a, 03b, 04, 05, 06 | [platform.openai.com](https://platform.openai.com) |
| Google Sheets OAuth2 | 01, 05, 06, 07 | Google Cloud Console |
| Gmail OAuth2 | 03a, 05, 06 | Google Cloud Console |
| Qdrant | 03a, 03b | [qdrant.io](https://qdrant.io) |
| Supabase | 04 | [supabase.com](https://supabase.com) |
| X (Twitter) OAuth2 | 05 | [developer.twitter.com](https://developer.twitter.com) |
| LinkedIn OAuth2 | 05 | [developer.linkedin.com](https://developer.linkedin.com) |
| JSearch (RapidAPI) | 06 | [rapidapi.com/letscrape-6bRBa3QguO5/api/jsearch](https://rapidapi.com) |
| Jina AI | 06 | [jina.ai](https://jina.ai) |
| Apify | 07 | [apify.com](https://apify.com) |
| LinkFinder AI | 07 | [linkfinderai.com](https://linkfinderai.com) |
| Vapi | AI Receptionist | [vapi.ai](https://vapi.ai) |

---
