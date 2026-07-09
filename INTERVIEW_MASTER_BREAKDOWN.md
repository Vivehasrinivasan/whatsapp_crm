# WhatsApp CRM & Automation Platform — Master Technical Breakdown

> **Purpose**: A rigorous, interview-ready technical reference covering every critical file, function, design pattern, and architectural decision in this repository.  
> **Audience**: You, the builder — preparing to defend every line of this codebase in a Solutions Architect / Senior Backend Engineer interview.

---

## Table of Contents

1. [The Goal & Core Business Problem](#1-the-goal--core-business-problem)
2. [The Ultimate Outcome & Metrics Gained](#2-the-ultimate-outcome--metrics-gained)
3. [Architectural Blueprint & Core Features](#3-architectural-blueprint--core-features)
4. [Comprehensive File-by-File & Function-by-Function Teardown](#4-comprehensive-file-by-file--function-by-function-teardown)
   - [4.1 `provider_adapter.py`](#41-provider_adapterpy)
   - [4.2 `whatsapp_sender.py`](#42-whatsapp_senderpy)
   - [4.3 `level2_profiler.py`](#43-level2_profilerpy)
   - [4.4 `insights_service.py`](#44-insights_servicepy)
   - [4.5 `server.py` & `scheduler_service.py`](#45-serverpy--scheduler_servicepy)
   - [4.6 Supporting Critical Files](#46-supporting-critical-files)
5. [The Perfect Interview Storytelling Framework (STAR Method)](#5-the-perfect-interview-storytelling-framework-star-method)
6. [Potential Interview Questions & Model Answers](#6-potential-interview-questions--model-answers)

---

## 1. The Goal & Core Business Problem

### 1.1 The Explicit Problem

Traditional Indian wholesale/retail businesses (kiranas, FMCG distributors, general stores) communicate with customers via WhatsApp — but they do it **manually**. The pain points are:

| Problem | Description |
|---|---|
| **Manual Customer Targeting** | Shop owners scroll through contact lists and send messages one-by-one. A 200-customer campaign takes an entire working day, and there is zero targeting intelligence — every customer gets the same generic "Sale today!" message. |
| **Spam & API Ban Risk** | WhatsApp aggressively bans accounts that send bulk unsolicited messages. Without controlled throttling, working-hours enforcement, and daily send caps, a business owner risks **permanent account termination** — losing their primary customer communication channel overnight. |
| **Unstructured Data Sources** | Purchase history lives in messy CSV exports from billing software (Tally, Busy, custom POS systems). Column names are inconsistent (`Cust_Name` vs `customer_name` vs `Full Name`), date formats vary wildly, and phone numbers lack standardization. Standard CRMs cannot ingest this data without expensive data cleaning consultants. |
| **No Behavioral Intelligence** | Generic CRMs classify customers by static tags. They have no concept of purchase recency, frequency, monetary value, or basket-size behavior. This means a VIP customer who shops daily and a one-time buyer who visited 6 months ago receive identical marketing — destroying conversion rates and wasting limited WhatsApp message quotas. |
| **Race Conditions in Dual Sending** | If two campaign batches overlap, or if a server restarts mid-campaign, messages can be sent twice to the same customer, or messages get stuck in a "processing" state forever — a **deadlock** that silently stalls the entire campaign pipeline. |

### 1.2 The Ultimate Business & Technical Goal

Build a **self-contained, high-concurrency WhatsApp CRM platform** that:

1. **Ingests raw, messy data** (CSV/XLSX/PDF) with intelligent column auto-detection and user-correctable column mapping.
2. **Automatically segments customers** using a proprietary **Hybrid RFM+B (Recency, Frequency, Monetary, Bulkiness)** scoring model with a 5-tier waterfall classification.
3. **Profiles individual customer behavior** using a Level 2 behavioral engine that computes category affinity, premium/bulk product preferences, co-purchase patterns, and recency-weighted engagement.
4. **Matches customers to the best promotional offer** via a Composite Affinity Rank engine (`Affinity = S × [1 + P]`).
5. **Sends personalized WhatsApp messages at scale** through a deterministic 6-state message engine with anti-spam throttling, working-hours enforcement, automatic retry with exponential backoff, and dead-letter queue (DLQ) management.
6. **Provides real-time campaign monitoring** with per-batch, per-segment, and per-message drill-down visibility.

---

## 2. The Ultimate Outcome & Metrics Gained

### 2.1 Concrete Technical Outcomes

| Outcome | How It's Achieved |
|---|---|
| **Resource Isolation** | The Scheduler Worker runs as an independent `asyncio` background loop inside the FastAPI ASGI server — it shares the event loop but is **logically decoupled** from the HTTP request/response cycle. A slow campaign never blocks API responses. |
| **Safety Guarantees Against API Bans** | Three-layer protection: (1) IST working-hours gate (9 AM–7 PM), (2) daily per-account send cap (default 200/day, configurable via `WHATSAPP_MAX_PER_DAY`), (3) inter-message jitter (0.5–1.5s randomized delay between sends) to mimic human behavior. |
| **Zero Duplicate Sends** | Atomic MongoDB `findAndModify` (via `update_one` with status precondition) acts as a distributed lock — only one worker instance can transition a message from `pending` → `processing`. |
| **Deadlock Prevention** | Three independent safeguards: (1) `asyncio.wait_for(timeout=45s)` hard-kills stalled processing, (2) `try...finally` releases any item still stuck in `processing` after handler completion, (3) orphan recovery scans for messages stuck in `processing` >60s and auto-resets them. |
| **Deterministic Testing** | The `DummyGateProvider` uses MD5 hashing to produce stable, reproducible delivery outcomes. Given the same phone numbers, **every demo run produces identical success/failure distributions** — enabling reliable QA. |
| **Intelligent Automation** | Per-customer template hydration fills 8 behavioral placeholders (`{{favorite_category}}`, `{{favorite_premium_product}}`, etc.) — transforming generic blasts into personally relevant messages without manual effort. |

### 2.2 Intelligence Automation Metrics

| Metric | Value |
|---|---|
| Customer Segments | 5 tiers: VIP, At-Risk, Potential Bulk, Loyal Frequent, Boring |
| Behavioral Variables per Customer | 8 template variables + 6 auxiliary fields |
| Offer Matching | O(C × O) composite affinity ranking (C=customers, O=active offers) |
| Retry Resilience | 3-attempt exponential backoff (15s → 30s → DLQ) |
| Throughput | 8 messages/poll cycle × 1 poll/7s ≈ ~69 messages/minute sustained |

---

## 3. Architectural Blueprint & Core Features

### 3.1 High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              FRONTEND (React + ShadCN UI)                   │
│  ┌──────────┐ ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐   │
│  │Dashboard │ │Campaign Creator  │ │Batch Monitor │ │Offers / Templates│   │
│  │  Page    │ │(Segment Templates│ │(Real-time    │ │   Management     │   │
│  │          │ │ AI Hydration)    │ │ Drill-down)  │ │                  │   │
│  └────┬─────┘ └───────┬──────────┘ └──────┬───────┘ └───────┬──────────┘   │
│       │               │                   │                 │              │
│       └───────────────┴───────────────────┴─────────────────┘              │
│                               │ REST API (HTTP)                            │
└───────────────────────────────┼────────────────────────────────────────────┘
                                │
┌───────────────────────────────┼────────────────────────────────────────────┐
│                      BACKEND (FastAPI ASGI Server)                         │
│                               │                                            │
│  ┌────────────────────────────┼────────────────────────────────────────┐   │
│  │              API LAYER (Routes)                                     │   │
│  │  auth.py │ customers.py │ batches.py │ shops.py │ monitoring.py    │   │
│  │  templates.py │ offers.py │ files.py │ dashboard.py                │   │
│  └────────────────────────────┼────────────────────────────────────────┘   │
│                               │                                            │
│  ┌────────────────────────────┼────────────────────────────────────────┐   │
│  │            SERVICE LAYER (Business Logic)                           │   │
│  │                            │                                        │   │
│  │  ┌─────────────────────────┼──────────────────────────────────┐     │   │
│  │  │ INSIGHT PIPELINE        │                                  │     │   │
│  │  │  file_service.py ──► customer_service.py ──► transaction   │     │   │
│  │  │  (CSV/XLSX/PDF        (Column Mapping,       _service.py  │     │   │
│  │  │   Ingestion,           Identity Upsert)      (Period-     │     │   │
│  │  │   SHA-256 Dedup)                              Scoped TX)  │     │   │
│  │  │       │                                          │         │     │   │
│  │  │       └──────────────────────────────────────────┤         │     │   │
│  │  │                                                  ▼         │     │   │
│  │  │                               insights_service.py          │     │   │
│  │  │                              (RFM Scoring + L2 Profiling)  │     │   │
│  │  │                                       │                    │     │   │
│  │  │                                       ▼                    │     │   │
│  │  │                              level2_profiler.py             │     │   │
│  │  │                             (Pandas Behavioral Engine)     │     │   │
│  │  └────────────────────────────────────────────────────────────┘     │   │
│  │                                                                     │   │
│  │  ┌────────────────────────────────────────────────────────────┐     │   │
│  │  │ CAMPAIGN ENGINE                                            │     │   │
│  │  │  batch_service.py ──► scheduler_service.py (Background)    │     │   │
│  │  │  (Campaign Creation,   (7s Poll, 6-State Machine,          │     │   │
│  │  │   Segment Routing,      Micro-Batch Processing,            │     │   │
│  │  │   Template Hydration)   Retry Backoff, Orphan Recovery)    │     │   │
│  │  │         │                        │                         │     │   │
│  │  │         │                        ▼                         │     │   │
│  │  │         │              provider_adapter.py                 │     │   │
│  │  │         │             (Adapter Pattern Router)             │     │   │
│  │  │         │                    │         │                   │     │   │
│  │  │         │                    ▼         ▼                   │     │   │
│  │  │         │          DummyGate    WhatsAppWeb                │     │   │
│  │  │         │          Provider     Provider                   │     │   │
│  │  │         │          (MD5 Sim)    (Playwright)               │     │   │
│  │  │         │                        │                         │     │   │
│  │  │         │                        ▼                         │     │   │
│  │  │         │              whatsapp_sender.py                  │     │   │
│  │  │         │             (Browser Automation)                 │     │   │
│  │  └─────────┼──────────────────────────────────────────────────┘     │   │
│  │            │                                                        │   │
│  │  ┌────────┼───────────────────────────────────────────────────┐     │   │
│  │  │ OFFER ENGINE                                               │     │   │
│  │  │  offers_service.py (Composite Affinity Rank: S × [1 + P]) │     │   │
│  │  └────────────────────────────────────────────────────────────┘     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                               │                                            │
│  ┌────────────────────────────┼────────────────────────────────────────┐   │
│  │          DATA LAYER (MongoDB via Motor Async Driver)                │   │
│  │  Collections: users, shops, files, customers, products,             │   │
│  │  transactions, customer_insights, templates, campaigns,             │   │
│  │  batches, messages, offers                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Main Product Features

| Feature | Description | Key File(s) |
|---|---|---|
| **Campaign Dashboard** | Overview of all shops, customer segments, upload cycles, and campaign stats | `DashboardPage.js`, `ShopDashboardPage.js`, `dashboard_service.py` |
| **Intelligent File Ingestion** | Upload CSV/XLSX/PDF → auto-detect columns → SHA-256 content-hash deduplication → B2 cloud storage | `file_service.py`, `classifier.py` |
| **Hybrid RFM+B Customer Segmentation** | 5-tier waterfall: VIP → At-Risk → Potential Bulk → Loyal Frequent → Boring | `classifier.py`, `insights_service.py` |
| **Level 2 Behavioral Profiler** | 8 per-customer template variables via pure Pandas analytics | `level2_profiler.py` |
| **Custom Offer Matching Engine** | Composite Affinity Rank (`S × [1+P]`) maps each customer → best offer | `offers_service.py` |
| **Segment-Based Template Routing** | Different message templates per customer segment in a single campaign | `batch_service.py`, `TemplatesPage.js` |
| **6-State Message Engine** | `pending → processing → sent/retry_wait/failed_permanently/cancelled` | `scheduler_service.py` |
| **Swappable Messaging Providers** | Adapter Pattern: `mock` ↔ `twilio` ↔ `whatsapp_web` via `.env` toggle | `provider_adapter.py` |
| **Real-Time Batch Monitoring** | Per-campaign, per-batch, per-message drill-down with error categorization | `MonitoringPage.js`, `BatchMonitorPage.js`, `monitoring_service.py` |
| **Anti-Spam Safety Net** | Working-hours gate, daily caps, inter-message jitter, consecutive-failure circuit breaker | `whatsapp_sender.py`, `scheduler_service.py` |

---

## 4. Comprehensive File-by-File & Function-by-Function Teardown

---

### 4.1 `provider_adapter.py`
**Path**: `backend/services/provider_adapter.py` (225 lines)

**Role**: The **single exit point** for all outbound messaging. The scheduler worker never calls WhatsApp APIs or mock functions directly — it always calls `ProviderAdapter.send_message()`, which routes to the active provider.

**Design Pattern**: **Adapter Pattern** (GoF) + **Strategy Pattern** for runtime provider swapping.

**Technologies**: `hashlib` (MD5), `asyncio`, `uuid`, `os.environ`

#### Class: `BaseProvider` (L32-36)
- Abstract base class with a single async method `send(phone, content, attempt_count) → Dict`
- All concrete providers inherit from this — **polymorphism** guarantees the scheduler can call any provider identically.

#### Class: `DummyGateProvider` (L41-126) — Deterministic MD5 Bucket Gate

This is the most interview-worthy provider. It simulates real-world delivery outcomes **deterministically**.

**Core Concept — MD5 Bucket Math** (L62-68):
```python
normalised = phone.strip().replace(" ", "").replace("(", "").replace(")", "").replace("-", "")
hash_hex = hashlib.md5(normalised.encode("utf-8")).hexdigest()
tail_int = int(hash_hex[-6:], 16)  # last 6 hex chars → integer 0..16,777,215
bucket = tail_int % 100            # modulo 100 → bucket 0..99
```

**Why MD5?**
- MD5 is a **deterministic** hash — same input always produces the same output.
- `hash_hex[-6:]` extracts 24 bits of entropy — enough for uniform distribution across 100 buckets.
- **Reproducibility**: Given the same CSV of phone numbers, every demo run produces **identical** success/failure distributions. This is critical for QA — you can predict exactly which customers will fail, retry, and succeed.

**Track Mapping** (for a 200-customer dataset):

| Bucket Range | Track Name | Behavior | Expected Count |
|---|---|---|---|
| 0–1 | Terminal DLQ | `invalid_number` → `failed_permanently` immediately | ~4 |
| 2 | Exhausted Network | `network_error` × 3 retries → DLQ | ~1 |
| 3–12 | Automated Recovery | `rate_limit` on attempt 1, success on attempt 2+ | ~20 |
| 13–99 | Clean | Instant `success` | ~175 |

**Programming Concepts Applied**:
- **Deterministic Simulation**: Reproducible test outcomes without real API dependencies
- **Phone Normalization**: Strips whitespace, parentheses, dashes before hashing — prevents hash collisions from formatting differences
- **Bucket Gating**: Uses modulo arithmetic for uniform distribution across lifecycle tracks
- **Simulated Latency**: `asyncio.sleep(random.uniform(0.05, 0.15))` — realistic 50–150ms network delay

#### Class: `WhatsAppWebProvider` (L150-167)
- Delegates to the `WhatsAppWebSender` Playwright singleton
- Maps the sender's result dict into the **Uniform Transaction Schema**
- Includes `reschedule_at` field for working-hours rescheduling

#### Class: `ProviderAdapter` (L182-224) — Static Adapter Router

**`_get_provider()` (L190-204)**:
- Reads `PROVIDER_MODE` from environment (defaults to `"mock"`)
- Lazy singleton initialization — first call instantiates the provider, subsequent calls reuse it
- Falls back to `DummyGateProvider` on unknown mode with a logged warning

**`send_message()` (L206-224)**:
- Static method — no instance required
- Wraps the provider's `send()` in a try/except that catches **any** exception and returns a `"temporary"` outcome — this means unhandled provider crashes never kill the scheduler.

**Uniform Transaction Schema** (returned by all providers):
```python
{
    "success": bool,
    "provider_sid": str | None,    # provider-specific message ID
    "error": str | None,           # "network", "rate_limit", "invalid_number", or None
    "outcome": str                 # "success" | "temporary" | "permanent"
}
```

---

### 4.2 `whatsapp_sender.py`
**Path**: `backend/services/whatsapp_sender.py` (198 lines)

**Role**: Real WhatsApp Web automation using **Playwright** (async Chromium browser control). This is the **production provider** — it actually sends WhatsApp messages by automating the WhatsApp Web interface.

**Technologies**: `playwright.async_api`, `asyncio`, `urllib.parse`, `datetime` (timezone-aware), `re`

#### IST Timezone Handling (L26-35)

```python
IST_OFFSET = timedelta(hours=5, minutes=30)

def _now_ist() -> datetime:
    return datetime.now(timezone.utc) + IST_OFFSET
```

**Why manual offset instead of `pytz`?**
- Zero external dependency for a fixed-offset timezone
- IST is UTC+5:30 and doesn't observe DST — a simple `timedelta` is correct and sufficient
- `_next_day_9am_ist_utc()` computes the next 9 AM IST as a UTC datetime — used for rescheduling messages that arrive outside working hours

#### Class: `WhatsAppWebSender` (L38-197)

**Safety Constants**:
```python
WORKING_HOURS_START = 9   # 9 AM IST
WORKING_HOURS_END   = 19  # 7 PM IST
MAX_MESSAGES_PER_DAY = 200
```

**`__init__()` (L43-53)** — Instance State:
| Field | Type | Purpose |
|---|---|---|
| `_max_per_day` | `int` | Daily send cap (env-configurable) |
| `_daily_count` | `int` | Messages sent today (in-memory counter) |
| `_daily_count_date` | `str` | Date string for counter reset detection |
| `_consecutive_failures` | `int` | Circuit breaker counter |
| `playwright` | `Playwright` | Playwright engine instance |
| `browser_context` | `BrowserContext` | Persistent browser session |
| `page` | `Page` | Active browser tab |
| `_is_running` | `bool` | Lifecycle guard |

**`start()` (L55-85)** — Playwright Session Initialization:
- Uses `launch_persistent_context()` with a local `whatsapp_profile/` directory
- **Persistent context saves cookies and localStorage** — the user scans the WhatsApp QR code **once**, and subsequent server restarts reuse the authenticated session
- Navigates to `web.whatsapp.com` with `wait_until="networkidle"` (waits for all network requests to settle)
- `headless` mode is configurable via `WHATSAPP_HEADLESS` env var

**`_reset_daily_counter_if_needed()` (L96-100)**:
- Compares current IST date string against stored date
- If the date has changed (midnight IST rollover), resets `_daily_count` to 0
- **In-memory only** — survives within a process lifecycle but resets on server restart (acceptable for single-instance deployments)

**`send_message()` (L113-188)** — Core Send Flow:

```
Guard: Working hours check (currently commented out for testing)
    ↓
Guard: Daily cap check (_daily_count >= _max_per_day → rate_limit)
    ↓
Guard: Circuit breaker (_consecutive_failures >= 5 → rate_limit)
    ↓
Guard: Playwright running check
    ↓
Step 1: Navigate to wa.me deep link with URL-encoded message
    ↓
Step 2: Wait for "Send" button selector (timeout 20s)
    ↓
    ├── If timeout → check for "Phone number is invalid" dialog → return invalid_number
    ↓
Step 3: Click send button
    ↓
Step 4: Wait 2s for delivery confirmation
    ↓
Reset _consecutive_failures, increment _daily_count
Return success with timestamp-based SID
```

**Circuit Breaker Pattern** (L128-133):
- If 5 consecutive sends fail, the sender assumes a systemic issue (WhatsApp rate limit, network outage) and returns `rate_limit` — which triggers bulk rescheduling of the entire campaign to the next day's 9 AM IST.
- Resets to 0 after triggering — this is a **simple circuit breaker** (not half-open/closed states).

**Module-Level Singleton** (L190-197):
```python
_sender_instance: WhatsAppWebSender | None = None

def get_whatsapp_sender() -> WhatsAppWebSender:
    global _sender_instance
    if _sender_instance is None:
        _sender_instance = WhatsAppWebSender()
    return _sender_instance
```
Ensures only one Playwright browser instance exists process-wide — prevents resource leaks from multiple Chromium instances.

---

### 4.3 `level2_profiler.py`
**Path**: `backend/utils/level2_profiler.py` (563 lines)

**Role**: The **behavioral intelligence engine**. Computes 8 per-customer template variables from raw transaction and product data using **pure Pandas** — no database access. This is the most analytically dense file in the codebase.

**Technologies**: `pandas`, `numpy`, `re`, `hashlib` (implicit via product matching), `typing`

#### Section 1: Product Strategy — Premium & Bulk Detection (L28-119)

**`tag_premium_products()` (L42-96)**:

Premium detection uses a **per-category statistical threshold**:

```python
threshold = mean_price + 1.0 × std_price
is_premium = (price > threshold) OR (product_type == "premium")
```

**Fallback Logic** (Critical for small/uniform datasets):
```python
if std == 0 or count <= 1:
    threshold = mean * 1.15  # 15% above mean when std is zero
```

**Why this matters**: If a category has only 1 product (e.g., "Electronics" with just "TV"), `std = 0`, and `mean + 1.0*std = mean` — meaning the TV would never be "premium". The 15% uplift fallback ensures that in single-product categories, the product can still be flagged as premium if it's significantly priced.

**Luxury Detection** (Global):
```python
luxury_cutoff = df["price"].quantile(0.95)
is_luxury = price >= luxury_cutoff  # Top 5% globally
```

**`tag_bulk_products()` (L99-119)** — Hybrid Keyword + Numeric Rule:
```python
is_bulk = keyword_match(name) OR unit_keyword_match(unit) OR quantity_per_unit > 5 OR product_type == "bulk"
```

Keywords detected via compiled regex: `kg`, `ltr`, `pack`, `bundle`, `combo`, `family`, `jar`, `dozen`, `sack`, `carton`, etc.

#### Section 2: Behavioral Mapping — Category Affinity (L123-169)

**`compute_category_affinity()` (L141-169)**:

The core affinity formula per category C:

```
Affinity(C) = 0.5 × spend_ratio(C) + 0.3 × freq_ratio(C) + 0.2 × recency_weight(C)
```

Where:
- `spend_ratio(C)` = (total spend in C) / (total spend across all categories)
- `freq_ratio(C)` = (transaction count in C) / (total transaction count)
- `recency_weight(C)` = sum of per-transaction recency weights

**Recency Weight Tiers** (L127-131):
| Days Since Purchase | Weight |
|---|---|
| ≤ 30 days | 1.0 |
| 31–90 days | 0.6 |
| > 90 days | 0.2 |

**Why weighted affinity?** A customer who spent ₹10,000 in "Groceries" last week is more valuable in that category than one who spent ₹15,000 six months ago. The 50/30/20 weighting captures spend magnitude, shopping frequency, and temporal freshness.

#### Section 3: Per-Customer Product Variables (L172-297)

Six specialized functions compute specific product picks:

| Function | Returns | Strategy |
|---|---|---|
| `_best_premium()` | Highest-spend premium product in favorite category | Filter `is_premium=True` → group by `product_id` → `sum(amount)` → `idxmax()` |
| `_best_bulk()` | Highest-quantity bulk product across all purchases | Filter `is_bulk=True` → group by `product_id` → `sum(quantity)` → `idxmax()` |
| `_recently_bought()` | Product from most recent transaction | Sort by `purchase_date` descending → `iloc[0]` |
| `_complementary_product()` | Most co-purchased product with anchor | Find all dates when anchor was bought → find most frequent other product on those dates |
| `_fallback_premium_for_category()` | Global best-selling premium in category (fallback) | Store-wide search when customer has no premium purchases |
| `_fallback_bulk_global()` | Global highest-quantity bulk product (fallback) | Store-wide search when customer has no bulk purchases |

**Co-Purchase Analysis (L228-261)** — The `_complementary_product()` function:
```
Step 1: Find all dates where anchor_product was bought (across ALL customers)
Step 2: Find all transactions on those dates (excluding anchor)
Step 3: Return the most frequently purchased product → "complementary"
```
This is a **market basket analysis** technique (association rule mining) implemented as a simple frequency count.

#### Section 4: Main Entry Point — `build_customer_profiles()` (L303-550)

**Input**: Transactions DataFrame + Products DataFrame + shop_id
**Output**: List of dicts (one per customer) — ready for MongoDB insertion

**Fallback Cascade** (the most defensive part of this codebase):

For `favorite_premium_product`, the fallback chain is:
```
Customer's premium purchases in favorite category
    → Customer's premium purchases in any category
    → Global best-selling premium in favorite category (all customers)
    → Global best-selling premium across entire store
    → Overall best-selling product in favorite category (even if not premium)
    → Overall best-selling product across entire store
```

**Why 6 levels of fallback?** In a real-world kirana store, a customer might have only 3 transactions for "rice" and "oil" — none of which are "premium." Without aggressive fallbacks, the `{{favorite_premium_product}}` placeholder would render as empty in their WhatsApp message — which looks broken and unprofessional. The cascade ensures **every customer gets a meaningful recommendation**, even if it's a store-wide popular pick.

---

### 4.4 `insights_service.py`
**Path**: `backend/services/insights_service.py` (415 lines)

**Role**: The **centralized state warehouse**. This is the master pipeline that orchestrates Level 1 (RFM scoring) and Level 2 (behavioral profiling) and merges their outputs into a single `customer_insights` document per customer. The `customer_insights` collection is the **single source of truth** — it can be deleted and fully regenerated from raw transactions.

**Technologies**: `pandas`, `numpy`, `motor.motor_asyncio` (MongoDB async), `pymongo.UpdateOne` (bulk writes)

#### `recalculate_all_insights()` (L34-235) — Master Pipeline

**Step-by-Step Execution Flow**:

```
Step 1: Load all transactions for shop → RAM (Pandas DataFrame)
Step 2: Load all products for shop → RAM
Step 3: Compute foundational metrics per customer_id:
        - recency_days = (max_purchase_date - customer_last_purchase).days
        - frequency = unique purchase dates count
        - monetary = sum(amount)
        - bulkiness = total_quantity / purchase_count
Step 4: Level 1 — RFM Quintile Scoring (_compute_rfm_scores)
Step 5: Level 2 — Behavioral Profiling (build_customer_profiles)
Step 6: Merge both → upsert into customer_insights via bulk_write
```

#### `_compute_rfm_scores()` (L242-270) — Log-Damped RFM Calculations

**Monetary Damping** (L255-257):
```python
df["monetary_log"] = np.log1p(df["monetary"])
df["m_rank"] = df["monetary_log"].rank(method="average")
df["m_score"] = _safe_qcut(df["m_rank"], labels_asc=[1, 2, 3, 4, 5])
```

**Why `log1p` instead of raw monetary?** Without log damping, a single "whale" customer (e.g., ₹500,000 spend) distorts the quintile boundaries — pushing 80% of customers into the bottom quintile. `log(1+x)` compresses the distribution, giving mid-tier spenders a fair score.

**Recency Scoring** (L247-248):
```python
df["r_score"] = _safe_qcut(df["recency_rank"], labels_asc=[5, 4, 3, 2, 1])
```
Labels are **reversed**: low recency rank (most recent) → score 5 (best). This is because lower days = more recent = more valuable.

#### `_safe_qcut()` (L273-281) — Safe Quintile Fallbacks

```python
def _safe_qcut(series, labels_asc, q=5):
    try:
        return pd.qcut(series, q=q, labels=labels_asc, duplicates="drop")
    except (ValueError, TypeError):
        try:
            return pd.cut(series, bins=q, labels=labels_asc, include_lowest=True)
        except Exception:
            return pd.Series(3, index=series.index)  # Flat fallback: everyone gets 3
```

**Three-tier fallback**:
1. `pd.qcut` — Equal-frequency bins (ideal). Fails when too many duplicate values.
2. `pd.cut` — Equal-width bins. Fails on degenerate distributions.
3. **Constant 3** — Everyone gets the median score. This is the "nuclear option" that prevents any data shape from crashing the pipeline.

**Why this matters in interviews**: `pd.qcut` is the most common pandas pitfall in real-world data science. If 150 out of 200 customers have `recency = 0` days (all purchased today), `qcut` cannot create 5 equal-frequency bins and throws `ValueError`. This fallback chain handles every edge case.

#### `_waterfall_segment()` (L284-323) — 5-Tier Decision Tree

```
Rule 1: VIP         — total >= 12 AND (m >= 4 OR f == 5)
Rule 2: At-Risk     — r == 1 AND total > 4
Rule 3: Potential Bulk — 5 <= total <= 11 AND bulkiness > store_avg
Rule 4: Loyal Frequent — total >= 5 AND f >= 3 AND f >= m
Rule 5: Boring      — catch-all
```

**Evaluated top-to-bottom; first match wins.** This is a **waterfall** (not a classification matrix) — order of rules matters. VIP is checked first so that high-value customers are never accidentally classified as At-Risk.

#### MongoDB Bulk Write Strategy (L153-223)

```python
from pymongo import UpdateOne
ops = []
for _, row in agg_df.iterrows():
    ops.append(
        UpdateOne(
            {"shop_id": shop_id, "customer_id": cust_id},
            {"$set": doc},
            upsert=True
        )
    )
if ops:
    await db.customer_insights.bulk_write(ops, ordered=False)
```

**Why `bulk_write(ordered=False)` instead of individual `update_one` calls?**
- **Performance**: 200 individual `update_one` calls = 200 round-trips to MongoDB. `bulk_write` batches them into a **single network round-trip**.
- **`ordered=False`**: Allows MongoDB to execute operations in parallel. If one upsert fails (e.g., duplicate key), others still proceed — no cascading failures.
- **Upsert**: If the customer exists → update. If new → insert. Handles both fresh calculation and recalculation.

**Dormant Customer Marking** (L219-223):
```python
await db.customer_insights.update_many(
    {"shop_id": shop_id, "customer_id": {"$nin": active_ids}},
    {"$set": {"segment": CustomerCategory.DORMANT.value}}
)
```
Customers who no longer appear in transaction data are marked as `dormant` — they're not deleted (preserving history) but excluded from active campaign targeting.

---

### 4.5 `server.py` & `scheduler_service.py`
**Path**: `backend/server.py` (140 lines) + `backend/services/scheduler_service.py` (703 lines)

#### `server.py` — FastAPI ASGI Setup

**Technologies**: `FastAPI`, `CORSMiddleware`, `dotenv`, `uvicorn`, `asyncio`

**`startup_event()` (L55-101)** — Boot Sequence:
```
1. Verify MongoDB connection (ping)
2. Initialize database indexes (all 12 collections)
3. Run Phase 7 migrations (schema evolution)
4. Run one-time behavior map migration (legacy → customer_insights)
5. Initialize SchedulerWorker(db) and start background polling
6. If PROVIDER_MODE=whatsapp_web → start Playwright context (non-blocking)
```

**Key Design Decision**: The `SchedulerWorker` is instantiated as a **module-level global** (`message_scheduler = None`) and started inside the ASGI `startup` event. This means:
- It shares the FastAPI event loop (no separate process/thread needed)
- It's automatically cleaned up on `shutdown` event
- The `asyncio.create_task(sender.start())` for Playwright is **non-blocking** — the server starts accepting HTTP requests immediately while the browser boots in the background

**`shutdown_event()` (L104-126)** — Graceful Teardown:
```
1. Stop scheduler worker (waits for current cycle to finish)
2. Stop WhatsApp Web sender (close Playwright browser)
3. Close MongoDB connection
```

#### `scheduler_service.py` — Deterministic 6-State Engine

**Technologies**: `apscheduler.AsyncIOScheduler`, `asyncio`, `datetime` (timezone-aware), `motor`

**6-State Lifecycle**:
```
pending ──→ processing ──→ sent               (success)
                       ├──→ retry_wait         (transient, attempt < 3)
                       ├──→ failed_permanently  (terminal OR exhausted)
                       └──→ cancelled           (user-aborted)
```

**Configurable Boundary Matrix** (L58-72):
```python
POLL_INTERVAL_SECONDS = 7        # Heartbeat frequency
MICRO_BATCH_SIZE = 8             # Messages per poll cycle
INTER_MSG_JITTER_MIN = 0.5      # Min delay between messages (seconds)
INTER_MSG_JITTER_MAX = 1.5      # Max delay between messages
MAX_RETRY_COUNT = 3              # Attempts before DLQ
RETRY_BACKOFF_BY_ATTEMPT = {1: 15, 2: 30}  # Backoff schedule (seconds)
ORPHAN_THRESHOLD_SECONDS = 60   # Stale processing detection
```

**`_poll_cycle()` (L122-211)** — Core Heartbeat:

```python
# ── Concurrency Guard ──
if self._processing:
    return  # Previous cycle still running — skip
self._processing = True

try:
    # ── Working Hours Gate ──
    # (currently disabled for testing — would return early outside 9AM-7PM IST)
    
    # ── Orphan Recovery ──
    await self._recover_orphans()
    
    # ── Time-Window Fetch (the fetch-and-lock query) ──
    query = {
        "status": {"$in": ["pending", "retry_wait"]},
        "next_attempt_at": {"$lte": current_time},
    }
    items = db.messages.find(query)
        .sort([("priority", 1), ("next_attempt_at", 1)])
        .limit(MICRO_BATCH_SIZE)
    
    # ── Process each item ──
    for item in items:
        # Check campaign status (paused/cancelled)
        await asyncio.wait_for(self._process_item(item), timeout=45.0)
        await asyncio.sleep(random.uniform(0.5, 1.5))  # Jitter
    
    # ── Auto-Complete Check ──
    if no items: await self._auto_complete_campaigns()

finally:
    self._processing = False
```

**Why `self._processing` guard?** APScheduler fires `_poll_cycle` every 7 seconds regardless of whether the previous cycle finished. Without the guard, two cycles could overlap — processing the same messages simultaneously, causing duplicate sends.

**`_process_item()` (L265-343)** — Atomic Concurrency Lock:

```python
# Step 1: Atomic lock via MongoDB conditional update
lock_result = await self.db.messages.update_one(
    {"id": item_id, "status": {"$in": ["pending", "retry_wait"]}},
    {"$set": {"status": "processing", ...}},
)
if lock_result.modified_count == 0:
    return  # Another worker already grabbed it
```

**This is a distributed lock without Redis.** The MongoDB `update_one` with a status precondition is atomic — only one caller can transition a message from `pending` → `processing`. If two workers try simultaneously, only one gets `modified_count == 1`; the other gets `0` and silently skips.

**`_handle_transient_failure()` (L416-490)** — Rate-Limit Bulk Reschedule:

```python
if error_msg == "rate_limit" and item.get("campaign_id"):
    await self.db.messages.update_many(
        {
            "campaign_id": item["campaign_id"],
            "status": {"$in": ["pending", "retry_wait", "processing"]},
            "id": {"$ne": item_id},
        },
        {"$set": {
            "next_attempt_at": next_attempt_at,
            "status": "retry_wait",
        }},
    )
```

**Why bulk reschedule on rate_limit?** If WhatsApp rate-limits one message, it will rate-limit all subsequent messages too. Instead of wasting API calls (and risking a ban), the scheduler proactively pushes **all remaining messages in that campaign** to the next retry window — saving network calls and protecting the account.

**`_recover_orphans()` (L217-259)** — Deadlock Prevention:

Detects messages stuck in `processing` for >60 seconds (caused by server crashes, hard timeouts, or unhandled exceptions). For each orphan:
- If `attempt_count >= MAX_RETRY_COUNT` → move to `failed_permanently`
- Otherwise → reset to `retry_wait` with incremented attempt count

**`_auto_complete_campaigns()` (L510-540)** — Dynamic Macro Consolidation:

When the queue is empty, scans for campaigns still marked `"sending"`. For each, counts remaining active messages. If 0 → flips status to `"completed"`. This prevents the frontend from showing an infinite loading spinner.

---

### 4.6 Supporting Critical Files

#### `batch_service.py` (777 lines) — Campaign Orchestration

**`create_batch()` (L42-325)**:
- Splits customer list into micro-batches of configurable size
- For each customer: resolves segment → selects template → hydrates 8 behavioral placeholders + 3 offer placeholders → creates message record
- **Segment-based template routing**: Different segments can receive different message templates in the same campaign
- **Priority mapping**: VIP/At-Risk → priority 1, Potential Bulk → 2, Loyal Frequent → 3, Boring → 4

#### `offers_service.py` (299 lines) — Composite Affinity Rank Engine

**Affinity Formula** (L146-187):
```
Affinity(customer, offer) = S(c, o) × [1.0 + P(c, o)]

S = 1 if customer.segment ∈ offer.target_segments, else 0   (binary gate)
P = 3.0 × I(product_match) + 1.0 × I(category_match)

product_match: any offer product ∈ {customer's favorite premium, bulk, or recent purchase}
category_match: offer.category == customer.favorite_category
```

**`match_offers_to_customers()` (L189-249)**: O(C × O) brute-force ranking. For each customer × offer pair, compute affinity score. Keep the argmax per customer.

#### `database.py` (315 lines) — Index Strategy

Defines **40+ MongoDB indexes** across 12 collections. Key strategic indexes:
- `scheduler_poll_query`: Compound index on `(status, next_attempt_at)` — the scheduler's hot path query
- `unique_batch_customer_message`: Prevents duplicate messages per customer per batch
- `unique_file_content_hash`: SHA-256 content dedup across file uploads
- `unique_shop_customer_phone`: Prevents duplicate customer records per shop

#### `classifier.py` (499 lines) — RFM+B Classifier & File Parser

Contains the standalone RFM+B classifier (used during file upload) and the multi-format file parser (CSV/XLSX/PDF). The `prepare_message()` function handles template placeholder substitution via simple string `replace()`.

---

## 5. The Perfect Interview Storytelling Framework (STAR Method)

### The Verbal Pitch

> *"Let me walk you through a project I built end-to-end — a high-concurrency WhatsApp CRM and automation platform for Indian retail businesses."*

---

#### S — Situation (The Problem)

> *"Indian wholesale and retail shop owners — think neighborhood kirana stores — rely on WhatsApp as their primary customer communication channel. But they do everything manually: scrolling through contact lists, copy-pasting the same generic 'Sale today!' message to hundreds of customers. A 200-customer campaign takes an entire working day.*
>
> *The bigger problem is data. Their purchase history is trapped in messy CSV exports from billing software like Tally or Busy — inconsistent column names, mixed date formats, unstandardized phone numbers. They have zero insight into who their VIP customers are, who's at risk of churning, or what products to recommend to whom.*
>
> *And the scariest part: WhatsApp aggressively bans accounts that send bulk messages. One bad campaign — too many messages, too fast, outside business hours — and the shop owner permanently loses their primary customer channel."*

---

#### T — Task (The Complication)

> *"I needed to build a system that solves three simultaneous engineering challenges:*
>
> *First, the data pipeline challenge. I had to ingest raw CSV, Excel, and even PDF files with completely arbitrary column schemas — auto-detect what each column represents, and let the user correct the mapping. Then transform that raw data into actionable customer intelligence.*
>
> *Second, the analytics challenge. Standard RFM segmentation breaks on skewed distributions. If you have one whale customer who spent ₹500,000 and everyone else is under ₹10,000, a naive quintile cut puts 80% of customers into the bottom tier — which is useless for targeting. I also needed to handle degenerate cases where most customers have identical recency or frequency values, which crashes standard `pd.qcut`.*
>
> *Third, the concurrency challenge. The message sender runs as a background worker inside the FastAPI event loop. I had to prevent race conditions where two poll cycles grab the same message, prevent deadlocks where a crashed message stays stuck in 'processing' forever, and implement anti-spam throttling — all without blocking the API server."*

---

#### A — Action (The Strategy & Architecture)

> *"I designed a three-layer architecture:*
>
> *For the data pipeline, I built an ingestion service with SHA-256 content-hash deduplication, a flexible column mapper that handles field name variations, and a two-level insight engine. Level 1 computes RFM+B scores using log-damped monetary values and a 3-tier fallback for quintile binning — `qcut` → `cut` → flat constant — so no data shape can crash the pipeline. Level 2 is a pure-Pandas behavioral profiler that computes 8 template variables per customer using a weighted category affinity formula and co-purchase market basket analysis.*
>
> *For the message engine, I implemented a deterministic 6-state machine: pending → processing → sent/retry_wait/failed_permanently/cancelled. The key innovation is using MongoDB's atomic `update_one` with a status precondition as a distributed lock — only one worker can transition a message from pending to processing, eliminating duplicate sends without needing Redis. I added three layers of deadlock prevention: a 45-second `asyncio.wait_for` timeout, a `try/finally` release guard, and a background orphan recovery scan.*
>
> *For the messaging infrastructure, I used the Adapter Pattern to make the transport layer swappable. In development, a DummyGateProvider uses MD5 hashing to produce deterministic delivery outcomes — same phone numbers always produce the same success/failure distribution. In production, a Playwright-based WhatsApp Web sender automates the browser with persistent session caching, working-hours enforcement, daily send caps, and a circuit breaker that triggers on 5 consecutive failures."*

---

#### R — Result (The Resolution & Execution)

> *"The final system processes 200-customer campaigns in under 5 minutes with full segment-targeted messaging, automatic retry with exponential backoff, and real-time monitoring with per-batch, per-segment drill-down.*
>
> *The data pipeline transforms raw, messy CSVs into 8 personalized template variables per customer — so instead of 'Sale today!', a VIP customer gets 'Hi Raman, we have 20% off on Tata Premium Tea — your favorite in our Premium Tea collection. Pair it with Bru Coffee for a great morning combo!'*
>
> *The deterministic testing mode means every demo produces identical results — I can predict exactly which 4 customers will permanently fail, which 20 will retry once and succeed, and which 175 will deliver instantly. That kind of reproducibility is invaluable for QA.*
>
> *And the anti-spam safety net — working hours, daily caps, jitter, circuit breaker — has kept zero accounts banned across all test runs."*

---

## 6. Potential Interview Questions & Model Answers

---

### Q1: Race Conditions — How do you prevent duplicate message sends when two poll cycles overlap?

**Model Answer**:

> *"I use a three-layer defense. The first layer is the `self._processing` boolean guard in the poll cycle — APScheduler fires every 7 seconds, but if the previous cycle is still running, the new one is silently skipped.*
>
> *The second, more critical layer is the atomic MongoDB concurrency lock. When the worker picks up a message, it immediately does an `update_one` with a compound filter: `{id: item_id, status: {$in: ['pending', 'retry_wait']}}`. It atomically sets the status to `processing`. If two workers try this simultaneously on the same message, MongoDB guarantees that only one gets `modified_count == 1` — the other gets `0` and returns immediately. This is effectively a distributed compare-and-swap lock.*
>
> *The third layer is the `try/finally` fallback: after processing completes, the finally block checks if the message is still in `processing` state — which would mean the state transition handlers somehow failed silently. If it is, it force-transitions to `failed_permanently` rather than leaving it stuck."*

---

### Q2: Memory Bottlenecks — How do you handle large transaction datasets in Pandas without running out of memory?

**Model Answer**:

> *"Currently, the `recalculate_all_insights` function loads all transactions for a shop into RAM as a Pandas DataFrame. For a typical small retail shop — maybe 10,000–50,000 transaction rows — this works fine, typically using 20–100MB of RAM.*
>
> *For scale, I'd implement three mitigations: First, chunked cursor loading with `batch_size` in the MongoDB query, processing customer groups incrementally rather than loading the entire shop at once. Second, column projection — the current query uses `{_id: 0}` but fetches all fields; I'd project only the 6 columns needed (customer_id, product_id, purchase_date, quantity, amount, category). Third, I'd move the per-customer `groupby` operations into MongoDB aggregation pipelines — the database is optimized for this and would eliminate the need to materialize the full DataFrame in Python memory.*
>
> *But for the current scale — this is a kirana store CRM, not a Fortune 500 analytics platform — the in-memory approach is the right tradeoff: simpler code, faster development, and Pandas' expressive API makes the affinity formulas and fallback cascades readable."*

---

### Q3: Playwright Automation Stability — What happens if WhatsApp Web changes their UI selectors?

**Model Answer**:

> *"This is the Achilles' heel of any browser automation approach. I mitigate it in several ways:*
>
> *First, I use WhatsApp's deep link API (`web.whatsapp.com/send?phone=...&text=...`) rather than navigating the DOM to find chat windows. This URL-based approach is less fragile than DOM traversal because WhatsApp maintains backward compatibility with their deep link format.*
>
> *Second, I use a single CSS selector — `button[aria-label="Send"]` — which targets an accessibility attribute rather than a class name. Accessibility attributes are less likely to change because WhatsApp needs to maintain WCAG compliance.*
>
> *Third, the circuit breaker pattern. If 5 consecutive messages fail (which would happen if a selector breaks), the sender automatically pauses and reschedules the entire campaign to the next day. This prevents burning through the daily quota on guaranteed failures.*
>
> *Fourth, the Adapter Pattern means I can switch to the Twilio WhatsApp Business API or any other provider by changing a single environment variable — the scheduler doesn't know or care which provider is active. If Playwright becomes unreliable, the swap is a config change, not a code change."*

---

### Q4: MongoDB Bulk Operations — Why `bulk_write(ordered=False)` instead of individual atomic writes or a `delete_many + insert_many` approach?

**Model Answer**:

> *"Three reasons: performance, atomicity, and fault tolerance.*
>
> *Performance: 200 individual `update_one` calls mean 200 round-trips to MongoDB. `bulk_write` batches them into a single network round-trip with a single write-concern acknowledgment. For a 200-customer shop, this cuts insight recalculation from ~2 seconds to ~200ms.*
>
> *Why not `delete_many` + `insert_many`? Because that's not atomic. Between the delete and the insert, the `customer_insights` collection is empty — any concurrent API request would get zero results. The upsert approach (`$set` with `upsert=True`) ensures the collection always has data; documents are updated in-place.*
>
> *`ordered=False` is critical for fault tolerance. With `ordered=True`, if document #50 fails (say, a validation error), documents #51–200 are skipped. With `ordered=False`, MongoDB processes all 200 independently — a single failure doesn't cascade. The error is logged, but the other 199 customers still get their insights updated."*

---

### Q5: Why a 5-tier waterfall instead of a standard RFM matrix (e.g., 5×5×5 = 125 cells)?

**Model Answer**:

> *"A 125-cell RFM matrix is great for enterprise analytics dashboards, but it's terrible for a small business CRM where the output needs to be actionable. A shop owner doesn't know what to do with 'R=3, F=4, M=2'. They need to know: 'This is a VIP — send the premium offer' or 'This customer is at risk of churning — send a win-back discount.'*
>
> *The 5-tier waterfall also solves a distribution problem. With small datasets (100–200 customers), a 125-cell matrix produces mostly empty cells. The waterfall compresses the score space into 5 meaningful segments with explicit business rules: VIP is the intersection of high total score AND high monetary or frequency. At-Risk specifically catches formerly-valuable customers whose recency has dropped to the bottom 20%. Potential Bulk uses the bulkiness dimension — a fourth factor beyond standard RFM — to identify pantry-stocking customers who buy large quantities.*
>
> *The waterfall order matters too. Rules are evaluated top-to-bottom, first match wins. VIP is checked first because a high-value customer should never be accidentally classified as At-Risk even if their recency is poor — they might be a seasonal bulk buyer. This prioritized evaluation is more intuitive than a matrix lookup."*

---

### Q6: How does the DummyGateProvider's MD5 hashing ensure a uniform distribution? Why not just use `random`?

**Model Answer**:

> *"`random` would give different outcomes on every run — you couldn't reproduce a bug or demonstrate a specific failure scenario reliably. MD5 is a cryptographic hash with the avalanche property: a single-bit change in input produces a completely different output. The modulo-100 of the last 6 hex digits (24 bits of entropy, range 0–16,777,215) distributes uniformly across 100 buckets.*
>
> *The phone number is normalized before hashing — stripping spaces, dashes, and parentheses — so '+91 98765 43210' and '919876543210' hash to the same bucket. This prevents formatting inconsistencies from producing different delivery outcomes for the same phone number.*
>
> *For a 200-customer dataset, the expected bucket distribution is: 2% terminal (4 customers), 0.5% network failure (1 customer), 10% retry-then-succeed (20 customers), 87% clean success (175 customers). This ratio closely mimics real-world WhatsApp delivery statistics, making the demo representative of production behavior."*

---

### Q7: What's the risk of the in-memory daily counter in `WhatsAppWebSender` resetting on server restart?

**Model Answer**:

> *"It's a real risk but an acceptable tradeoff for the current architecture. If the server restarts mid-day, the counter resets to 0, potentially allowing more than 200 messages in a single day.*
>
> *To fully fix this, I'd store the daily counter in MongoDB — an atomic `$inc` on a document like `{date: '2026-07-09', count: 150}`. But that adds a database round-trip to every single message send, which is a latency cost I chose to avoid for a single-instance deployment.*
>
> *The practical risk is low: server restarts are rare during business hours, and even if the counter resets, the other safety layers (working-hours gate, inter-message jitter, circuit breaker) still protect against ban-level behavior. For a multi-instance deployment, I'd absolutely move to a shared counter — either in MongoDB or Redis."*

---

### Q8: How would you scale this system to handle 10,000+ customers per shop?

**Model Answer**:

> *"Three bottlenecks need addressing:*
>
> *First, the insight pipeline. The current approach loads all transactions into a Pandas DataFrame in RAM. At 10,000 customers × 50 transactions each = 500,000 rows, that's ~200MB — still feasible on a single machine. But I'd refactor the aggregation (Step 3 in `recalculate_all_insights`) into a MongoDB aggregation pipeline — `$group` by customer_id with `$max(purchase_date)`, `$sum(amount)`, `$count` — which is O(N) on the database side with index-backed scans, eliminating the need for the DataFrame entirely for Level 1.*
>
> *Second, the scheduler throughput. At 8 messages/cycle × 1 cycle/7s = ~69 messages/minute, a 10,000-customer campaign takes ~145 minutes. I'd increase `MICRO_BATCH_SIZE` to 50 and reduce `POLL_INTERVAL_SECONDS` to 3, giving ~1,000 messages/minute. For true horizontal scale, I'd use MongoDB's Change Streams as a push-based trigger instead of polling.*
>
> *Third, the behavioral profiler. The `_complementary_product` function does a full DataFrame scan for co-purchase dates. At 500,000 transaction rows, this is O(N). I'd pre-compute a co-purchase matrix as a scheduled job and cache it in MongoDB — turning the runtime lookup into an O(1) dictionary read."*

---

### Q9: Why FastAPI instead of Django or Flask?

**Model Answer**:

> *"Three reasons: async-native, performance, and the background worker pattern.*
>
> *FastAPI is built on Starlette/ASGI, which means every request handler is an async coroutine running on an `asyncio` event loop. This is critical because my MongoDB driver (Motor) and my Playwright browser automation are both async. With Flask/Django, I'd need `asyncio.run()` wrappers or threading — adding complexity and losing the single-event-loop benefit.*
>
> *The background worker pattern is the second reason. The `SchedulerWorker` runs on the same `asyncio` event loop as the HTTP server via APScheduler's `AsyncIOScheduler`. This means the worker shares the same Motor connection pool and can directly access `self.db` without cross-process serialization. In Flask, I'd need Celery + Redis as a separate process — massive operational overhead for what's fundamentally a simple polling loop.*
>
> *Performance: FastAPI with uvicorn is benchmarked at 2-3x the throughput of Flask for I/O-bound workloads, which matters when the dashboard is polling for real-time batch status updates."*

---

### Q10: How do you handle the case where a customer appears in multiple uploaded files with different data?

**Model Answer**:

> *"The system uses a multi-layer deduplication strategy:*
>
> *At the file level, SHA-256 content hashing prevents the same file from being processed twice. The unique index is `(user_id, shop_id, data_purpose, content_hash)` — so uploading the exact same CSV twice returns the existing file_id instead of creating a duplicate.*
>
> *At the customer level, the unique index is `(shop_id, phone)` — phone number is the natural key for identity. If a customer appears in a new file with updated data, the upsert strategy in `customer_service.py` updates their record in place.*
>
> *At the transaction level, the unique compound index on `(shop_id, customer_id, product_id, purchase_date, purchase_qty, total_amount)` prevents duplicate transaction rows. Period-scoped uploads use `period_tag` to replace all transactions for a given period, then rebuild insights — this is the 'replace-and-regenerate' pattern rather than incremental merging, which eliminates the complexity of conflict resolution."*

---

*End of Master Breakdown. Good luck with the interview — you've built something genuinely sophisticated. Own every line.*
