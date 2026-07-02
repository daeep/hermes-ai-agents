# Hermes AI Agents

Autonomous AI agent deployment for task automation, infrastructure monitoring, code review, and research. Powered by [Hermes Agent](https://github.com/NousResearch/hermes-agent) (Nous Research).

---

## Overview

5 autonomous AI agents running 24/7 across bare-metal and Kubernetes:

```
┌─────────────────────────────────────────────┐
│              HERMES AI AGENTS                │
│                                              │
│  Bare-Metal (2)         Kubernetes (3)       │
│  ┌──────────┐          ┌──────────────┐     │
│  │ M5 Max   │          │ antonio      │     │
│  │ Primary  │          │ Code review  │     │
│  │ Agent    │          │ Research     │     │
│  └──────────┘          └──────────────┘     │
│  ┌──────────┐          ┌──────────────┐     │
│  │ M4 Pro   │          │ edith        │     │
│  │ Dev      │          │ Web dev      │     │
│  │ Agent    │          │ Design       │     │
│  └──────────┘          └──────────────┘     │
│                         ┌──────────────┐     │
│                         │ prueba       │     │
│                         │ Testing      │     │
│                         │ Experiments  │     │
│                         └──────────────┘     │
│                                              │
│  Claude Code Agents (on-demand via SSH)      │
│  ┌──────────────┐  ┌──────────────┐         │
│  │ designer     │  │ video        │         │
│  │ Web/UI dev   │  │ Video/Media  │         │
│  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────┘
```

---

## Agent Profiles

### Bare-Metal Agents

| Agent | Host | Role | Key Tasks |
|-------|------|------|-----------|
| **Primary** | M5 Max (64GB) | Infrastructure, DevOps | K8s management, monitoring, GitOps, performance testing |
| **Dev** | M4 Pro (64GB) | Development | Code generation, CI/CD, research, deployment |

### Kubernetes Agents

| Agent | Namespace | Role | Key Tasks |
|-------|-----------|------|-----------|
| **antonio** | `hermes-antonio` | Code Review | PR review, quality gates, security scanning |
| **edith** | `hermes-edith` | Web Development | Frontend/backend work, UI design, content |
| **prueba** | `hermes-prueba` | Testing | Experiments, new tool testing, prototyping |

### Claude Code Agents

| Agent | Purpose | Access |
|-------|---------|--------|
| **designer** | Web UI/UX development | SSH tunnel + web preview |
| **video** | Video processing & media | SSH tunnel + web preview |

---

## Custom Skills

30+ custom skills extending agent capabilities:

| Category | Skills | Examples |
|----------|--------|----------|
| **DevOps** | 8 skills | K8s management, OPNsense, Talos cluster, CNPG, Longhorn |
| **MLOps** | 6 skills | vLLM serving, EXO cluster, llama.cpp, HuggingFace Hub |
| **Monitoring** | 4 skills | Prometheus/Grafana dashboards, kernel panic diagnosis |
| **Automation** | 5 skills | n8n administration, cron jobs, web scraping, embeddings |
| **Apple Ecosystem** | 5 skills | iMessage, Notes, Reminders, FindMy, macOS admin |
| **Creative** | 4 skills | ASCII art, architecture diagrams, Excalidraw, music gen |

---

## Use Cases

### 1. Automated Code Review
Agents review PRs, run security scans, enforce quality gates, and suggest improvements before human review.

### 2. Infrastructure Monitoring
Agents query Prometheus/Grafana APIs, analyze trends, detect anomalies, and suggest remediation — without manual dashboard checking.

### 3. RAG Pipeline Automation
Agents scrape web sources (forums, docs), store in PostgreSQL + pgvector, generate embeddings, and enable semantic search.

### 4. GitOps Workflow
Agents create commits, open PRs, validate Flux manifests, and manage the full GitOps lifecycle.

### 5. Research & Analysis
Agents perform multi-source research, synthesize findings, and produce structured reports.

---

## Integration Points

| System | Integration |
|--------|------------|
| **Kubernetes** | Direct API access, Flux GitOps |
| **Prometheus/Grafana** | Metric queries, dashboard inspection |
| **PostgreSQL (CNPG)** | Query execution, schema management |
| **Git (Forgejo)** | PR creation, review, merge |
| **n8n** | Workflow triggering, data pipelines |
| **DGX Spark (vLLM)** | Model inference, embeddings generation |
| **SSH** | Remote execution across all nodes |

---

## Tech Stack

- **Framework:** Hermes Agent (Python, open-source, Nous Research)
- **LLM Backend:** DeepSeek V4 Pro (primary), DeepSeek V4 Flash (light tasks)
- **Inference:** Self-hosted vLLM on DGX Spark (Gemma 4 27B)
- **Embeddings:** Qwen3-Embedding-0.6B (self-hosted)
- **Memory:** SQLite + vector search (FTS5)
- **Gateway:** Multi-platform messaging (Telegram, Discord, Slack-ready)
- **Deployment:** K8s (Flux GitOps), bare-metal (systemd)
