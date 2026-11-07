# Sobat UMKM

Exported WorkBuddy workflow for **Challenge B — The Office Agent**, Tencent Cloud Indonesia
Hackathon 2026 ("AI CAN DO IT").

One agent turns a messy WhatsApp chat, voice note or receipt into a structured plan, asks a single
clarifying question when something is genuinely ambiguous, and **never sends, schedules or records
anything until a human approves the exact details**. Text inside messages and documents is treated
as data, never as instructions. Every requested / decided / proposed / refused step is written to
the audit log.

> **Requirement:** the three SobatPesan MCP services must be running on localhost before the agent
> can do anything. See [Requirements](#1-requirements).

## What is in this folder

| Path | What it is |
|---|---|
| `export/sobat-umkm.zip` | **Submission artifact** — the exported single-agent WorkBuddy expert (3 files) |
| `export/sobat-umkm/` | Unpacked copy of the same package, for review |
| `export/README.md` | Bundle manifest: agent profile, source policy, onboarding flow, evidence checklist |
| `config/workbuddy.mcp.json` | Portable MCP config template for the 3 services |
| `prompts/nightly-check.md` | Prompt for a scheduled (unattended) daily run |
| `examples/chat-7-hari/` | 89-message fictional 7-day WhatsApp log + answer key, for testing |
| `29322b52-…pdf` | Hackathon handbook (Challenge B requirements, timeline, scoring) |
| `.gitignore` | Excludes `.workbuddy-ai/`, `.DS_Store`, secrets, temp and build files |

Only **one** expert is used: `sobat-umkm` (`expertType: agent`). The earlier 4-role Expert Team
was removed. The agent performs a self-check before proposing, so the rubric's independent
secondary-validator bonus is not claimed.

## 1. Requirements

### 1a. SobatPesan MCP backend on localhost

The backend is a separate Docker Compose project, **not** part of this repository. It provides the
three MCP servers the agent calls.

**Download the Docker bundle:** [sobatpesan-1.0-bundle.zip](https://drive.google.com/file/d/175jmxjaFbbNLhZ9vduyMDlCNWCSV4Qf_/view?usp=sharing)

1. Download and extract the zip on the **same machine that runs WorkBuddy** (the MCP servers are
   localhost-only). Follow the README inside the bundle for first-run setup.
2. Start Docker Desktop, then run `docker compose up -d` from the extracted folder.
3. Verify the three health endpoints below.

If the services are not active when you summon the expert, the agent stops at **Step 0** and prints
these same recovery steps — including the download link above — instead of guessing or working from
memory.

Containers restart with Docker Desktop. Verify all three are healthy:

```bash
curl http://127.0.0.1:8101/health
curl http://127.0.0.1:8102/health
curl http://127.0.0.1:8103/health
```

| Service | MCP endpoint | Purpose |
|---|---|---|
| `sobatpesan-messaging` | `http://127.0.0.1:8101/mcp` | WhatsApp conversations, media, voice-note transcripts |
| `sobatpesan-memory` | `http://127.0.0.1:8102/mcp` | Business profile, reminders, run log, audit |
| `sobatpesan-office` | `http://127.0.0.1:8103/mcp` | Contacts, rules, documents, ledger, **`propose_plan`** |

Supporting endpoints (human UI, not MCP): `http://localhost:8103/` for plans / orders / outbox /
audit, and `http://localhost:8080` for the Bahasa WhatsApp add-number page. Postgres is on
`localhost:5434`; the agent does not connect to it directly.

All three servers are Streamable HTTP. For the local demo `MCP_AUTH_TOKEN` is empty in `.env`, so
**no headers are needed**. If you set a token, add `Authorization: Bearer <token>` to all three.

### 1b. WorkBuddy AI

WorkBuddy runs locally: the Mac must be powered on, not sleeping, and the app running at execution
time. Time-based automations also require this.

The live MCP config is **user-level**, at `~/.workbuddy-ai/mcp.json` — not inside this repo.
Working contents:

```json
{
  "mcpServers": {
    "sobatpesan-messaging": { "type": "http", "url": "http://127.0.0.1:8101/mcp" },
    "sobatpesan-memory":    { "type": "http", "url": "http://127.0.0.1:8102/mcp" },
    "sobatpesan-office":    { "type": "http", "url": "http://127.0.0.1:8103/mcp" }
  }
}
```

`config/workbuddy.mcp.json` in this repo is a portable template using `streamableHttp` and a
`${MCP_AUTH_TOKEN}` placeholder — copy/adjust it, don't assume it works verbatim.

After writing the config:

1. **Restart WorkBuddy** — a config written after launch is not picked up.
2. Open the connector page → custom connectors → **Trust** all three `sobatpesan-*` servers.
3. Start a **new conversation** — a running session's tool list is fixed at session start.

## 2. Install the expert

On this machine `sobat-umkm` is already registered in Expert Center. On a fresh machine either
import `export/sobat-umkm.zip` through the Expert Center, or copy the unpacked folder into
`~/.workbuddy-ai/plugins/marketplaces/my-experts/plugins/sobat-umkm/` and add its entry to
`~/.workbuddy-ai/plugins/marketplaces/my-experts/.codebuddy-plugin/marketplace.json`:

```json
{ "name": "sobat-umkm", "source": "./plugins/sobat-umkm", "description": "…", "version": "1.0.0" }
```

Restart WorkBuddy and re-select **Sobat UMKM**. Avatars are placeholders only (no images were
generated).

## 3. Quick test

1. Summon **Sobat UMKM** and ask `Cek hari ini`. Step 0 calls `list_rules`, `list_channels` and
   `list_runs`; if all three answer, the wiring is live. If one fails, the agent stops and prints
   the exact recovery steps instead of guessing.
2. For the full scenario, follow `examples/chat-7-hari/README.md`: paste `chat-7-hari.json` with
   its suggested prompt. The answer key lists the orders, payments, expenses, duplicate
   corrections, injected instructions and ambiguity cases the agent should surface.
3. The agent returns a `confirmation_summary` and an `approval_url`. Nothing is executed.
4. Approve or reject at `http://localhost:8103/plans/<id>`, then check `get_plan` — only
   `status: executed` means something actually happened.

## 4. Trigger model

**User request or a time-based automation only.** There is no webhook, no auto-reply and no
incoming-message trigger. Incoming WhatsApp messages are stored; the agent reads them on the next
explicit request or scheduled run. `prompts/nightly-check.md` is the scheduled-run prompt and
restricts unattended runs to reading, `remember` and `propose_plan`.

Automations are created separately in WorkBuddy (they are not part of this repo). Suggested WIB
schedules: nightly check 20:30 daily, morning brief 07:00 Mon–Sat, weekly finance 18:00 Sunday.
Scheduled runs must not land during Friday 11:30–13:00 WIB or on public holidays.

## 5. Trust boundary

`propose_plan` is the **only** path to the outside world; sending, scheduling and bookkeeping are
performed by the backend only after a human approves. Owner-facing output cites business records
(`sale_no`, `document_id`, `media_id`, `plan_id`, rule names), never tool names, connector names or
`run_id`; tool provenance goes into `record_audit(... details ...)`. Prompt-injection attempts found
inside chats or documents are reported and flagged, never obeyed.

## 6. Submission status

Required items still to add before submitting: the one-page **Accountability sheet**, **test
results with screenshots (including failures)**, an **architecture diagram with trust boundaries**,
and a **5–8 minute demo video** showing a refusal, clarification or approval wait. `export/README.md`
has the evidence checklist. A public repository URL satisfies the "source code or exported
WorkBuddy workflow with evidence of use" requirement.
