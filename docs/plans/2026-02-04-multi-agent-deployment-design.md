# Multi-Agent Deployment Design for Phala Cloud

**Date:** 2026-02-04
**Status:** Draft
**Author:** Brainstormed with Claude

## Overview

Transform the existing single-agent Phala CVM deployment into a multi-agent system where:

- An **orchestrator agent** handles onboarding and coordination
- **Worker agents** are created per-user via Gmail OAuth
- All agents share access to company tools (Notion, Attio, Google Workspace)
- Agents coordinate cross-org while humans approve all actions

## Goals

1. **Minimal setup** — single CVM, pre-installed skills, bootstrap config
2. **Easy onboarding** — user requests or admin initiates, Gmail OAuth verifies identity
3. **Shared resources** — uniform access to Notion, Attio, Google via skills
4. **Human-driven automation** — agents propose, humans approve
5. **Extensible** — add chat channels (Telegram, Discord) per-agent later

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Phala CVM                                 │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                     OpenClaw Gateway                           │ │
│  │                                                                │ │
│  │  ┌─────────────┐    creates/manages    ┌──────────────────┐   │ │
│  │  │ Orchestrator│───────────────────────►│  Worker Agents   │   │ │
│  │  │    Agent    │◄──sessions_send/spawn──│ alice│bob│carol  │   │ │
│  │  └──────┬──────┘                        └────────┬─────────┘   │ │
│  │         │                                        │             │ │
│  │         │              ┌─────────────────────────┘             │ │
│  │         │              │                                       │ │
│  │         ▼              ▼                                       │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │                  Shared Skills                           │  │ │
│  │  │   steipete/gog │ steipete/notion │ kesslerio/attio-crm  │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                │                                    │
│                           Port 18789                                │
└────────────────────────────────┼────────────────────────────────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        ▼                        ▼                        ▼
   Admin Dashboard          User Dashboards          Future: Chat
   ?token=ADMIN             ?token=USER_XXX          (Telegram, etc.)
```

### Key Components

| Component | Role |
|-----------|------|
| **Orchestrator Agent** | Admin agent — handles onboarding requests, creates worker agents, manages shared config |
| **Worker Agents** | One per user — personal assistant with access to shared tools |
| **Shared Skills** | gog (Google), notion, attio-crm — available to all agents |
| **S3 Storage** | Encrypted persistence — survives CVM restarts/destruction |

## Onboarding Flow

Two paths to onboard new users, both ending in Gmail OAuth verification.

### Path 1: User Request Flow

```
User                    Orchestrator                      System
  │                          │                               │
  │  "I'd like an agent"     │                               │
  │  (email/dashboard msg)   │                               │
  ├─────────────────────────►│                               │
  │                          │  Generate unique invite link  │
  │                          │  (contains pending agent ID)  │
  │                          ├──────────────────────────────►│
  │   "Click here to setup"  │                               │
  │◄─────────────────────────┤                               │
  │                          │                               │
  │  Click link → Gmail OAuth│                               │
  ├──────────────────────────┼──────────────────────────────►│
  │                          │                               │
  │                          │  OAuth complete, email verified│
  │                          │◄──────────────────────────────┤
  │                          │                               │
  │                          │  Create agent config          │
  │                          │  Generate user token          │
  │                          │  Add to agents.list           │
  │                          ├──────────────────────────────►│
  │                          │                               │
  │  "Your agent is ready"   │                               │
  │  (dashboard URL + token) │                               │
  │◄─────────────────────────┤                               │
```

### Path 2: Admin-Initiated Flow

```
Admin                   Orchestrator                      System
  │                          │                               │
  │  "Onboard alice@co.com"  │                               │
  ├─────────────────────────►│                               │
  │                          │  Send invite email to Alice   │
  │                          ├──────────────────────────────►│
  │                          │                               │
  │  "Invite sent"           │        (Alice clicks link,    │
  │◄─────────────────────────┤         completes OAuth)      │
  │                          │                               │
  │                          │  OAuth complete notification  │
  │                          │◄──────────────────────────────┤
  │                          │                               │
  │  "Alice is now active"   │  Create agent, notify Alice   │
  │◄─────────────────────────┤──────────────────────────────►│
```

### Gmail OAuth Details

The orchestrator uses the `gog` (Google Workspace CLI) skill for OAuth:

1. **Invite link** points to a lightweight page that triggers `gog auth add`
2. **OAuth scopes requested:**
   - `gmail.readonly` — read emails for context
   - `gmail.send` — send on user's behalf (with approval)
   - `userinfo.email` — verify identity
3. **Credentials stored** in agent's auth profile
4. **Email becomes agent ID** — `alice@company.com` → agent ID `alice-company-com`

## Agent Configuration

### Agent Definition

```jsonc
{
  "agents": {
    "list": [
      {
        "id": "orchestrator",
        "name": "Orchestrator",
        "default": true,
        "workspace": "/data/openclaw/workspaces/orchestrator",
        "identity": {
          "name": "Company Hub",
          "personality": "Helpful admin assistant that onboards users and coordinates work"
        },
        "subagents": {
          "allowAgents": ["*"]
        }
      },
      {
        "id": "alice-company-com",
        "name": "Alice's Agent",
        "workspace": "/data/openclaw/workspaces/alice-company-com",
        "identity": {
          "name": "Alice's Assistant",
          "personality": "Personal work assistant for Alice"
        },
        "subagents": {
          "allowAgents": ["orchestrator", "*"]
        }
      }
    ]
  }
}
```

### Token-Based Routing

Each user gets a unique token that routes them to their agent:

```jsonc
{
  "gateway": {
    "auth": {
      "token": "ADMIN_TOKEN_HERE",
      "userTokens": {
        "ALICE_TOKEN_ABC123": { "agentId": "alice-company-com" },
        "BOB_TOKEN_DEF456": { "agentId": "bob-company-com" }
      }
    }
  }
}
```

**Dashboard URLs:**

| User | URL |
|------|-----|
| Admin | `https://<cvm>-18789.<gateway>.phala.network?token=ADMIN_TOKEN` |
| Alice | `https://<cvm>-18789.<gateway>.phala.network?token=ALICE_TOKEN_ABC123` |

## Shared Resources via Skills

Skills are pre-installed during CVM setup, available to all agents from day one.

### Skills to Install

| Skill | Source | Purpose |
|-------|--------|---------|
| `steipete/gog` | clawhub.ai | Gmail, Calendar, Drive, Docs, Sheets, Contacts |
| `steipete/notion` | clawhub.ai | Create/read/update pages, query databases |
| `kesslerio/attio-crm` | clawhub.ai | Contacts, companies, deals, tasks, notes |

### Dockerfile Addition

```dockerfile
# Pre-install shared skills
RUN openclaw skills install steipete/gog && \
    openclaw skills install steipete/notion && \
    openclaw skills install kesslerio/attio-crm
```

### Skills Configuration

```jsonc
{
  "skills": {
    "installed": {
      "steipete/gog": {
        "enabled": true,
        "config": {
          "clientId": "${GOOGLE_CLIENT_ID}",
          "clientSecret": "${GOOGLE_CLIENT_SECRET}"
        }
      },
      "steipete/notion": {
        "enabled": true,
        "config": {
          "apiKey": "${NOTION_API_KEY}"
        }
      },
      "kesslerio/attio-crm": {
        "enabled": true,
        "config": {
          "apiKey": "${ATTIO_API_KEY}"
        }
      }
    }
  }
}
```

### Access Model

- **Uniform access** — all agents have the same permissions
- **Per-user Gmail** — each user's agent uses their own OAuth token
- **Shared company tools** — Notion/Attio use company-wide API keys

## Inter-Agent Communication

Agents coordinate through OpenClaw's `sessions_send` and `sessions_spawn` tools.

### Communication Patterns

| Pattern | Example | Tool |
|---------|---------|------|
| **Escalate to orchestrator** | "I need admin help with X" | `sessions_send` to orchestrator |
| **Peer request** | Alice asks Bob: "Status on Project Y?" | `sessions_send` to bob-company-com |
| **Broadcast** | Orchestrator: "New policy update" | `sessions_send` to each worker |
| **Background task** | "Research this and report back" | `sessions_spawn` (non-blocking) |

### Example: Cross-Team Coordination

```
Alice's Agent                    Bob's Agent
     │                                │
     │  sessions_send:                │
     │  "What's the deal status       │
     │   for Acme Corp?"              │
     ├───────────────────────────────►│
     │                                │  (queries Attio)
     │                                │
     │  "Acme Corp: $50k,             │
     │   Stage: Negotiation"          │
     │◄───────────────────────────────┤
     │                                │
     │  (proposes action to Alice)    │
```

## Human Approval Workflow

Every action proposal flows through the user's dashboard for approval.

### What Requires Approval

| Action | Approval Needed |
|--------|-----------------|
| Send email | ✅ Always |
| Create/edit Notion page | ✅ Always |
| Update Attio record | ✅ Always |
| Read email/docs | ❌ No (read-only) |
| Search Notion/Attio | ❌ No (read-only) |
| Message another agent | ✅ Always |
| Schedule calendar event | ✅ Always |

**Principle:** Reads are free, writes require approval.

### Proposal Format

```markdown
📋 **Action Proposal**

**Type:** Send Email
**Via:** Gmail (alice@company.com)

**To:** client@acme.com
**Subject:** Follow-up on Q2 proposal

**Body:**
> Hi Sarah,
> Following up on our conversation yesterday...

---
Reply with:
- "approved" or "yes" to send
- "edit: [your changes]" to modify
- "no" to cancel
```

### Future: Graduated Autonomy

The architecture supports adding trust levels later:

```jsonc
{
  "agents": {
    "list": [{
      "id": "alice-company-com",
      "autonomy": {
        "level": 1,  // 1 = always ask (default)
        "grants": [] // Future: ["gmail.send", "notion.update"]
      }
    }]
  }
}
```

## Dashboard Access & Security

### URL Structure

```
https://<app_id>-18789.<gateway>.phala.network?token=<USER_TOKEN>
```

### Token Types

| Token | Holder | Access |
|-------|--------|--------|
| **Admin Token** | Orchestrator admin | Full gateway access |
| **User Token** | Individual worker | Only their agent's dashboard |

### Security Considerations

| Concern | Mitigation |
|---------|------------|
| Token leakage | 24-byte random tokens, rotate if compromised |
| Session hijacking | HTTPS enforced via Phala gateway |
| Cross-agent access | Gateway enforces isolation per token |
| Admin exposure | Keep admin token separate from user URLs |

## Implementation

### Files to Modify

```
phala-deploy/
├── Dockerfile              # Add skill installation layer
├── entrypoint.sh           # Bootstrap orchestrator + skills config
├── docker-compose.yml      # Add GOOGLE_*, NOTION_*, ATTIO_* env vars
├── secrets/
│   └── .env.example        # Updated with all required credentials
└── orchestrator/
    └── SOUL.md             # Orchestrator persona & instructions (new)
```

### Required Credentials

| Credential | Where to Get |
|------------|--------------|
| `GOOGLE_CLIENT_ID` | Google Cloud Console → APIs & Services → Credentials |
| `GOOGLE_CLIENT_SECRET` | Same as above |
| `NOTION_API_KEY` | notion.so/my-integrations → Create integration |
| `ATTIO_API_KEY` | Attio dashboard → Settings → API |
| `REDPILL_API_KEY` | redpill.ai (existing) |
| `MASTER_KEY` | `head -c 32 /dev/urandom | base64` (existing) |

### Implementation Order

1. **Update Dockerfile** — add skill installation
2. **Update entrypoint.sh** — bootstrap orchestrator agent + skill config
3. **Update docker-compose.yml** — pass skill credentials
4. **Create orchestrator SOUL.md** — define onboarding behavior
5. **Update secrets/.env.example** — document all required vars
6. **Test locally** — docker build + run
7. **Deploy to Phala** — `phala deploy`

## Future Enhancements

- **Graduated autonomy** — per-action grants, trust levels
- **Chat channels** — add Telegram/Discord per-agent via bindings
- **Shared memory** — company knowledge base accessible to all agents
- **Role-based access** — if uniform access becomes insufficient
- **Invite link portal** — self-service landing page for onboarding
