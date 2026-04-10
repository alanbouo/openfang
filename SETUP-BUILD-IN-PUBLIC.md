# Build-in-Public System — Setup Guide (Coolify)

This guide walks through setting up the build-in-public content system on your Coolify deployment.

## Prerequisites

- OpenFang deployed on Coolify (Docker build from this repo)
- API keys for: an LLM provider (Groq, Anthropic, Gemini, or OpenAI)
- No social platform API keys needed — this system produces content, you post manually

---

## Step 1 — Set Environment Variables on Coolify

In your Coolify dashboard → Service → Environment Variables, add:

| Variable | Required | Where to get it |
|----------|----------|----------------|
| `GROQ_API_KEY` | **Yes** (or another LLM) | [console.groq.com](https://console.groq.com) |
| `GEMINI_API_KEY` | Recommended (fallback) | [aistudio.google.com](https://aistudio.google.com/apikey) |
| `OPENFANG_API_KEY` | **Yes** | Generate any strong secret |

That's it — only LLM keys. No Twitter/LinkedIn/YouTube API keys needed.

These are already wired in `docker-compose.yml`. Coolify will pass them to the container.

---

## Step 2 — Deploy (Rebuild)

Since we added 5 new agents to `bundled_agents.rs`, Coolify needs to rebuild the Docker image:

1. Push the code changes to your git repo
2. In Coolify → Service → click **Redeploy** (or it auto-deploys if you have webhooks)
3. Wait for the build to complete (~5-10 min for Rust)

After deploy, the new agents are available at `/opt/openfang/agents/` inside the container.

---

## Step 3 — Spawn the Agents (API Calls)

Once OpenFang is running, spawn the 5 new agents via the API. Replace `YOUR_HOST` with your Coolify domain and `YOUR_API_KEY` with your `OPENFANG_API_KEY`.

### 3a. Build-in-Public Curator
```bash
curl -X POST https://YOUR_HOST/api/agents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"template": "build-in-public-curator"}'
```

### 3b. Build-in-Public Poster
```bash
curl -X POST https://YOUR_HOST/api/agents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"template": "build-in-public-poster"}'
```

### 3c. Weekly Digest Writer
```bash
curl -X POST https://YOUR_HOST/api/agents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"template": "weekly-digest-writer"}'
```

### 3d. Video Scriptwriter
```bash
curl -X POST https://YOUR_HOST/api/agents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"template": "video-scriptwriter"}'
```

### 3e. Short Scriptwriter
```bash
curl -X POST https://YOUR_HOST/api/agents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"template": "short-scriptwriter"}'
```

### 3f. Marketing Director (already bundled)
```bash
curl -X POST https://YOUR_HOST/api/agents \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"template": "marketing-director"}'
```

---

## Step 4 — Activate the Hands

### 4a. Collector Hand (RSS/X monitoring)
```bash
curl -X POST https://YOUR_HOST/api/hands/collector/activate \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "config": {
      "target_subject": "AI agents, dev tools, open source, Rust ecosystem, building in public",
      "collection_depth": "deep",
      "update_frequency": "daily",
      "focus_area": "technology",
      "alert_on_changes": true,
      "report_format": "markdown"
    }
  }'
```

That's the only Hand needed. No Twitter Hand — the system doesn't post anywhere.

---

## Step 5 — Verify Everything is Running

### List all agents
```bash
curl https://YOUR_HOST/api/agents \
  -H "Authorization: Bearer YOUR_API_KEY" | jq '.[].name'
```

Expected output should include:
```
"build-in-public-curator"
"build-in-public-poster"
"weekly-digest-writer"
"video-scriptwriter"
"short-scriptwriter"
"marketing-director"
```

### List active hands
```bash
curl https://YOUR_HOST/api/hands/active \
  -H "Authorization: Bearer YOUR_API_KEY" | jq
```

---

## Step 6 — Customize Your Sources

After spawning, message the **build-in-public-curator** agent to configure your specific feeds:

```bash
curl -X POST https://YOUR_HOST/api/agents/CURATOR_AGENT_ID/message \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "Here are my RSS feeds to monitor daily:\n- https://news.ycombinator.com/rss\n- https://lobste.rs/rss\n- [ADD YOUR FEEDS]\n\nAnd these X/Twitter topics to track:\n- building in public\n- AI agents\n- Rust programming\n- [ADD YOUR TOPICS]\n\nPlease store these as your daily monitoring targets."
  }'
```

Similarly, message **build-in-public-poster** to set your brand voice:

```bash
curl -X POST https://YOUR_HOST/api/agents/POSTER_AGENT_ID/message \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "message": "My brand voice: [DESCRIBE HOW YOU TALK]. I build [WHAT YOU BUILD]. My audience is [WHO]. Store this as the brand voice for all future posts."
  }'
```

---

## How Data Flows (Important!)

### X/Twitter Data Flow
The **Collector Hand** (activated in Step 4a) does the heavy lifting:
1. Runs autonomously on schedule (daily by default)
2. Uses `web_search` to monitor your configured topics
3. Writes `collector_report_YYYY-MM-DD.md` files to disk
4. The **build-in-public-curator** reads these reports and adds relevant items to `shared.bip.daily_digest`

You don't manually feed X/Twitter data — the Collector Hand gathers it automatically.

### Manual Input (Your Daily Updates)
You provide your "what I built today" updates by messaging the **build-in-public-poster** agent directly. The agent stores your input to `shared.bip.manual_input`, then combines it with the daily digest to create posts.

---

## Daily Usage

### Morning — Trigger the curator
```bash
curl -X POST https://YOUR_HOST/api/agents/CURATOR_AGENT_ID/message \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Run your daily collection and produce today'\''s digest."}'
```

### Mid-day — Share what you built (Manual Input)
This is how you feed your daily updates into the system. The poster agent stores this to `shared.bip.manual_input` and creates drafts:

```bash
curl -X POST https://YOUR_HOST/api/agents/POSTER_AGENT_ID/message \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Today I built:\n- Implemented OAuth2 in the OpenFang API\n- Struggled with token refresh logic, solved it by using a custom middleware\n- Key learning: Always validate the token BEFORE processing the request\n\nDraft X and LinkedIn posts about this. Combine with today's digest if relevant."}'
```

The agent will:
1. Store your update to `shared.bip.manual_input`
2. Read `shared.bip.daily_digest` (from the curator)
3. Draft posts that combine your build update + curated insights
4. Write drafts to `posts-2026-04-10.md`

### Afternoon — Read your drafts
Open the OpenFang dashboard at `https://YOUR_HOST`, go to the poster agent, and read its output files.
Or fetch via API:
```bash
curl https://YOUR_HOST/api/agents/POSTER_AGENT_ID/files \
  -H "Authorization: Bearer YOUR_API_KEY"
```
Then manually copy the posts you like to X, LinkedIn, your blog, etc.

### Weekly — Generate blog digest
```bash
curl -X POST https://YOUR_HOST/api/agents/DIGEST_AGENT_ID/message \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Compile this week'\''s digest into a blog post."}'
```

### As needed — Video scripts
```bash
# Long-form script
curl -X POST https://YOUR_HOST/api/agents/VIDEO_AGENT_ID/message \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Write a video script about [TOPIC]. Type: tutorial, ~15 minutes."}'

# Shorts batch
curl -X POST https://YOUR_HOST/api/agents/SHORTS_AGENT_ID/message \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"message": "Produce 5 YouTube Shorts scripts from today'\''s digest."}'
```

---

## Optional — Set Up Cron Schedules

Automate the daily curator run:
```bash
curl -X POST https://YOUR_HOST/api/schedules \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "CURATOR_AGENT_ID",
    "cron": "0 8 * * *",
    "message": "Run your daily collection and produce today'\''s digest.",
    "description": "Daily morning content collection"
  }'
```

Automate the weekly blog digest:
```bash
curl -X POST https://YOUR_HOST/api/schedules \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agent_id": "DIGEST_AGENT_ID",
    "cron": "0 10 * * 5",
    "message": "Compile this week'\''s digest into a blog post.",
    "description": "Weekly Friday blog digest"
  }'
```

---

## Architecture Summary

```
You (daily input via message)
    │
    ▼
┌────────────────────────┐
│  Marketing Director    │ ← orchestrates all agents
└────────┬───────────────┘
         │ agent_send
    ┌────┼────┬──────────┬──────────────┐
    ▼    ▼    ▼          ▼              ▼
Curator Poster Digest  Video-Script  Short-Script
  │       │     Writer    Writer        Writer
  │       │       │         │              │
  ▼       ▼       ▼         ▼              ▼
shared.bip.*  (shared memory namespace)
  │                 │
  ▼                 ▼
┌──────────┐    ┌───────────────────────────────┐
│ Collector │    │  Output files (.md)           │
│   Hand    │    │  posts-*.md, blog-*.md,       │
│ (RSS/X)  │    │  script-*.md, shorts-*.md     │
└──────────┘    │  YOU read & post manually     │
                  └───────────────────────────────┘
```

## Files Created

| File | Purpose |
|------|---------|
| `agents/build-in-public-curator/agent.toml` | RSS/X content curation |
| `agents/build-in-public-poster/agent.toml` | X/LinkedIn post drafting |
| `agents/weekly-digest-writer/agent.toml` | Weekly blog digest |
| `agents/video-scriptwriter/agent.toml` | YouTube long-form scripts |
| `agents/short-scriptwriter/agent.toml` | YouTube Shorts scripts |
