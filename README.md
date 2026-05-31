<p align="center">
  <img src="./img/dispatcher.svg" width="100%">
</p>
<div align="center">

  **Self-hosted incident management. API-first. No SaaS. No noise.**

[![License: AGPL v3](https://img.shields.io/badge/License-AGPL_v3-blue.svg)](https://www.gnu.org/licenses/agpl-3.0)
[![Go](https://img.shields.io/badge/Go-1.26-00ADD8?logo=go)](https://golang.org)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-18-336791?logo=postgresql)](https://postgresql.org)
[![Status](https://img.shields.io/badge/Status-In_Development-orange)]()
[![GitLab](https://img.shields.io/badge/Code%20on-GitLab-FC6D26?logo=gitlab)](https://gitlab.com/dispatcher-api/dispatcher)

 ⭐ **Star this repo** if you find it useful — and follow the project on **[GitLab](https://gitlab.com/dispatcher-api/dispatcher)** where the actual code lives.

</div>

---

## What is Dispatcher?

Dispatcher is a middleware service between your monitoring systems and your on-call engineers.

It takes a chaotic stream of raw alerts and turns them into **one actionable incident**, routes it to **the right person**, and **automatically escalates** if they don't respond — all without manager involvement.

```mermaid
flowchart LR
    P[Prometheus] --> D[Dispatcher]
    Z[Zabbix] --> D
    S[Sentry] --> D

    D --> A{"ACK?"}

    A -->|yes| E[On-call]
    A -->|timeout| N[Escalation]

    E --> R[Resolved]

    %% styles
    classDef node fill:#161b22,stroke:#30363d,color:#e6edf3,stroke-width:1px;
    classDef action fill:#0d1117,stroke:#238636,color:#e6edf3,stroke-width:1px;
    classDef danger fill:#0d1117,stroke:#da3633,color:#e6edf3,stroke-width:1px;
    classDef decision fill:#0d1117,stroke:#1f6feb,color:#e6edf3,stroke-width:1px;

    class P,Z,S,D node;
    class E,R action;
    class N danger;
    class A decision;
```

### Without Dispatcher

> 14:00 — database goes down. Prometheus fires 50 alerts.  
> Engineer Bob gets 50 notifications.  
> He spends 10 minutes figuring out it's the same problem.

### With Dispatcher

> 14:00:15 — 50 alerts arrive. One fingerprint. **One incident #INC-2041.**  
> One notification to the on-call.  
> 14:05:00 — no response? Auto-escalated to the next engineer.  
> 14:06:00 — acknowledged. All timestamps recorded.

**50 alerts → 1 incident → 1 notification. Automatic escalation. Accurate reports.**

---

## 📦 Where is the code?

> **This GitHub repo is the project's public page and mirror.**  
> All development happens on **[GitLab →](https://gitlab.com/YOUR_USERNAME/dispatcher)**

| What | Where |
|---|---|
| ⭐ Stars & visibility | **GitHub** (you're here) |
| 🔧 Source code | **[GitLab](https://gitlab.com/dispatcher-api/dispatcher)** |
| 🐛 Issues & roadmap | **[GitLab Issues](https://gitlab.com/dispatcher-api/dispatcher/-/boards)** |
| 💬 Community | **[Discord]** |

---

## Core Features

| Feature | Description |
|---|---|
| **Alert Grouping** | Groups alerts by fingerprint hash (source + server + cluster). 50 alerts become 1 incident |
| **Smart Routing** | Checks the on-call schedule in real time and notifies the right engineer |
| **Auto-Escalation** | If no acknowledgment in 5 minutes — escalates to the next person in the chain |
| **MTTA / SLA Tracking** | Records every timestamp. Reports show exactly who responded when |
| **On-Call Schedules** | Monthly rotations, shift assignments, fingerprint-based responsibility splitting |
| **Multi-source** | Prometheus, Zabbix, Sentry — any system that can send a webhook |

---

## Tech Stack

```mermaid
flowchart TD
    A["API (Go)<br/>REST + WebSocket"] 
    
    A --> B["PostgreSQL<br/>Incidents · Users · Schedules"]
    A --> C["Redis<br/>Dedup Lock"]
    A --> D["RabbitMQ<br/>Escalation Queue"]

    %% styles
    classDef main fill:#111827,stroke:#3b82f6,color:#f9fafb,stroke-width:1px;
    classDef db fill:#161b22,stroke:#30363d,color:#e5e7eb,stroke-width:1px;
    classDef queue fill:#161b22,stroke:#ef4444,color:#e5e7eb,stroke-width:1px;

    class A main;
    class B,C db;
    class D queue;
```

- **Go** — API server and escalation consumer
- **PostgreSQL** — incidents, users, schedules, reports
- **Redis** — SETNX deduplication lock (prevents race conditions on parallel alerts)
- **RabbitMQ** — delayed escalation queue with `x-delay` plugin (precise 5-min timers, survives restarts)
- **WebSocket** — real-time broadcast to all connected dashboard clients

---

## How It Works

### 1. Alert arrives

```http
POST /v1/alerts
Content-Type: application/json

{
  "alerts": [{
    "source": "prometheus",
    "name": "HighCPU",
    "severity": "critical",
    "labels": { "server": "db-master-01", "cluster": "prod" },
    "annotations": { "description": "CPU usage is 95% on db-master-01" },
    "timestamp": "2026-04-18T14:00:15Z"
  }]
}
```

### 2. Fingerprint computed

```
fingerprint = SHA256("prometheus" + "db-master-01" + "prod")
```

`name` and `timestamp` are intentionally excluded — HighCPU and DiskFull on the same server is **one problem**.

### 3. Deduplication lock (Redis SETNX)

```
SETNX lock:fingerprint:<hash> 1 EX 30
```

Prevents two parallel alerts from creating two incidents.

### 4. Incident created or updated

- **New fingerprint** → create incident, notify on-call, queue escalation in RabbitMQ
- **Existing fingerprint** → increment `alert_count`, broadcast WebSocket update

### 5. Escalation consumer

RabbitMQ delivers the message after exactly 5 minutes:

```python
if incident.status == "acknowledged": return  # Engineer responded in time
if incident.status == "resolved":    return  # Already closed
# Escalate to next step
notify(next_user, incident)
queue_next_escalation(incident, step + 1)
```

---

## Incident Lifecycle

```mermaid
flowchart LR
    A[POST /v1/alerts] --> O[OPEN]

    O --> C{"ACK?"}

    C -->|yes| K[ACKNOWLEDGED]
    C -->|timeout| E[ESCALATED]

    K --> I[Investigation]
    I --> R[RESOLVED]

    %% styles
    classDef node fill:#161b22,stroke:#30363d,color:#e6edf3,stroke-width:1px;
    classDef action fill:#0d1117,stroke:#238636,color:#e6edf3,stroke-width:1px;
    classDef danger fill:#0d1117,stroke:#da3633,color:#e6edf3,stroke-width:1px;
    classDef decision fill:#0d1117,stroke:#1f6feb,color:#e6edf3,stroke-width:1px;

    class A,O,I node;
    class K,R action;
    class E danger;
    class C decision;
```

> Transition from `resolved` back to `open` is not allowed.  
> If the problem returns — a new incident is created.

---

## Getting Started

### Prerequisites

- Docker & Docker Compose
- Go 1.26+ (for local development)

### Run with Docker Compose

```
*coming soon*
```

> Full setup guide and Docker Compose file are in the **[GitLab repo](https://gitlab.com/dispatcher-api/dispatcher)**.

---

## API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/v1/alerts` | Ingest alerts from monitoring systems |
| `GET` | `/v1/incidents` | List incidents (filterable by status, severity) |
| `GET` | `/v1/incidents/{id}` | Get incident details |
| `POST` | `/v1/incidents/{id}/acknowledge` | Acknowledge an incident |
| `POST` | `/v1/incidents/{id}/resolve` | Resolve an incident |
| `GET` | `/v1/schedules/now` | Get current on-call engineer |
| `POST` | `/v1/schedules` | Create a shift |
| `GET` | `/v1/reports/sla` | SLA and MTTA report data |
| `WS` | `/ws` | WebSocket connection for live updates |

---

## 🚀 Beta Program

The first **10 beta testers** will receive **3 months of free access** to the full API + UI after MVP launch.

To apply: *[form link — coming soon]*

---

## License

Dispatcher core is licensed under **[AGPL-3.0](LICENSE)**.

- ✅ Free to self-host
- ✅ Open source forever
- 💼 Paid UI coming with the full MVP

---

## Contributing

Issues, ideas, and PRs are welcome — **[open them on GitLab](https://gitlab.com/dispatcher-api/dispatcher/-/boards)**.

- **Bugs** → [GitLab Issues](https://gitlab.com/dispatcher-api/dispatcher/-/boards)
- **Feature ideas** → [Discord] → `#💡-feature-requests`
- **Questions** → Discord `#❓-help`

---

<div align="center">

Built by **Oshan** · Solo developer · Open-source believer

*Let's build the best self-hosted incident management. Together.* 🚀

**[→ GitLab repo with full source code](https://gitlab.com/dispatcher-api/dispatcher)**

</div>
