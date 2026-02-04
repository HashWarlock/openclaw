# Multi-Agent Deployment Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Transform phala-deploy into a multi-agent system with orchestrator-based onboarding via Gmail OAuth

**Architecture:** Single CVM with orchestrator agent that creates worker agents per-user. All agents share skills (gog, notion, attio-crm) installed at build time. Token-based routing gives each user their own dashboard.

**Tech Stack:** Docker, OpenClaw gateway, clawhub skills, Gmail OAuth via gog skill

**Design doc:** `docs/plans/2026-02-04-multi-agent-deployment-design.md`

---

## Task 1: Update secrets/.env.example with skill credentials

**Files:**
- Modify: `phala-deploy/secrets/.env.example`

**Step 1: Add skill credential placeholders**

Replace the entire file with:

```env
# Master key — derives rclone crypt passwords + gateway auth token via HKDF
# Generate with: head -c 32 /dev/urandom | base64
MASTER_KEY=

# Redpill AI provider (get from redpill.ai)
REDPILL_API_KEY=

# --- Skills credentials ---

# Google Workspace (for gog skill)
# Get from: Google Cloud Console → APIs & Services → Credentials → OAuth 2.0 Client
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Notion (for notion skill)
# Get from: notion.so/my-integrations → Create integration
NOTION_API_KEY=

# Attio CRM (for attio-crm skill)
# Get from: Attio dashboard → Settings → API
ATTIO_API_KEY=

# --- Encrypted S3 storage (optional — omit S3_BUCKET to use local volume) ---
S3_BUCKET=
S3_ENDPOINT=
S3_PROVIDER=Cloudflare
S3_REGION=auto
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

**Step 2: Commit**

```bash
git add phala-deploy/secrets/.env.example
git commit -m "chore(phala-deploy): add skill credentials to .env.example"
```

---

## Task 2: Update docker-compose.yml with skill environment variables

**Files:**
- Modify: `phala-deploy/docker-compose.yml`

**Step 1: Add skill credential env vars**

Add these lines after the existing environment variables (after line 34, before `restart:`):

```yaml
      # Skills credentials
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID:-}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET:-}
      NOTION_API_KEY: ${NOTION_API_KEY:-}
      ATTIO_API_KEY: ${ATTIO_API_KEY:-}
```

The full environment section should now be:

```yaml
    environment:
      OPENCLAW_STATE_DIR: /data/openclaw
      NODE_ENV: production
      # Master key — derives rclone crypt passwords + gateway auth token via HKDF
      MASTER_KEY: ${MASTER_KEY:-}
      # Redpill AI provider
      REDPILL_API_KEY: ${REDPILL_API_KEY:-}
      # Encrypted S3 storage (optional — omit S3_BUCKET to use local volume)
      S3_BUCKET: ${S3_BUCKET:-}
      S3_PREFIX: ${S3_PREFIX:-openclaw-state}
      S3_ENDPOINT: ${S3_ENDPOINT:-}
      S3_REGION: ${S3_REGION:-us-east-1}
      S3_PROVIDER: ${S3_PROVIDER:-Other}
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID:-}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY:-}
      # Override derived crypt passwords (optional — only if not using MASTER_KEY)
      RCLONE_CRYPT_PASSWORD: ${RCLONE_CRYPT_PASSWORD:-}
      RCLONE_CRYPT_PASSWORD2: ${RCLONE_CRYPT_PASSWORD2:-}
      # Skills credentials
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID:-}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET:-}
      NOTION_API_KEY: ${NOTION_API_KEY:-}
      ATTIO_API_KEY: ${ATTIO_API_KEY:-}
    restart: unless-stopped
```

**Step 2: Commit**

```bash
git add phala-deploy/docker-compose.yml
git commit -m "chore(phala-deploy): add skill credentials to docker-compose.yml"
```

---

## Task 3: Update Dockerfile to install skills

**Files:**
- Modify: `phala-deploy/Dockerfile`

**Step 1: Add skill installation layer**

Add this new layer after the openclaw install (after line 52, before "Layer 6: Shell environment"):

```dockerfile
# Layer 6: Install shared skills via clawhub
# Skills are installed to /root/.openclaw/skills (global location)
ENV OPENCLAW_STATE_DIR=/root/.openclaw
RUN npx clawhub install steipete/gog && \
    npx clawhub install steipete/notion && \
    npx clawhub install kesslerio/attio-crm && \
    rm -rf /root/.npm /tmp/*
```

**Note:** The existing "Layer 6: Shell environment" becomes "Layer 7".

**Step 2: Update subsequent layer comments**

Change line 54-56 from:
```dockerfile
# Layer 6: Shell environment
```

To:
```dockerfile
# Layer 7: Shell environment
```

**Step 3: Commit**

```bash
git add phala-deploy/Dockerfile
git commit -m "feat(phala-deploy): install gog, notion, attio-crm skills in Docker image"
```

---

## Task 4: Update entrypoint.sh to bootstrap orchestrator agent

**Files:**
- Modify: `phala-deploy/entrypoint.sh`

**Step 1: Replace the bootstrap config section**

Replace lines 123-173 (the entire "Bootstrap config if none exists" section) with the following expanded version:

```sh
# Bootstrap config if none exists — generates full Redpill provider + model catalog + orchestrator agent
CONFIG_FILE="$STATE_DIR/openclaw.json"
if [ ! -f "$CONFIG_FILE" ]; then
  BOOT_TOKEN="${GATEWAY_AUTH_TOKEN:-$(head -c 32 /dev/urandom | base64 | tr -d '/+=' | head -c 32)}"
  export CONFIG_FILE BOOT_TOKEN
  export GOOGLE_CLIENT_ID GOOGLE_CLIENT_SECRET NOTION_API_KEY ATTIO_API_KEY
  node -e "
    // Import Redpill config functions from installed openclaw package
    const PKG = '/usr/lib/node_modules/@h4x3rotab/openclaw/dist';
    const { applyRedpillConfig } = require(PKG + '/commands/onboard-auth.config-core.js');

    const base = {
      gateway: {
        mode: 'local',
        bind: 'lan',
        auth: {
          token: process.env.BOOT_TOKEN,
          userTokens: {}  // Will be populated as workers are onboarded
        },
        controlUi: { dangerouslyDisableDeviceAuth: true },
      },
      update: { checkOnStart: false },
      agents: {
        list: [
          {
            id: 'orchestrator',
            name: 'Orchestrator',
            default: true,
            workspace: '/data/openclaw/workspaces/orchestrator',
            identity: {
              name: 'Company Hub',
              personality: 'Helpful admin assistant that onboards users and coordinates work across the organization'
            },
            subagents: {
              allowAgents: ['*']
            }
          }
        ],
        defaults: {
          memorySearch: {
            provider: 'openai',
            model: 'qwen/qwen3-embedding-8b',
            remote: { baseUrl: 'https://api.redpill.ai/v1', apiKey: process.env.REDPILL_API_KEY || undefined },
            fallback: 'none',
          },
        },
      },
      skills: {
        installed: {
          'steipete/gog': {
            enabled: true,
            config: {
              clientId: process.env.GOOGLE_CLIENT_ID || undefined,
              clientSecret: process.env.GOOGLE_CLIENT_SECRET || undefined
            }
          },
          'steipete/notion': {
            enabled: true,
            config: {
              apiKey: process.env.NOTION_API_KEY || undefined
            }
          },
          'kesslerio/attio-crm': {
            enabled: true,
            config: {
              apiKey: process.env.ATTIO_API_KEY || undefined
            }
          }
        }
      }
    };

    // Apply full Redpill provider config (model catalog + default model)
    let cfg = applyRedpillConfig(base);

    // Override default model to Kimi K2.5
    cfg.agents.defaults.model.primary = 'redpill/moonshotai/kimi-k2.5';

    // Inject Redpill API key if provided
    if (process.env.REDPILL_API_KEY) {
      cfg.models.providers.redpill.apiKey = process.env.REDPILL_API_KEY;
    }

    require('fs').writeFileSync(process.env.CONFIG_FILE, JSON.stringify(cfg, null, 2));
  " 2>&1 || {
    # Fallback: write minimal config if node import fails (e.g. package structure changed)
    echo "Warning: full config generation failed, writing minimal config."
    cat > "$CONFIG_FILE" <<CONF
{"gateway":{"mode":"local","bind":"lan","auth":{"token":"$BOOT_TOKEN","userTokens":{}},"controlUi":{"dangerouslyDisableDeviceAuth":true}},"update":{"checkOnStart":false},"agents":{"list":[{"id":"orchestrator","name":"Orchestrator","default":true,"workspace":"/data/openclaw/workspaces/orchestrator","identity":{"name":"Company Hub","personality":"Helpful admin assistant"},"subagents":{"allowAgents":["*"]}}],"defaults":{"memorySearch":{"provider":"openai","model":"qwen/qwen3-embedding-8b","remote":{"baseUrl":"https://api.redpill.ai/v1"},"fallback":"none"}}}}
CONF
  }
  echo "Created config at $CONFIG_FILE (token: ${GATEWAY_AUTH_TOKEN:+derived}${GATEWAY_AUTH_TOKEN:-random})"

  # Create orchestrator workspace directory
  mkdir -p "$STATE_DIR/workspaces/orchestrator"
fi
```

**Step 2: Commit**

```bash
git add phala-deploy/entrypoint.sh
git commit -m "feat(phala-deploy): bootstrap orchestrator agent with skills config"
```

---

## Task 5: Create orchestrator SOUL.md

**Files:**
- Create: `phala-deploy/orchestrator/SOUL.md`

**Step 1: Create the directory and file**

```bash
mkdir -p phala-deploy/orchestrator
```

**Step 2: Write the SOUL.md content**

Create `phala-deploy/orchestrator/SOUL.md`:

```markdown
# Orchestrator Agent

You are the Company Hub — an admin assistant that manages worker agents for the organization.

## Your Responsibilities

### 1. Onboarding New Users

When someone requests an agent (either directly or via admin instruction):

1. **Collect their email** — ask for their work Gmail address
2. **Generate invite link** — create a unique onboarding URL with their pending agent ID
3. **Guide OAuth flow** — help them complete Gmail authentication via the `gog` skill
4. **Create their agent** — once OAuth completes:
   - Generate a unique agent ID from their email (e.g., `alice@company.com` → `alice-company-com`)
   - Generate a unique dashboard token
   - Add their agent to the config
   - Send them their dashboard URL

### 2. Agent Coordination

- Help workers find each other when they need cross-team information
- Broadcast important announcements to all workers
- Escalate issues that workers cannot resolve

### 3. Managing Shared Resources

All workers have access to these skills (you do too):
- **gog** — Gmail, Calendar, Drive, Docs, Sheets, Contacts
- **notion** — Company wiki and databases
- **attio-crm** — Customer and deal information

## Onboarding Flow

### User Request Flow

When a user messages you asking for an agent:

```
User: "I'd like to get an agent set up"

You: "Great! I'll help you get set up. First, what's your work email address?"

User: "alice@company.com"

You: "Perfect! I'll create an agent for you. Please complete the Gmail
authentication by clicking this link: [generate OAuth link via gog skill]

Once you've completed that, I'll send you your personal dashboard URL."

[After OAuth completes]

You: "You're all set! Your personal agent dashboard is at:
https://<cvm>-18789.<gateway>.phala.network?token=<ALICE_TOKEN>

Bookmark this link — it's your private portal to your agent."
```

### Admin-Initiated Flow

When an admin tells you to onboard someone:

```
Admin: "Please onboard alice@company.com"

You: "I'll send Alice an invite. One moment..."

[Send invite email via gog skill]

You: "Done! I've sent an onboarding invitation to alice@company.com.
I'll let you know when she completes the setup."

[After Alice completes OAuth]

You: "Alice has completed onboarding! Her agent is now active."
```

## Creating a New Agent

To add a new worker agent, you need to update the gateway config:

1. Generate a unique token: 24 random bytes, base64url encoded
2. Create agent entry with:
   - `id`: email converted to slug (dots → hyphens, @ → hyphen)
   - `name`: User's name + "'s Agent"
   - `workspace`: `/data/openclaw/workspaces/<agent-id>`
   - `identity`: Personal assistant personality
   - `subagents.allowAgents`: `["orchestrator", "*"]`
3. Add token mapping to `gateway.auth.userTokens`
4. Create their workspace directory

## Human Approval

All workers (including you) follow "always ask" — propose actions, wait for human approval before executing. This includes:
- Sending emails
- Creating/editing Notion pages
- Updating Attio records
- Messaging other agents
- Scheduling calendar events

Read-only operations (search, read) do not require approval.

## Communication

You can message any worker agent using `sessions_send`. Workers can message you to escalate issues or request admin help.

Keep coordination efficient — only involve humans when decisions need to be made.
```

**Step 3: Commit**

```bash
git add phala-deploy/orchestrator/SOUL.md
git commit -m "feat(phala-deploy): add orchestrator SOUL.md with onboarding instructions"
```

---

## Task 6: Update Dockerfile to copy orchestrator SOUL.md

**Files:**
- Modify: `phala-deploy/Dockerfile`

**Step 1: Add COPY instruction for orchestrator files**

Add this line after the entrypoint COPY (after line 71, before `ENV NODE_ENV=production`):

```dockerfile
# Copy orchestrator persona
COPY phala-deploy/orchestrator/ /opt/openclaw/orchestrator/
```

**Step 2: Update entrypoint.sh to use the bundled SOUL.md**

In `entrypoint.sh`, add this after the "Create orchestrator workspace directory" line:

```sh
  # Copy orchestrator persona files
  if [ -d /opt/openclaw/orchestrator ]; then
    cp -r /opt/openclaw/orchestrator/* "$STATE_DIR/workspaces/orchestrator/" 2>/dev/null || true
  fi
```

**Step 3: Commit**

```bash
git add phala-deploy/Dockerfile phala-deploy/entrypoint.sh
git commit -m "feat(phala-deploy): bundle orchestrator SOUL.md in Docker image"
```

---

## Task 7: Update README.md with multi-agent documentation

**Files:**
- Modify: `phala-deploy/README.md`

**Step 1: Add multi-agent section**

Add this new section after "## What's next" (before "## How S3 storage works"):

```markdown
## Multi-agent setup

This deployment runs a multi-agent system with an **orchestrator** that can onboard new users.

### How it works

1. **Orchestrator agent** — the default agent, handles onboarding and coordination
2. **Worker agents** — one per user, created via Gmail OAuth
3. **Shared skills** — all agents can use gog (Google), notion, and attio-crm

### Onboarding new users

**User-initiated:** User messages the orchestrator asking for an agent. Orchestrator guides them through Gmail OAuth, then creates their agent and sends them a dashboard URL.

**Admin-initiated:** Admin tells orchestrator "onboard alice@company.com". Orchestrator sends an invite email, and creates the agent once Alice completes OAuth.

### Dashboard URLs

Each user gets a unique dashboard URL:

```
https://<app_id>-18789.<gateway>.phala.network?token=<USER_TOKEN>
```

- **Admin token** (from MASTER_KEY): Full access to orchestrator
- **User tokens** (generated per-user): Access only to their agent

### Required credentials

For the skills to work, you need these in your `secrets/.env`:

| Credential | Where to get it |
|------------|-----------------|
| `GOOGLE_CLIENT_ID` | Google Cloud Console → APIs & Services → Credentials |
| `GOOGLE_CLIENT_SECRET` | Same as above |
| `NOTION_API_KEY` | notion.so/my-integrations |
| `ATTIO_API_KEY` | Attio dashboard → Settings → API |

### Inter-agent communication

Agents can message each other using `sessions_send`:
- Workers can escalate to orchestrator
- Workers can request info from each other
- Orchestrator can broadcast to all workers

### Human approval

All agents follow "always ask" — they propose actions and wait for human approval before executing writes (send email, update Notion, etc.). Reads don't require approval.
```

**Step 2: Commit**

```bash
git add phala-deploy/README.md
git commit -m "docs(phala-deploy): add multi-agent setup documentation"
```

---

## Task 8: Local build and verification

**Files:** None (verification only)

**Step 1: Build the Docker image locally**

```bash
cd /Users/hashwarlock/Projects/openclaw
docker build -f phala-deploy/Dockerfile -t openclaw-cvm-multiagent:test .
```

Expected: Build completes successfully, skills install during build

**Step 2: Verify skills are installed in the image**

```bash
docker run --rm openclaw-cvm-multiagent:test ls -la /root/.openclaw/skills/
```

Expected: Shows gog, notion, attio-crm skill directories

**Step 3: Verify entrypoint generates correct config**

```bash
docker run --rm \
  -e MASTER_KEY=$(head -c 32 /dev/urandom | base64) \
  -e REDPILL_API_KEY=test \
  openclaw-cvm-multiagent:test \
  cat /data/openclaw/openclaw.json 2>/dev/null | head -100
```

Expected: JSON with `agents.list` containing orchestrator, `skills.installed` with three skills

**Step 4: Record results and commit any fixes**

If issues found, fix and re-run. Once working:

```bash
git add -A
git commit -m "test(phala-deploy): verify multi-agent Docker build works"
```

---

## Task 9: Final commit and summary

**Step 1: Create summary commit if needed**

If all previous commits succeeded, no additional commit needed. Otherwise:

```bash
git add -A
git commit -m "feat(phala-deploy): complete multi-agent deployment setup

- Add orchestrator agent bootstrapped at startup
- Pre-install gog, notion, attio-crm skills
- Add skill credentials to env configuration
- Document multi-agent setup in README
- Create orchestrator SOUL.md with onboarding instructions"
```

**Step 2: Verify git log**

```bash
git log --oneline -10
```

Expected: Series of commits for each task

---

## Deployment (Manual — after plan execution)

Once the implementation is complete and verified locally:

1. **Rebuild and push Docker image:**
   ```bash
   docker build -f phala-deploy/Dockerfile -t h4x3rotab/openclaw-cvm:multiagent .
   docker push h4x3rotab/openclaw-cvm:multiagent
   ```

2. **Update docker-compose.yml image reference** (or use local build)

3. **Add credentials to secrets/.env**

4. **Deploy to Phala:**
   ```bash
   cd phala-deploy
   phala deploy -n openclaw-multiagent -c docker-compose.yml -e secrets/.env -t tdx.medium --dev-os --wait
   ```

5. **Access orchestrator dashboard** at the generated URL with admin token
