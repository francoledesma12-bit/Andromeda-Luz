# LUZ — AI Sales Orchestrator for Local Commerce

> **Production system** actively serving 3 WhatsApp lines + Telegram for Rosario Exclusivos (tech retailer, Argentina).
> Running continuously since April 2026 on DigitalOcean VPS.

---

## What is LUZ?

LUZ is a multi-channel AI orchestration system that automates customer service, real-time pricing, human routing, and commercial intelligence for a physical technology store. It handles hundreds of daily conversations across multiple WhatsApp lines, resolves product queries from a live Google Sheets catalog, connects to Odoo ERP for stock and CRM, and routes complex cases to human sellers — all in real time.

No demos. No prototypes. This system runs in production serving real customers every day.

---

## Architecture: The Orchestra

```
WhatsApp Line 1 (Libertad)  ──┐
WhatsApp Line 2 (Paraguay)  ──┼──► nuevo_rosario_run.py ──► LuzBrain
WhatsApp Line 3 (Ecommerce) ──┘         (runner)          (core_luz.py)
Telegram                    ────────────► bot.py

LuzBrain per-message flow:
  1. RuleManager     ──► deterministic rules (FAQ, hours, payments) — no LLM
  2. PromptRouter    ──► selects right prompt per line and context
  3. StateManager    ──► loads per-client conversation state
  4. MemoryManager   ──► loads persistent client memory
  5. LLM + Tool Call ──► OpenAI with catalog search function
       ├── GoogleSheetPrices  ──► live catalog (refreshed every 5 min)
       ├── AgenteOdoo         ──► ERP stock + CRM leads
       └── derivar.py         ──► human handoff detection
  6. CarritoManager  ──► shopping cart updates
  7. Save state + memory
```

---

## Core Modules

### Runner & Brain

| File | Description |
|------|-------------|
| `nuevo_rosario_run.py` | Production runner — manages 3 WhatsApp instances + polling loop + singleton process lock |
| `nuevo_cerebro/core_luz.py` | LuzBrain — main AI brain, orchestrates all modules per incoming message |
| `bot.py` | Telegram bot — personal assistant, monitoring, and operational control |

### nuevo_cerebro/ (New Brain Modules)

| Module | Description |
|--------|-------------|
| `rule_manager.py` | Deterministic business rules — answers FAQs without consuming LLM tokens |
| `prompt_router.py` | Routes to the right prompt based on WhatsApp line and conversation context |
| `state_manager.py` | Tracks conversation state per client (SQLite-backed) |
| `memory_manager.py` | Persistent memory per client across sessions |
| `carrito_manager.py` | Shopping cart management within a conversation |
| `optout_manager.py` | Customer opt-out and blocking management |
| `sheet_exclusions.py` | Filters out unavailable or excluded products from results |

### nuevo_cerebro/tools/ (Tool Agents)

| Tool | Description |
|------|-------------|
| `agente_buscador.py` | Product search — fuzzy matching + category filtering against live catalog |
| `agente_odoo.py` | Odoo ERP integration — stock queries, CRM leads, sale orders |
| `derivar.py` | Human handoff — detects when to route to a human seller |
| `google_sheet_prices.py` | Real-time price catalog from Google Sheets |
| `catalogo_categorias.py` | Category taxonomy and routing |

### nuevo_cerebro/prompts/ (Line-Specific Prompts)

| Prompt | Line |
|--------|------|
| `prompt_rosario.txt` | Main retail line — Rosario Exclusivos customers |
| `prompt_interno.txt` | Internal operational line |
| `prompt_iphone.txt` | Specialized Apple/iPhone line |
| `prompt_mayorista.txt` | Wholesale and distributor line |
| `prompt_paraguay_header.txt` | Paraguay cross-border commerce line |

### Specialized Agents

| Agent | Description |
|-------|-------------|
| `agente_vision.py` / `agente_vision_gemini.py` | Image analysis — identifies products from customer-sent photos |
| `agente_financiero.py` | Financial queries — exchange rates, payment methods, installments |
| `agente_control_final.py` | Response quality control — final validation before sending |
| `agente_filtrador.py` | Content filter — detects off-topic or inappropriate content |
| `agente_datos.py` | Data extraction and structuring from conversations |
| `agente_felino.py` | Biological wellness integration (pet care monitoring) |
| `agente_imagenes.py` | Image generation and media handling |
| `agente_creativo_gemini.py` | Creative content generation via Google Gemini |
| `agente_resumen_odoo.py` | ERP data summarization for reports |

### WhatsApp Integration

| File | Description |
|------|-------------|
| `whatsapp_handler.py` | Sends messages, media, and audio via Green-API |
| `whatsapp_collector.py` | Polls and processes incoming messages from Green-API |

### Andromeda Dashboard

| File | Description |
|------|-------------|
| `api_andromeda.py` | FastAPI backend — commercial intelligence endpoints |
| `dashboard1/src/` | React 19 + Vite frontend — real-time commercial dashboard |

### Support Modules

| Module | Description |
|--------|-------------|
| `campana_final.py` | Campaign broadcast to segmented customer lists |
| `gmail_reader.py` | Supplier email monitoring for stock and pricing updates |
| `bybit_handler.py` | Bybit crypto trading integration |
| `coati_db.py` | SQLite database layer for order and shipping tracking |
| `stats_manager.py` | Session and conversation statistics |
| `chart_generator.py` | Chart generation for dashboard visualizations |
| `filtro_rubro_engine.py` | Category-based product filtering engine |
| `andromeda_sync.py` | Data synchronization between modules |
| `luz_analitica.py` | Conversation analytics and reporting |
| `multiagente_logic.py` | Multi-agent coordination logic |
| `memory_manager.py` | Root-level memory management utilities |

---

## Integrations

| Service | Purpose | Module |
|---------|---------|--------|
| **Green-API** | WhatsApp messaging — 3 concurrent instances | `whatsapp_handler.py` |
| **OpenAI API** | LLM reasoning + function calling | `core_luz.py` |
| **Google Sheets** | Live product catalog and real-time pricing | `google_sheet_prices.py` |
| **Odoo ERP** | Stock management, CRM leads, sale orders | `agente_odoo.py` |
| **Gmail API** | Supplier email monitoring | `gmail_reader.py` |
| **Google Gemini** | Image analysis + creative content generation | `agente_vision_gemini.py` |
| **Groq** | Fast inference for vision and audio tasks | `agente_vision_groq.py` |
| **Telegram Bot API** | Personal monitoring and operational control | `bot.py` |
| **Bybit API** | Crypto trading automation | `bybit_handler.py` |

---

## Security

LUZ and Andromeda handle sensitive commercial data across multiple businesses: customer conversations, product pricing, billing information, ERP records, and loyalty points balances. Security practices implemented:

- All credentials loaded via `.env` — never hardcoded in source
- Google service account keys at restricted filesystem permissions (600)
- Per-client conversation isolation in SQLite with write locks
- Singleton process lock prevents duplicate bot instances and message duplication
- Customer opt-out system for privacy compliance
- QR tokens signed with HMAC and short expiration (Andromeda loyalty)
- Anti-concurrency locks on financial operations (points ledger)

---

## Deployment

DigitalOcean VPS (Ubuntu 22.04), three concurrent processes:

```bash
# 1. API backend
cd /root/luz_bot
nohup ./venv/bin/python api_andromeda.py > api.out 2>&1 &

# 2. Bot runner (WhatsApp + Telegram)
nohup ./venv/bin/python nuevo_rosario_run.py > bot.out 2>&1 &

# 3. React dashboard
cd /root/luz_bot/dashboard1
nohup npm run dev -- --host 0.0.0.0 > dash.out 2>&1 &
```

Environment variables required — see `.env.example`.

---

## Tech Stack

- **Python 3.12** — async runtime, FastAPI, SQLite, OpenAI SDK, Google APIs
- **React 19 + Vite** — dashboard frontend
- **OpenAI** — GPT-4o with function calling
- **Google Gemini** — multimodal vision and creative tasks
- **DigitalOcean VPS** — production deployment

---

## About

Built and maintained by **Franco Ledesma** for **Rosario Exclusivos** — a technology retail store in Rosario, Argentina.

This system handles real customer conversations daily, automating ~80% of initial responses and routing complex cases to human sellers.

Part of the **Andromeda** commercial intelligence ecosystem — a SaaS loyalty and analytics platform for local commerce in Latin America.
