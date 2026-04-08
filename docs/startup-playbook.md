# Startup Playbook: Hermes Agent as Your AI Marketing & Sales Team

> For solo founders and small teams building hardware + software products

---

## Why This Matters

You're a 1-2 person hardware/software startup. You can build the product, but marketing, sales, and community building fall through the cracks because there aren't enough hours in the day.

Hermes Agent can run 24/7 on a $5-10/month VPS as your **always-on marketing engine** — posting content, monitoring competitors, following up with leads, managing community channels, and creating sales materials while you focus on building.

This guide covers the highest-impact workflows for a resource-constrained founder, starting with quick wins you can set up in a day.

---

## Quick Wins (Day 1)

### 1. Social Media on Autopilot

**Skill:** `skills/social-media/xitter/SKILL.md`

Use the X/Twitter skill to maintain a consistent social media presence without manual effort.

**What it does:**
- Post tweets, replies, and quote tweets via `x-cli`
- Search for relevant conversations in your niche
- Monitor mentions and bookmark potential leads
- Like and retweet relevant content for engagement

**Set up scheduled posting with cron:**
```bash
# Post a product tip every weekday morning at 9am
hermes cron create "0 9 * * 1-5" \
  --prompt "Write and post a tweet about a practical tip related to [your product category]. Keep it under 280 chars, be helpful not salesy. Use the xitter skill to post it." \
  --skills xitter

# Monitor mentions and competitor activity every 30 minutes
hermes cron create "*/30 * * * *" \
  --prompt "Search X for mentions of [your brand] and [competitor names]. Summarize any new mentions. Bookmark interesting leads." \
  --skills xitter \
  --deliver telegram
```

**Requirements:** X API credentials (paid developer access — see skill docs for details).

### 2. Content Repurposing Pipeline

**Skill:** `skills/media/youtube-content/SKILL.md`

Turn existing video content (yours or industry talks) into a week's worth of marketing material.

**What it does:**
- Extracts transcripts from any YouTube video
- Converts to Twitter threads (280-char optimized), blog posts, key quotes, chapter summaries
- Supports multiple languages

**Workflow:**
1. Feed it a YouTube URL of your product demo, a conference talk, or an industry video
2. Get back a Twitter thread, blog post draft, and key quotes — all from one video
3. Schedule the thread via xitter cron, use the blog post for your site, share quotes on social

### 3. Competitive Intelligence on Autopilot

**Skill:** `skills/research/blogwatcher/SKILL.md`

Never miss what your competitors are doing.

**What it does:**
- Monitors blogs and RSS/Atom feeds for new posts
- Automatic feed discovery from URLs
- OPML import for bulk blog management
- Tracks read/unread status

**Set up a weekly digest:**
```bash
# Monday morning competitive intel digest
hermes cron create "0 8 * * 1" \
  --prompt "Check all monitored blogs for new posts from the past week. Summarize the key announcements, product updates, and pricing changes from [competitor1], [competitor2], [competitor3]. Flag anything that affects our positioning." \
  --skills blogwatcher \
  --deliver telegram
```

---

## Sales & Lead Management

### 4. CRM in Notion

**Skill:** `skills/productivity/notion/SKILL.md`

Use Notion as a lightweight CRM that Hermes reads and writes to automatically.

**Set up a lead database with these properties:**
- Name (title)
- Email (email)
- Source (select: Website, Twitter, Discord, Referral, Trade Show)
- Status (select: New, Contacted, Replied, Qualified, Won, Lost)
- Last Contact (date)
- Notes (rich text)
- Deal Value (number)

**Automated lead follow-up:**
```bash
# Daily at 10am: check for leads that haven't been contacted in 5+ days
hermes cron create "0 10 * * *" \
  --prompt "Query the Leads database in Notion. Find any leads where Status is 'Contacted' or 'New' and Last Contact is more than 5 days ago. For each one, draft a personalized follow-up email based on their Notes field. Send the drafts to me for review." \
  --skills notion \
  --deliver telegram
```

**Requirements:** Notion API key + share your database with the integration.

### 5. Email Outreach Sequences

**Skills:** `skills/email/himalaya/SKILL.md` + `skills/productivity/google-workspace/SKILL.md`

Automate email outreach without paying for expensive email marketing tools.

**Himalaya** handles IMAP/SMTP email directly from the terminal:
- Send, search, and organize emails
- Multiple account support
- Non-interactive composition (perfect for automation)

**Google Workspace** integrates Gmail, Calendar, Drive, Sheets:
- Gmail search operators (`is:unread`, `from:`, `newer_than:`)
- Send HTML emails
- Read/write Google Sheets for tracking

**Example outreach workflow:**
```bash
# Weekly: send personalized outreach to new leads from Google Sheets
hermes cron create "0 10 * * 2" \
  --prompt "Read the 'Outreach Queue' tab in the leads Google Sheet. For each row where 'Sent' column is empty, compose a personalized introductory email about [your product]. Send via himalaya. Mark the row as sent with today's date." \
  --skills himalaya,google-workspace
```

### 6. Stripe Payment Automation

**Platform:** `gateway/platforms/webhook.py`

When a customer pays, trigger an automated workflow.

**What webhooks enable:**
- Receive HTTP POST events with HMAC signature validation
- Route to Hermes with templated prompts
- Deliver responses to Telegram, email, or any platform

**Example:** Stripe payment → thank-you email + CRM update + Telegram notification:
```bash
hermes webhook create \
  --url /stripe-payments \
  --secret whsec_your_stripe_secret \
  --prompt "A customer just paid. Name: {payload.data.object.customer_email}, Amount: {payload.data.object.amount_paid}. Send them a thank-you email via himalaya. Update their status to 'Won' in the Notion leads database. Notify me on Telegram." \
  --skills himalaya,notion \
  --deliver telegram
```

---

## Community Building & Customer Support

### 7. Discord/Telegram Community Bot

**Platforms:** `gateway/platforms/discord.py`, `gateway/platforms/telegram.py`

Run Hermes as your community's support agent across multiple channels simultaneously.

**What it does:**
- Answers product questions using its memory of your product specs, docs, and FAQs
- Routes complex issues to you via DM
- Tracks each customer's history across sessions (memory persists)
- Serves Discord, Telegram, WhatsApp, and email from a single agent instance

**Set up:**
1. Configure the gateway with your platform tokens in `~/.hermes/config.yaml`
2. Store your product documentation, specs, and FAQs in Hermes's memory (`MEMORY.md`)
3. The agent answers questions automatically and escalates what it can't handle

**Scheduled community engagement:**
```bash
# Post a weekly tip to your Discord community
hermes cron create "0 12 * * 3" \
  --prompt "Write a helpful tip about [your product category] for our community. Include a practical example. Post it to the #tips channel." \
  --deliver discord
```

**Key advantage:** The memory system (`tools/memory_tool.py`) means Hermes remembers each customer's past interactions, hardware version, known issues, and preferences — continuity that feels like a dedicated support person.

---

## Content Creation Engine

### 8. Product Visuals & Marketing Graphics

**Skill:** `skills/mlops/models/stable-diffusion/SKILL.md`

Generate product mockups, social media graphics, and ad creatives.

**Capabilities:**
- Text-to-image generation (product concepts, lifestyle shots)
- Image-to-image (style transfer, product variations)
- Multiple models: SD 1.5, SDXL, SD 3.0, Flux
- ControlNet for spatial conditioning
- LoRA for consistent brand style

### 9. Pitch Decks & Sales Materials

**Skill:** `skills/productivity/powerpoint/SKILL.md`

Auto-generate presentations for investors, customers, and partners.

**Use cases:**
- Investor pitch decks
- Product one-pagers
- Sales presentations with your latest specs and pricing
- Trade show materials

### 10. Branded Audio Content

**Skill:** `skills/media/heartmula/SKILL.md`

Generate music for product videos, ads, and social content — no licensing fees.

- Full songs from lyrics + tags
- MP3 output at 48kHz stereo
- Multilingual support
- Local/offline generation

---

## Automated Marketing Workflows

Here are the cron schedules that give a solo founder the most leverage:

| Schedule | What It Does | Skills Used |
|---|---|---|
| `0 9 * * 1-5` | Morning social media post (product tip or industry insight) | xitter |
| `*/30 * * * *` | Monitor X mentions and competitor activity | xitter |
| `0 18 * * 5` | Draft weekly newsletter from the week's content | himalaya, google-workspace |
| `0 8 * * 1` | Monday competitive intelligence digest | blogwatcher |
| `0 10 * * *` | Check for stale leads in Notion, draft follow-ups | notion, himalaya |
| `0 12 * * 3` | Post community tip to Discord/Telegram | (gateway delivery) |
| `0 7 * * *` | Daily sales pipeline summary | notion, google-workspace |

---

## Marketing Stack Architecture

```
                        ┌─────────────────────┐
                        │    Hermes Agent      │
                        │   (VPS / $5-10/mo)   │
                        └──────────┬──────────┘
                                   │
          ┌────────────────────────┼─────────────────────────┐
          │                        │                         │
    ┌─────▼─────┐          ┌──────▼──────┐          ┌──────▼──────┐
    │  Outbound  │          │   Inbound   │          │    Data     │
    │  Channels  │          │   Channels  │          │   Stores    │
    └─────┬─────┘          └──────┬──────┘          └──────┬──────┘
          │                       │                        │
  ┌───────┼───────┐       ┌──────┼───────┐        ┌──────┼───────┐
  │       │       │       │      │       │        │      │       │
X/Twitter Email Discord  Stripe  Web   Telegram  Notion Sheets  Gmail
                WhatsApp  Hooks  Forms  Community  CRM   Leads  Outreach
                                                                
                        ┌─────────────────────┐
                        │   Cron Scheduler     │
                        │                      │
                        │  Posts · Follow-ups   │
                        │  Digests · Monitoring │
                        └─────────────────────┘
```

---

## What's Not Built-In (and Workarounds)

| Gap | Workaround |
|---|---|
| Shopify / e-commerce | Use Stripe webhooks for payment events; manage inventory in Notion/Sheets |
| Google Ads / Meta Ads | Manual for now; MCP servers may add this in the future |
| SEO tracking | Domain intelligence skill (`skills/domain/`) for basic recon; use external SEO tools |
| Analytics dashboards | Pipe data to Google Sheets; use Sheets as a lightweight dashboard |
| Landing page builder | External tool (Carrd, etc.) with webhook to Hermes for form submissions |
| SMS marketing | SMS adapter available via gateway, but requires Twilio setup |

---

## Getting Started Checklist

Deploy your AI marketing team in a day:

1. **Deploy Hermes** on a $5-10/month VPS (or locally for testing)
2. **Connect Telegram** for personal notifications (fastest to set up)
3. **Set up X/Twitter** skill with API credentials for social posting
4. **Create a Notion CRM** database and connect it to Hermes
5. **Configure 3 starter cron jobs:**
   - Daily social media post
   - Weekly competitive digest
   - Daily lead follow-up check
6. **Store your product knowledge** in Hermes's memory (specs, FAQs, pricing)
7. **Launch a community channel** (Discord or Telegram) with Hermes as support bot
8. **Add email outreach** via Himalaya or Google Workspace when ready
9. **Set up Stripe webhooks** when you start taking payments

**Estimated monthly cost:** $5-10 VPS + LLM API usage (~$10-30/month for moderate usage via OpenRouter) = **$15-40/month for a 24/7 marketing assistant**.

---

## Tips for Solo Founders

- **Start with 2-3 cron jobs**, not 20. Add more as you validate what works.
- **Use Telegram as your control plane.** All digests, alerts, and drafts come to your phone.
- **Let Hermes draft, you approve.** Use cron delivery to review content before it goes live until you trust the outputs.
- **Memory is your moat.** The more Hermes learns about your product, customers, and market, the better its outputs get. Feed it your docs, specs, and customer conversations.
- **Profile isolation** lets you run separate Hermes instances for different brands or products if needed.
