# SleepLab - Telegram bot
This is a demo of ready-to-market project - a Telegram assistant that helps solving sleep problems.

## SleepLabBot 

A production-grade Telegram assistant for diagnostics of sleep problems to provide personalized plan. Built using Cursor to manage full development cycle: from initial logic & workflow design, documentation, managing backlog & prioritization (MoSCoW), iteration - to deployment with DB logic, scheduling, notifications, payment integrations.


### Demo (Loom)
- Walkthrough video: https://www.loom.com/share/5d5d15164d5a45dcba01daa947cce515
  

## Executive Summary
SleepLabBot guides users through a structured, medically‑informed questionnaire (L0/L1/L2), delivers a personalized PDF diagnosis selected from pre‑uploaded PDFs, and orchestrates delayed delivery with reminders. It supports paid flows (promo codes, direct payments, Tilda → Robokassa), robust idempotency, and operational tooling for smoke tests and user state inspection on Heroku.


## Key Outcomes
- Reliable payment collection and reconciliation via webhooks
- Deterministic, idempotent delivery of diagnosis (instant and delayed flows)
- Resilient scheduling through APScheduler (Heroku dyno-friendly)
- Operator tooling to set up fixtures, smoke-test critical paths, and inspect user state


## System Architecture
```
┌────────────────────────────── Heroku (Prod) ───────────────────────────────┐
│                                                                            │
│  Telegram Bot (aiogram, asyncio)                                           │
│  • FSM for multi-step flows (branching by answers; gender‑specific)        │
│  • Sends questions & collects answers                                      │
│  • Triggers scheduling for diagnosis                                       │
│                                                                            │
│  Webhook Worker (payments, Tilda)                                          │
│  • Robokassa & Tilda webhook endpoints                                     │
│  • Idempotent payment confirmation                                         │
│  • Cleanup of payment warning messages                                     │
│                                                                            │
│  APScheduler (delayed tasks)                                               │
│  • Schedules delayed diagnosis PDFs                                        │
│  • Safe retries & job introspection                                        │
│                                                                            │
│  Database Layer                                                            │
│  • Heroku Postgres (prod)                                                  │
│  • SQLite (local dev)                                                      │
│                                                                            │
│  PDF Delivery                                                              │
│  • Pre‑uploaded diagnosis PDFs (rule‑based selection)                      │
│  • Telegram file delivery                                                  │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘

Local Dev: identical logic; SQLite instead of Postgres. CLI scripts use a “light” config to avoid initializing the full bot stack.
```


## Core Features
- Multi-block, multi-step flows questionnaire (L0/L1/L2) - branching by answers; gender‑specific 
- Saved checkpoints, and resumable sessions
- Payment flows: Robokassa (with promo codes), Tilda landing → Robokassa
- Post-payment UX polish: automatic deletion of “payment not completed” banners
- Delayed diagnosis delivery (with reminders) via APScheduler
- Admin/test tooling: fixtures, smoke tests, user status, scheduler status
- Idempotency: safe to re-run webhooks, avoid duplicates and race conditions


## User Flow (High Level)
1) Onboarding and L0/L1/L2 tests via Telegram messages and inline buttons
2) Payment gating (price banner, promo codes, redirect) and confirmation
3) Diagnosis creation: instant delivery upon completion or delayed schedule
4) Delayed flow: user sees a “2-hour” preface → scheduler sends PDFs later
5) Post-delivery state locking to avoid duplicates; explicit user flags updated


## Payments & Webhooks
- Providers: Robokassa (primary), Tilda (landing → Robokassa redirect)
- Webhooks: confirm payment, set internal flags, delete warning banners, unlock next level tests and further diagnosis delivery
- Idempotency: deduplicate by invoice/order identifiers; defensive updates in DB
- UX: after successful payment, user is automatically returned to the bot


## Delayed Diagnosis (Scheduler)
- APScheduler plans delivery after specific conditions (e.g., completion or post‑/start flows)
- Jobs are tracked and can be displayed from CLI (`--show_jobs` flag)
- Delivery marks `diagnosis_sent = True` only after successful file transmission
- Resilience: safe retries, no early “sent” flags, avoiding silent failures


## Data Model (Overview)
- `users`: profile, FSM checkpoints, flags for test progression and delivery
- `test_results`: per‑test answers and computed results, `diagnosis_sent`
- `payments`: orders, amounts, promo code metadata, confirmation flags
- Referential logic: `user_has_test_result`, `get_latest_test_result`, `user_has_diagnosis_sent`
- Safety: extra columns like `last_payment_warning_message_id` for UX cleanup


## Reliability & Idempotency Principles
- Webhook handlers are side‑effect safe if re‑invoked (network retries, duplicates)
- Deletions of transient UI messages (payment warnings) are guarded with state checks
- Scheduler jobs set final “sent” flags only after confirmed delivery
- DB connections hardened (timeouts/keepalives) against hanging in cloud environments


## Operations & Tooling
The project includes CLI utilities (used on Heroku and locally) to speed up QA and support.

- `scripts.fixtures`: set up user state (checkpoints, payments, promo types), optionally show scheduled jobs
- `scripts.user_status`: print user profile + scheduled jobs (`--show_jobs`)
- `scripts.smoke`: smoke tests for critical user journeys
- `scripts.scheduler_status`: list APScheduler jobs

Examples (Heroku dyno):
```bash
heroku run -a <app> --no-tty -- python -u -m scripts.fixtures --user <id> --scenario l2_completed --plan 5 --show_jobs
heroku run -a <app> --no-tty -- python -u -m scripts.user_status --user <id> --show_jobs
```


## Tech Stack
- Language: Python 3.x (asyncio)
- Framework: aiogram (Telegram Bot API)
- Scheduling: APScheduler
- Databases: PostgreSQL (Heroku), SQLite (local)
- Hosting: Heroku (dynos, logs, Dataclips)
- Payments: Robokassa (incl. promo codes), Tilda webhooks → Robokassa
- PDFs: selection of pre‑uploaded diagnosis PDFs and Telegram file delivery
- CI‑style scripts: `argparse`-based CLI tools for fixtures, smoke tests, scheduler


## Security & Privacy
- No plaintext secrets committed; environment-driven configuration
- Minimal PII stored; only data required for service operation and auditing
- Idempotent webhook design prevents accidental double charges or duplicate deliveries


## My Role (Owner / IC)
- Product: Journey design (L0/L1/L2) / UX flow, Figma screens, pricing flows, delayed delivery UX, texts & PDF's desing, testing, bug fixing
- Tech: Architecture, database schema, webhook design, scheduler reliability (AI-paired)
- Ops: Heroku deployment, troubleshooting, test fixtures, documentation (with AI-paired)
- QA: Manual/CLI-based smoke tests, state introspection, proactive monitoring (with AI-paired)


## Lessons Learned
- Treat webhooks and schedulers as first‑class ops surfaces (visibility + control)
- UX polish (e.g., deleting outdated banners) reduces confusion and support load


## Roadmap (Next)
- Subscription model: trial → monthly recurring; upsell after diagnosis
- Daily sleep diary + 3 habits: reminders, tracking, weekly digest
- Weekly digest: PNG chart (matplotlib) sent as Telegram photo
- AI additions: LLM guidance with RAG over curated sleep knowledge base; predictive signals for deterioration with proactive advice


## How to Use This README
This file is designed for a public portfolio repository. The source code remains private; the README + Loom demo illustrate the architecture, ops, and product craft behind a productionized Telegram bot.

— End —
