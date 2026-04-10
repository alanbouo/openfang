# Build in Public Communication System — Requirements Specification

## Overview
A content curation and creation system using OpenFang to support a "build in public" strategy across multiple platforms.

## Goals
- Share daily learnings and building progress
- Curate and share tech insights from X/Twitter sources and RSS feeds
- Repurpose popular social content into blog posts
- Generate video scripts (long-form + shorts) to enable YouTube production

## Platforms
| Platform | Purpose | Frequency |
|----------|---------|-----------|
| **Twitter/X** | Daily build updates, curated tech insights | 1-3x daily |
| **LinkedIn** | Professional thought leadership | 3-4x weekly |
| **YouTube** | Long-form videos (you record from scripts) | 1-2x weekly |
| **YouTube Shorts** | Short scripts: quick tips, hooks, highlights | 3-5x weekly |
| **Blog** | Weekly digest of popular content, expanded tutorials | 1x weekly |

## Content Types
1. **Daily Build Updates** — What was built/learned today
2. **Curated Tech Insights** — Summarized findings from RSS/X feeds
3. **Tutorial Content** — How-to guides based on learnings
4. **Video Scripts (Long-form)** — Structured outlines, talking points, hooks for YouTube videos
5. **Short Scripts** — Punchy 30-60s scripts for YouTube Shorts (hook → insight → CTA)

## Automation Level
**Content Creation Only** — Agents draft all content (posts, scripts, blog drafts) and present it to the user. The user manually copies and publishes wherever they want. No auto-posting.

## Content Flow
```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│  RSS Feeds      │────→│              │     │                 │
│  X/Twitter (web)│────→│  Collector   │────→│  Curator Agent  │
│  Manual Input   │──┐  │   Hand       │     │  (daily digest) │
└─────────────────┘  │  └──────────────┘     └────────┬────────┘
                     │                                │
                     │         ┌──────────────────────┼──────────────┐
                     │         │                      │              │
                     ▼         ▼                      ▼              ▼
              ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
              │  Post Drafter │  │  Video Script │  │  Short Script │
              │    Agent      │  │    Writer     │  │    Writer     │
              │ (X/LinkedIn)  │  │ (YT long-form)│  │(YT Shorts)   │
              └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
                      │                  │                  │
                      ▼                  ▼                  ▼
              ┌───────────────────────────────────────────────────┐
              │              Output Files (for YOU)               │
              │  posts-2026-04-10.md  │  script-*.md  │  blog-*.md│
              │  YOU copy & paste to X, LinkedIn, blog, etc.      │
              └───────────────────────────────────────────────────┘
```

## Required OpenFang Components

### Agents to Spawn
| Agent | Role |
|-------|------|
| `social-marketer` | Creates platform-specific content for X/LinkedIn |
| `blog-writer` | Writes weekly digest blog posts from popular social content |
| `video-scriptwriter` | (Custom) Writes structured YouTube video scripts with outlines, hooks, talking points |
| `short-scriptwriter` | (Custom) Writes punchy 30-60s YouTube Shorts scripts (hook → insight → CTA) |
| `content-curator` | (Custom) Filters and summarizes RSS/X feed content |

### Hands to Activate
| Hand | Purpose |
|------|---------|
| `Collector` | Monitors RSS feeds and X sources for tech insights |

### Channels to Configure
None — this system does NOT publish. It produces content files that you manually post.

## Data Sources to Integrate
- RSS feeds (tech blogs, newsletters)
- X/Twitter lists (tech influencers, builders)
- Manual daily input (what was built/learned)

## Workflow
1. **Morning**: Collector Hand gathers overnight updates from RSS/X
2. **Mid-day**: User provides "what I built today" input
3. **Afternoon**: Social-Marketer agent drafts X/LinkedIn posts
4. **Evening**: User reads drafts and manually posts to X/LinkedIn/blog
5. **Weekly**: Blog-Writer creates digest from most engaged posts
6. **As needed**: Video-Scriptwriter drafts long-form scripts, Short-Scriptwriter drafts Shorts scripts

## Implementation Plan

### Phase 1 — Configuration & API Keys

**Goal:** Get OpenFang configured with an LLM provider. No social platform API keys needed.

**Action items:**
1. Set env vars on Coolify:
   - `GROQ_API_KEY` — for LLM (agents default to Groq)
   - `GEMINI_API_KEY` — for fallback models
   - `OPENFANG_API_KEY` — for API authentication

---

### Phase 2 — Adapt the Content-Curator Agent for Build-in-Public

**Goal:** Retool the existing `content-curator` agent to monitor YOUR specific RSS feeds and X sources.

**What already exists:**
- `agents/content-curator/agent.toml` — generic content curation agent

**Action items:**
1. Create a new agent `build-in-public-curator` (fork of content-curator) with:
   - Hardcoded list of YOUR RSS feeds (tech blogs, newsletters)
   - Hardcoded list of X accounts/lists to monitor
   - System prompt focused on "builder" content: dev tools, AI, open source, etc.
   - Output format: daily digest stored to `shared.bip.daily_digest`
2. Add `web_fetch` tool for pulling RSS XML and parsing it
3. Store curated items in shared memory under `shared.bip.*` namespace

---

### Phase 3 — Adapt the Social-Marketer Agent for Your Voice

**Goal:** Customize the existing `social-marketer` agent to post in YOUR voice.

**What already exists:**
- `agents/social-marketer/agent.toml` — generic multi-platform social agent

**Action items:**
1. Create `build-in-public-poster` agent (fork of social-marketer) with:
   - Your brand voice baked into the system prompt
   - Platform rules: X (daily build updates) + LinkedIn (3-4x/week thought leadership)
   - Reads from `shared.bip.daily_digest` (curator output) + manual input
   - Writes drafts to `shared.bip.post_queue` for approval
   - Content format: "🔨 Built today: ...", "📚 Learned today: ...", curated threads
2. Output: dated markdown files (e.g., `posts-2026-04-10.md`) the user copies from

---

### Phase 4 — Activate the Collector Hand

**Goal:** Autonomous background monitoring of RSS/X feeds.

**What already exists:**
- `crates/openfang-hands/bundled/collector/HAND.toml` — full 7-phase collector

**Action items:**
1. Activate Collector Hand via API with config:
   ```json
   {
     "target_subject": "AI agents, dev tools, open source, Rust ecosystem",
     "collection_depth": "deep",
     "update_frequency": "daily",
     "focus_area": "technology",
     "alert_on_changes": true,
     "report_format": "markdown"
   }
   ```
2. The Collector produces `collector_report_YYYY-MM-DD.md` — this feeds the curator

---

### Phase 5 — Blog Writer for Weekly Digest

**Goal:** Auto-generate weekly blog post from most engaged social content.

**What already exists:**
- `agents/blog-writer/agent.toml` — SEO blog writer

**Action items:**
1. Create `weekly-digest-writer` agent (fork of blog-writer) with:
   - System prompt to pull from `shared.bip.*` memory (week's posts, engagement data)
   - Reads the week's post drafts and daily digests
   - Generates a "This Week I Built" blog post
   - Stores draft to file for approval
2. Schedule weekly (cron or manual trigger)

---

### Phase 6 — Video Script Agents

**Goal:** Generate ready-to-record scripts for YouTube long-form and Shorts.

**What already exists:**
- `agents/writer/agent.toml` — general content writer (base for forks)

**Action items:**
1. Create `video-scriptwriter` agent with:
   - System prompt for structured YouTube scripts: hook, intro, sections with talking points, CTA, outro
   - Reads from `shared.bip.*` (curated insights, build updates) for topic ideas
   - Outputs: title, thumbnail idea, script outline, full script with timestamps
   - Stores drafts to file for review
2. Create `short-scriptwriter` agent with:
   - System prompt for 30-60s vertical video scripts
   - Format: hook (first 3s) → insight/tip → CTA
   - Reads from `shared.bip.daily_digest` to repurpose social content into shorts
   - Batch produces 3-5 short scripts at a time

---

### Phase 7 — Wire It All Together with Marketing Director

**Goal:** Orchestrate the full daily workflow.

**What already exists:**
- `agents/marketing-director/agent.toml` — orchestrator with `agent_send` capabilities

**Action items:**
1. Customize marketing-director to know the daily workflow:
   - Morning: Check Collector Hand report → send to curator
   - Mid-day: Accept manual "what I built today" input
   - Afternoon: Tell social-marketer to draft posts
   - Evening: Present approval queue
   - Weekly: Trigger blog-writer for digest
2. The director uses `agent_send` to coordinate all specialist agents
3. All agents share `shared.bip.*` memory namespace

---

### Implementation Order

| Step | What | Effort | Blocking? |
|------|------|--------|-----------|
| **1** | Get LLM API keys (Groq, Gemini) | User action | Yes — blocks everything |
| **2** | Create `build-in-public-curator` agent | New agent.toml | No |
| **3** | Create `build-in-public-poster` agent | New agent.toml | No |
| **4** | Activate Collector Hand | API call + config | Needs LLM key |
| **5** | Create `weekly-digest-writer` agent | New agent.toml | No |
| **6** | Create `video-scriptwriter` + `short-scriptwriter` agents | New agent.toml x2 | No |
| **7** | Customize marketing-director for daily workflow | Edit agent.toml | After steps 2-5 |
| **9** | Test end-to-end with manual input | Manual | After all above |

### What Needs to Be BUILT (Code Changes)

1. **5 new agent.toml files** — `build-in-public-curator`, `build-in-public-poster`, `weekly-digest-writer`, `video-scriptwriter`, `short-scriptwriter`
2. **Edit marketing-director** — add build-in-public workflow to system prompt
3. **Bundle new agents** — add them to `bundled_agents.rs` for Docker deployment
