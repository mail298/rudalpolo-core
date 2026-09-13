---
name: agent-runtime-sync
description: "Use when migrating or syncing AI agent runtime via Git."
version: 1.0.0
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [devops, agent-portability, git-backup, hermes, migration, zero-leakage]
---

# AI Agent Runtime & State Portability via Git

## Overview

Autonomous AI agent environments accumulate persistent intelligence: system configurations, specialized skill libraries, structured memories, scheduled automated cron jobs, and persona definitions. Migrating an agent to a new server or synchronizing its state across environments without starting from scratch requires disciplined version-controlled Git synchronization that guarantees **zero credential leakage** and **instant zero-setup restoration**.

---

## Core Engineering Workflows

### 1. State Separation: Portable Core vs Ephemeral Runtime

Before synchronizing the agent directory (e.g. `~/.hermes`), strictly segregate portable state from local host-specific runtime artifacts:

| Category | Components | Sync Status |
| :--- | :--- | :--- |
| **Brain & Memory** | `SOUL.md`, `config.yaml`, `memories/`, `skills/`, `cron/jobs.json` | ✅ **Track in Git** |
| **Templates** | `.env.example`, `README.md` | ✅ **Track in Git** |
| **Secrets & Keys** | `.env`, `auth.json`, `shared/`, `*.key`, `*.token`, `*.pem`, `install_id` | ❌ **Strictly Ignore** |
| **Databases & State** | `*.db`, `*.db-wal`, `*.db-shm`, `state/`, `sessions/`, `terminal-sessions/` | ❌ **Strictly Ignore** |
| **Caches & Binaries** | `bin/`, `cache/`, `logs/`, `audio_cache/`, `image_cache/`, `*.sock`, `*.pid` | ❌ **Strictly Ignore** |

---

### 2. Zero-Leakage Git Configuration (`.gitignore`)

Always place an airtight `.gitignore` in the agent root before the initial commit:

```gitignore
# Credentials, secrets, and environment tokens
.env*
!.env.example
auth.json
shared/
*.key
*.token
*.pem
install_id*
.install_id*

# Compiled binaries and machine-specific tools
bin/

# SQLite databases, sockets, locks, and live process state
*.db
*.db-*
*.lock
*.sock
*.pid
state/
sessions/
terminal-sessions/
kanban/
sandboxes/
runtime/
pending_messages/
pairing/
lsp/
.curator_backups/

# Logs, temporary cache, and binary artifacts
logs/
cache/
audio_cache/
image_cache/
*.log
*.etag
models_dev_cache.json
ollama_cloud_models_cache.json
provider_models_cache.json
processes.json
gateway_state.json
channel_directory.json
.hermes_history
.update_check
cron/executions.db
cron/output/
cron/ticker_*
cron/.tick.lock

# Backup snapshots
*.bak*
```

---

### 3. Sanitized Environment Templating (`.env.example`)

Scaffold a clean `.env.example` file exposing every required configuration key with placeholder values:

```bash
# AI Model Proxy / Gateway
HERMES_CUSTOM_9ROUTER_WN_CDN_NET_API_KEY=your_api_key_here

# Messaging & Bot Channels
TELEGRAM_BOT_TOKEN=your_telegram_bot_token_here
TELEGRAM_ALLOWED_USERS=your_telegram_user_id_here
TELEGRAM_HOME_CHANNEL=your_telegram_chat_id_here

# Productivity Integrations
TODOIST_API_TOKEN=your_todoist_api_token_here
NOTION_API_KEY=your_notion_api_key_here
NOTION_API_TOKEN=your_notion_api_token_here
```

---

### 4. Zero-Setup Target Server Restoration Procedure

On any new VPS or container environment:

1. **Install Runtime:**
   ```bash
   curl -fsSL https://raw.githubusercontent.com/nousresearch/hermes-agent/main/install.sh | bash
   ```
2. **Clone Private Core Repository:**
   ```bash
   git clone git@github.com:mail298/rudalpolo-core.git ~/.hermes
   ```
3. **Hydrate Secrets:**
   ```bash
   cp ~/.hermes/.env.example ~/.hermes/.env
   # Populate tokens in ~/.hermes/.env
   ```
4. **Launch Agent Gateway:**
   ```bash
   hermes gateway start
   ```

The agent initializes immediately with full conversational memory, standing user preferences, scheduled tasks, and specialized skill libraries intact.
