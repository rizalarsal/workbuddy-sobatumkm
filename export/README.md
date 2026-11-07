# Sobat UMKM — exported WorkBuddy workflow

Submission bundle for **Challenge B — The Office Agent**, Tencent Cloud Indonesia Hackathon 2026.

**One expert only.** On 2026-11-07 the 4-role Expert Team (`sobat-umkm-office`) was removed from
the marketplace registration, package directory, cache and export folder. `sobat-umkm.zip` is now
the sole submission artifact. Honest trade-off: the single agent performs a **self-check** before
proposing, so the rubric's independent secondary-validator bonus is no longer claimed. Everything
else — decomposition, clarification, approval gate, injected-instruction handling and audit trail —
stays.

## What is in `sobat-umkm.zip`

A valid, registered WorkBuddy **single agent** (`expertType: agent`), 3 files:
`.codebuddy-plugin/plugin.json`, `README.md`, `agents/sobat-umkm.md`.
`export/sobat-umkm/` is an unpacked copy of the same package.

One Bahasa Indonesia agent runs the whole flow:

`Step-0 pre-flight → read once → one draft → self-check → propose_plan → show approval_url → stop`

It binds all three MCP services through inline `mcpServers:` frontmatter
(`sobatpesan-messaging` :8101, `sobatpesan-memory` :8102, `sobatpesan-office` :8103 — those are the
live Docker service names, not branding). Numbers-only questions produce no draft. The agent has
**no capability to send, schedule, record or change anything**: `propose_plan` is the only path to
the outside world, a human approves at `http://localhost:8103/plans/<id>`, and only
`get_plan: executed` counts as done.

`validate_expert.py` → valid, with one advisory warning (`defaultInitPrompt` intentionally differs
from `quickPrompts[0]`). Registered in `my-experts/.codebuddy-plugin/marketplace.json`.

## Expert Center profile

| Editor field | Value | `plugin.json` field |
|---|---|---|
| Name | Sobat UMKM | `displayName.en` |
| Profession (big heading) | Sobat UMKM | `profession.en` / `profession.zh` |
| Alias | Amir Sobat UMKM | no alias field exists → `keywords` + package README |
| Category | Operations (`09-OperationsHR`) | `categoryId` |
| About | full Bahasa Indonesia description | `displayDescription.en` (verbatim), `description` |
| Expertise (3) | WhatsApp · Nota, faktur & formulir · Setuju dulu, baru jalan | `tags` |
| Try asking me (3) | cek pesan hari ini · cek total nota · omzet & piutang | `quickPrompts` |
| Default first message | "Halo, saya pemilik usaha rumahan. Kenalan dulu dan bantu saya mulai." | `defaultInitPrompt` |

The spec has 12 fixed categories including `08-FinanceInvestment` but **no Small Business**, so
Operations was kept. Internal id, directory, agent id, manifest, registration and zip are all
`sobat-umkm`.

## Source policy — what the owner sees

An early answer cited `dari get_finance_summary` four times. Three lines literally asked the agent
to name the tool, and the source whitelist listed only message/document ids, so a finance aggregate
had nothing legal to cite. A source is now defined as **a business record the owner can reopen**,
never a function name.

| Owner-facing source | Example |
|---|---|
| message / media / document id | `id pesan 42`, `media_id …`, `document_id …` |
| business records | `sale_no 1`, `sale_id …`, `sku TP-001`, `invoice_number …`, `task_id …` |
| operational rule | "salat Jumat 11.30–13.00 WIB" from `list_rules` |
| aggregate with no single record | period + record count ("buku 2–6 Nov WIB, 1 pesanan / 3 baris item") |

Tool names, connector names, `run_id` and SQL table names are banned from the owner summary; they
belong in `record_audit(... details ...)` as audit provenance. Numbers-only answers do not append
the plan footer "Belum ada yang dikirim…".

## Business profile & onboarding

No business type is assumed. Every bakery word was removed; owner name, business name and products
come from `recall(scope="business", tags=["profil"])`. `http://localhost:8080` is the Bahasa
add-number page; `/channels` is a JSON endpoint, not a page. The `/channels/<id>/pair` deep link
stays because it is a real HTML page.

The `remember` call must include `tags=["profil"]` — `recall` requires all tags to be present, so
without it the profile would never be found in later sessions.

## Opening-message flow (final)

An early first run produced: *"Belum ada profil usaha yang tersimpan, dan belum ada kanal WhatsApp
yang terhubung. Mari saya kirim pesan pembuka."* A literal-phrase ban failed because the model
**paraphrased** it. The final design has no state report at all.

**Pertama kali (no profile)** — send exactly this, nothing else, then stop. No WhatsApp line, no
reminders, no work offer:

```
Halo! Saya Sobat, asisten admin untuk usaha Anda. Boleh kenalan dulu supaya saya menyapa dan mencatat dengan benar? Cukup satu baris: panggilan Anda (Ibu/Bapak/Kak + nama), nama usaha, dan apa yang Anda jual.
```

Exception: if the owner's first message already contains work (pasted chat, receipt photo, numbers
question), do the work first, then close with the introduction.

**Setelah pemilik menjawab** — save with
`remember(scope="business", kind="fact", subject="profil usaha", tags=["profil"], content="<panggilan>; <nama usaha>; <yang dijual>")`, then reply in at most three lines: confirmation in the owner's own
words; **only if no `CONNECTED` channel**, the optional `http://localhost:8080` add-number sentence
("Tidak wajib: chat dan foto nota juga bisa langsung ditempel di sini"); then a concrete offer, not
an open question.

**Sesi berikutnya** — one-line greeting with nickname + business, today's `list_due` content in one
sentence (not the tool), the WhatsApp line only if still unconnected, then the concrete offer.

Owner text may never contain profil / memori / kanal / status / tersimpan / tool names, nor announce
what the agent will do or found. The wrong sentence is quoted verbatim in the prompt.

## Scheduled-run prompt review

`prompts/nightly-check.md` was checked against the live MCP tool inventory (39 tools) and
rewritten. Four schema bugs would have failed a real run: `get_conversation` needs
`conversation_id`; `mark_processed` needs `through_seq`; WhatsApp photos should use `media_id` not
base64; `propose_plan` needs `actions`. Also resolved: "only read / `remember` / `propose_plan`"
vs internal bookkeeping calls, the connector-name ban vs the Step-0 error message, `request_text`
must be the source message (a scheduled run has no owner utterance), explicit ISO `due_before`,
`check_schedule(start=…, attendees=[…])`, and the "always propose" exception when an action cannot
be built without guessing.

## Supporting files (this workspace)

| File | Purpose |
|---|---|
| `README.md` (repo root) | Setup requirements, localhost MCP install, quick test |
| `config/workbuddy.mcp.json` | Portable MCP config template for the 3 services |
| `prompts/nightly-check.md` | Scheduled-run user prompt |
| `examples/chat-7-hari/` | 89-message 7-day WhatsApp test log + answer key |
| `29322b52-….pdf` | Hackathon handbook (Challenge B, scoring, timeline) |

The `docs/` folder, the 6-role source prompts (`prompts/agents/`) and the old
`examples/dataset-sample/` were removed when the workspace was cleaned up for the commit.
Full connection details and suggested WIB schedules now live in the repo root `README.md`.

## Evidence of use — still to capture

The handbook requires "**exported WorkBuddy workflow with evidence of use**". Add screenshots to
an `evidence/` folder:

1. Connector page showing all three servers Trusted
2. Expert Center showing the registered **Sobat UMKM** single agent
3. A real run: Step-0 pre-flight → one draft → self-check → `propose_plan`
4. Approval page `http://localhost:8103/plans/<id>` with `confirmation_summary`
5. `get_plan` showing `status: proposed` (nothing executed) and a later `rejected` plan
6. Audit log showing observed / decided / proposed / refused events
7. The two-Pak-Budi ambiguity question and the Friday-prayer schedule conflict
8. A refusal, clarification or approval-wait moment for the demo video

## Verified on 2026-11-06

- All 3 MCP services healthy; 39 tools (8 messaging + 11 memory + 20 office)
- `find_contacts "Pak Budi"` → 2 matches, `ambiguous: true`
- `check_schedule` Friday 12:00 WIB → `ok: false`, `friday_prayer` conflict + suggestions
- memory `start_run` → `finish_run` → `list_runs`
- `propose_plan` → `status: proposed`; `get_plan` → pending, nothing executed

## Before testing

Restart WorkBuddy (the MCP config was written after the app launched), Trust the three connectors,
and start a **new conversation** — a running session's tool list is fixed at session start.
Re-select **Sobat UMKM** after the team removal.

**Quick smoke test:** ask *"Cek hari ini"*. Step 0 calls `list_rules`, `list_channels` and
`list_runs`; if all three return, the wiring is live. If one fails, the agent stops and prints a
fixed recovery message with the backend download link (https://drive.google.com/file/d/175jmxjaFbbNLhZ9vduyMDlCNWCSV4Qf_/view?usp=sharing). It never guesses.

## Not included

- Avatar images (`avatars/` holds a placeholder only)
- The backend service binaries/source — this bundle is the WorkBuddy workflow only. Download
  `sobatpesan-1.0-bundle.zip` at https://drive.google.com/file/d/175jmxjaFbbNLhZ9vduyMDlCNWCSV4Qf_/view?usp=sharing
  (the agent's Step-0 error message points here when the MCP services are not active)
- Automations — not created; suggested WIB schedules are listed in the repo root `README.md`
