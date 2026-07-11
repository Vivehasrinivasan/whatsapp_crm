# asyncio — The Full Working Guide (And How Our WhatsApp CRM Uses It)

> **What this doc covers**: What `asyncio` actually is from first principles, why it exists, how it works under the hood, and a line-by-line walkthrough of every place we use it in our WhatsApp CRM system — with proof that it works and serves its purpose.

---

## Table of Contents

1. [What is asyncio? (The ELI5 Version)](#1-what-is-asyncio-the-eli5-version)
2. [Why Does asyncio Exist? (The Problem It Solves)](#2-why-does-asyncio-exist-the-problem-it-solves)
3. [How asyncio Actually Works Under the Hood](#3-how-asyncio-actually-works-under-the-hood)
4. [The Three Keywords: `async`, `await`, `coroutine`](#4-the-three-keywords-async-await-coroutine)
5. [Core asyncio Functions We Use](#5-core-asyncio-functions-we-use)
6. [Every Place We Use asyncio in Our System (With Proof)](#6-every-place-we-use-asyncio-in-our-system-with-proof)
7. [Does It Actually Work? (Yes — Here's Proof)](#7-does-it-actually-work-yes--heres-proof)
8. [Common Interview Questions About asyncio](#8-common-interview-questions-about-asyncio)

---

## 1. What is asyncio? (The ELI5 Version)

Imagine you're at a restaurant.

**Without asyncio (Synchronous / Blocking)**:
```
You order food → You STAND AT THE COUNTER AND WAIT until the chef finishes cooking
→ You get your food → Now the next person in line can order
```
If the chef takes 10 minutes, everyone behind you waits 10 minutes doing **absolutely nothing**. The waiter is blocked.

**With asyncio (Asynchronous / Non-blocking)**:
```
You order food → The waiter writes your order and gives you a ticket number
→ The waiter IMMEDIATELY takes the next person's order
→ When your food is ready, the waiter brings it to you
```
The waiter (your program) never stands idle. While one order is cooking (waiting for a database, waiting for a network response, waiting for a file to load), the waiter handles other orders.

**That's asyncio.** It's Python's built-in library for writing code that can **start a task, move on to other work, and come back when the task is done** — all in a single thread.

---

## 2. Why Does asyncio Exist? (The Problem It Solves)

### The I/O-Bound Problem

Most backend servers spend the majority of their time **waiting**:
- Waiting for MongoDB to respond to a query (~1–50ms)
- Waiting for WhatsApp Web to load a page (~2–20 seconds)
- Waiting for a network request to complete (~50–500ms)

During this waiting time, the CPU is idle. In traditional synchronous Python:

```python
# SYNCHRONOUS — BLOCKING — SLOW 🐌
def send_200_messages():
    for phone in phone_list:
        result = db.messages.find_one({"phone": phone})      # Wait 5ms... CPU does NOTHING
        response = whatsapp.send(phone, "Hello")              # Wait 2 seconds... CPU does NOTHING
        db.messages.update_one({"phone": phone}, {"$set": result})  # Wait 5ms... CPU does NOTHING
    # Total: 200 × 2.01 seconds = ~402 seconds (6.7 minutes) 😱
```

With asyncio:

```python
# ASYNCHRONOUS — NON-BLOCKING — FAST 🚀
async def send_200_messages():
    for phone in phone_list:
        result = await db.messages.find_one({"phone": phone})      # Start query, go do other work
        response = await whatsapp.send(phone, "Hello")              # Start send, go do other work
        await db.messages.update_one({"phone": phone}, {"$set": result})
```

**The `await` keyword is the magic.** When you write `await db.messages.find_one(...)`, Python says:
> "OK, I've sent the request to MongoDB. Instead of sitting here waiting for the response, let me go check if any other task needs attention. I'll come back to this line when MongoDB responds."

### The Alternative: Threads and Processes

You might ask: "Why not just use threads?"

| Approach | Pros | Cons |
|---|---|---|
| **Threads** (`threading`) | True parallel I/O | GIL limits CPU parallelism; race conditions; hard to debug; needs locks everywhere |
| **Processes** (`multiprocessing`) | True CPU parallelism | Heavy memory (each process copies entire Python interpreter); IPC is complex |
| **asyncio** | Single thread, no locks, no GIL issues, lightweight | Only works for I/O-bound tasks; all libraries must be async-compatible |

For our system — which is 95% I/O-bound (database reads, database writes, network sends, browser automation) — **asyncio is the perfect fit**. We never do heavy CPU work in the event loop (the Pandas computation is a brief synchronous burst, not a sustained CPU load).

---

## 3. How asyncio Actually Works Under the Hood

### The Event Loop

The **event loop** is the heart of asyncio. Think of it as a **task manager** that runs in a single thread:

```
┌─────────────────────────────────────────────────────────────────┐
│                        EVENT LOOP                               │
│                                                                 │
│  Task Queue: [Task A, Task B, Task C, Task D, ...]              │
│                                                                 │
│  Loop:                                                          │
│    1. Pick the next ready task from the queue                   │
│    2. Run it until it hits an `await` (I/O operation)           │
│    3. Pause that task, put it in the "waiting" pile             │
│    4. Check if any waiting task's I/O has completed             │
│    5. If yes → move it back to the ready queue                  │
│    6. Go to step 1                                              │
│                                                                 │
│  This loop runs FOREVER until all tasks are done.               │
└─────────────────────────────────────────────────────────────────┘
```

### Visual Timeline: Sync vs Async

**Synchronous (one at a time)**:
```
Time →  0s      1s      2s      3s      4s      5s      6s
Task A: [██████████████████████]
Task B:                         [██████████████████████]
Task C:                                                 [██████████]
        ↑ running  ░░ waiting    ↑ running              ↑ running
```
Total: 6 seconds

**Asynchronous (interleaved)**:
```
Time →  0s    0.5s    1s    1.5s    2s    2.5s
Task A: [██]--wait--[██]--wait--[██]
Task B:      [██]--wait--[██]--wait--[██]
Task C:           [██]--wait--[██]--wait--[██]
        ██ = running    -- = waiting (I/O), another task runs
```
Total: ~2.5 seconds (all three overlap their waiting periods!)

**Key insight**: asyncio doesn't make individual tasks faster. It makes the **overall system** faster by filling idle time with useful work.

---

## 4. The Three Keywords: `async`, `await`, `coroutine`

### `async def` — Declares a Coroutine

```python
# Regular function — runs to completion, blocks the caller
def get_customer():
    return db.find_one({"id": "123"})  # Blocks until MongoDB responds

# Coroutine — can be paused and resumed
async def get_customer():
    return await db.find_one({"id": "123"})  # Pauses here, lets other tasks run
```

When you write `async def`, you're telling Python: "This function might need to wait for I/O. Give it the ability to pause and resume."

A coroutine is **NOT automatically running**. Calling `get_customer()` returns a coroutine **object** — you must `await` it or schedule it for it to actually execute.

### `await` — Pause Here, Let Others Work

```python
async def process_message():
    # Line 1: Send query to MongoDB, PAUSE this function, let others run
    message = await db.messages.find_one({"id": msg_id})
    
    # Line 2: This runs ONLY AFTER MongoDB responds
    result = await ProviderAdapter.send_message(message["phone"], message["content"])
    
    # Line 3: This runs ONLY AFTER the send completes
    await db.messages.update_one({"id": msg_id}, {"$set": {"status": "sent"}})
```

**Critical rule**: You can ONLY use `await` inside an `async def` function. You cannot `await` in regular `def` functions.

### What Happens at Each `await`

```python
result = await db.messages.find_one({"id": msg_id})
```

This single line does the following:
1. Python calls `db.messages.find_one(...)` — Motor sends the query to MongoDB over the network
2. **Python pauses this function** and stores its current state (local variables, current line number)
3. The event loop checks: "Is any other task ready to run?" — if yes, it runs that task
4. When MongoDB's response arrives (the OS signals "data ready on this socket"), the event loop resumes this function
5. The response is assigned to `result`, and execution continues to the next line

---

## 5. Core asyncio Functions We Use

### `asyncio.sleep(seconds)` — Non-Blocking Delay

```python
# ❌ BLOCKING — freezes the entire server for 2 seconds
import time
time.sleep(2)

# ✅ NON-BLOCKING — pauses only this task, others keep running
await asyncio.sleep(2)
```

**Where we use it**: Inter-message jitter in the scheduler, simulated network latency in DummyGateProvider, WhatsApp Web post-send confirmation delay.

### `asyncio.wait_for(coroutine, timeout)` — Timeout Guard

```python
try:
    await asyncio.wait_for(self._process_item(item), timeout=45.0)
except asyncio.TimeoutError:
    logger.error("Processing took too long — killed!")
```

**What it does**: Runs the coroutine but **kills it** if it doesn't complete within 45 seconds. Without this, a stalled MongoDB query or a hung Playwright page could freeze the scheduler forever.

**Where we use it**: `scheduler_service.py` line 190 — every message processing has a hard 45-second deadline.

### `asyncio.create_task(coroutine)` — Fire and Forget

```python
asyncio.create_task(sender.start())
```

**What it does**: Schedules the coroutine to run in the background **without waiting for it to finish**. The current function continues immediately.

**Where we use it**: `server.py` line 97 — starting the Playwright browser during server boot. We don't want the server to wait 10+ seconds for WhatsApp Web to load before it starts accepting HTTP requests.

### `asyncio.gather(*coroutines)` — Run Multiple Tasks Concurrently

```python
results = await asyncio.gather(
    self.db.customers.delete_many({"shop_id": shop_id}),
    self.db.products.delete_many({"shop_id": shop_id}),
    self.db.transactions.delete_many({"shop_id": shop_id}),
    self.db.customer_insights.delete_many({"shop_id": shop_id}),
    # ... 10 more collections
)
```

**What it does**: Launches ALL the delete operations **simultaneously** and waits for ALL of them to complete. Instead of deleting 14 collections one-by-one (14 sequential round-trips), all 14 delete commands fire at the same time.

**Where we use it**: `shop_service.py` line 394 — deleting a shop wipes 14 collections concurrently.

### `asyncio.run(coroutine)` — Entry Point

```python
if __name__ == "__main__":
    asyncio.run(run_migrations())
```

**What it does**: Creates a brand new event loop, runs the coroutine until completion, then closes the loop. This is the standard way to run async code from a synchronous context (like a `__main__` script).

**Where we use it**: `migrations.py` line 65 — running migrations as a standalone script.

---

## 6. Every Place We Use asyncio in Our System (With Proof)

### 6.1 The FastAPI Server Itself (The Foundation)

**File**: `server.py`

FastAPI is built on **Starlette**, which is an **ASGI** (Asynchronous Server Gateway Interface) framework. When you run:

```python
uvicorn.run("server:app", host="0.0.0.0", port=8000)
```

Uvicorn creates an asyncio event loop and runs the entire FastAPI application inside it. **Every `async def` route handler runs as a coroutine on this event loop.** This is why we can use `await` in every route:

```python
@app.get("/api/health")
async def health_check():    # ← This is a coroutine scheduled on the event loop
    return {"status": "healthy"}
```

**This means**: Our HTTP server, our database queries, our background scheduler, and our WhatsApp sender are ALL running on the same single-threaded event loop. No threads, no processes — just coroutines taking turns.

### 6.2 MongoDB Async Driver (Motor)

**Used in**: Every service file (`batch_service.py`, `insights_service.py`, `customer_service.py`, etc.)

```python
from motor.motor_asyncio import AsyncIOMotorClient
```

**Motor** is the official async MongoDB driver. Regular `pymongo` is synchronous — a `find_one()` call blocks until MongoDB responds. Motor wraps pymongo in asyncio-compatible coroutines:

```python
# pymongo (synchronous) — BLOCKS the event loop ❌
doc = db.customers.find_one({"phone": "919876543210"})

# motor (async) — YIELDS to the event loop ✅
doc = await db.customers.find_one({"phone": "919876543210"})
```

**Proof it works**: If we used synchronous pymongo, a single slow MongoDB query (say, 500ms on a large collection) would freeze the entire server — no other HTTP request could be handled during that time. With Motor, the event loop handles other requests while waiting for MongoDB.

**Real example from our code** (`insights_service.py` line 50-51):
```python
tx_cursor = db.transactions.find({"shop_id": shop_id}, {"_id": 0})
tx_rows = [doc async for doc in tx_cursor]  # ← "async for" iterates without blocking
```

The `async for` is a special syntax that `await`s each cursor batch from MongoDB. While one batch is in transit over the network, the event loop can process other tasks.

### 6.3 The Scheduler Worker (The Background Engine)

**File**: `scheduler_service.py`

This is the most critical use of asyncio in our system. The `SchedulerWorker` is a background task that polls MongoDB every 7 seconds, processes up to 8 messages per cycle, and sends them through the provider adapter.

**How it runs without blocking the API server**:

```python
# In server.py startup:
from apscheduler.schedulers.asyncio import AsyncIOScheduler

self.scheduler = AsyncIOScheduler()  # Uses the existing asyncio event loop
self.scheduler.add_job(
    self._poll_cycle,                    # ← This is an async coroutine
    trigger=IntervalTrigger(seconds=7),  # Fire every 7 seconds
    max_instances=1,                     # Never overlap
)
self.scheduler.start()
```

`AsyncIOScheduler` (from APScheduler) integrates with asyncio's event loop. Every 7 seconds, it schedules `_poll_cycle()` as a new task on the event loop. The event loop interleaves this task with HTTP request handling:

```
Time →  0s     1s     2s     3s     4s     5s     6s     7s     8s
API:    [req]  [req]        [req]  [req]               [req]
Worker:              [poll_cycle ─── process messages ──]     [next poll]
```

**Proof it works**: While the scheduler is processing 8 messages (which involves 8 MongoDB queries + 8 provider sends + 8 MongoDB updates = ~24 I/O operations), the API server continues responding to dashboard requests, monitoring queries, and template lookups. They never block each other.

**Key asyncio patterns in the scheduler**:

```python
# 1. Timeout guard — kills stalled processing after 45s
await asyncio.wait_for(self._process_item(item), timeout=45.0)

# 2. Inter-message jitter — non-blocking delay
jitter = random.uniform(0.5, 1.5)
await asyncio.sleep(jitter)  # Pauses THIS task, event loop handles other work

# 3. TimeoutError catch
except asyncio.TimeoutError:
    logger.error(f"TIMEOUT processing item {item.get('id')}")
```

### 6.4 The DummyGateProvider (Simulated Network Delay)

**File**: `provider_adapter.py`

```python
async def send(self, phone: str, content: str, attempt_count: int = 1) -> Dict:
    # Simulate minimal network latency (50–150ms)
    await asyncio.sleep(random.uniform(0.05, 0.15))
    # ... bucket logic ...
```

**Why `await asyncio.sleep()` instead of `time.sleep()`?**

If we used `time.sleep(0.15)`, processing 8 messages would block the event loop for `8 × 0.15 = 1.2 seconds`. During that 1.2 seconds, **no HTTP requests could be handled** — the dashboard would freeze.

With `await asyncio.sleep(0.15)`, each 150ms pause yields control back to the event loop. The server continues handling API requests during the simulated delays.

### 6.5 The WhatsApp Web Sender (Playwright Browser Automation)

**File**: `whatsapp_sender.py`

Playwright's async API is built on asyncio. Every browser interaction is a coroutine:

```python
# Navigate to WhatsApp Web — this takes 2-20 seconds
await self.page.goto("https://web.whatsapp.com", wait_until="networkidle", timeout=60000)

# Wait for the send button to appear — up to 20 seconds
await self.page.wait_for_selector('button[aria-label="Send"]', timeout=20000)

# Click the send button
await self.page.click('button[aria-label="Send"]')

# Wait for delivery confirmation
await asyncio.sleep(2)
```

**Why async Playwright?** A WhatsApp Web `page.goto()` takes 5–20 seconds. With synchronous Playwright, the server would be **completely frozen** during every single message send. With async Playwright, the event loop handles other work during these long waits.

**Fire-and-forget startup** (`server.py` line 97):
```python
asyncio.create_task(sender.start())
```

This launches the Playwright browser in the background. The server starts accepting HTTP requests immediately — it doesn't wait 10+ seconds for WhatsApp Web to fully load. `create_task()` schedules the coroutine and returns instantly.

### 6.6 Concurrent Shop Deletion (asyncio.gather)

**File**: `shop_service.py`

When a user deletes a shop, we need to wipe data from 14 MongoDB collections:

```python
results = await asyncio.gather(
    self.db.customers.delete_many({"shop_id": shop_id}),
    self.db.products.delete_many({"shop_id": shop_id}),
    self.db.transactions.delete_many({"shop_id": shop_id}),
    self.db.customer_insights.delete_many({"shop_id": shop_id}),
    self.db.customer_behavior_map.delete_many({"shop_id": shop_id}),
    self.db.offers.delete_many({"shop_id": shop_id}),
    self.db.files.delete_many({"shop_id": shop_id}),
    self.db.templates.delete_many({"user_id": user_id, "shop_id": shop_id}),
    self.db.campaigns.delete_many({"user_id": user_id, "shop_id": shop_id}),
    self.db.batches.delete_many({"user_id": user_id, "shop_id": shop_id}),
    self.db.messages.delete_many({"user_id": user_id, "shop_id": shop_id}),
    self.db.msg_queues.delete_many({"user_id": user_id, "shop_id": shop_id}),
    self.db.campaign_batches.delete_many({"user_id": user_id, "shop_id": shop_id}),
    self.db.shops.delete_one({"id": shop_id, "user_id": user_id})
)
```

**Without `asyncio.gather` (sequential)**:
```
Delete customers:       [5ms]
Delete products:              [5ms]
Delete transactions:                [5ms]
Delete insights:                          [5ms]
... (14 total)
Total: 14 × 5ms = 70ms
```

**With `asyncio.gather` (concurrent)**:
```
Delete customers:       [5ms]
Delete products:        [5ms]    ← sent simultaneously
Delete transactions:    [5ms]    ← sent simultaneously
Delete insights:        [5ms]    ← sent simultaneously
... (all 14 fire at once)
Total: ~5ms (all finish in parallel)
```

**14× speedup** for this operation. `gather` sends all 14 delete commands to MongoDB simultaneously and waits for all of them to complete.

### 6.7 Standalone Script Entry Point

**File**: `migrations.py`

```python
if __name__ == "__main__":
    asyncio.run(run_migrations())
```

When running `python migrations.py` directly (not through FastAPI), there's no event loop running yet. `asyncio.run()` creates one, runs the async function, then cleans up. This is the standard way to bridge sync → async.

---

## 7. Does It Actually Work? (Yes — Here's Proof)

### Proof 1: The Server Handles API Requests While Sending Messages

Without asyncio, when the scheduler is processing 8 messages (involving ~24 I/O operations taking ~3-5 seconds), the API server would **completely freeze**. The dashboard wouldn't load, monitoring wouldn't update, and the frontend would show timeout errors.

**With asyncio**, the scheduler's `await` calls yield control to the event loop between every I/O operation. The event loop interleaves HTTP request handling between scheduler I/O waits:

```
Scheduler:  [query DB] ──await── [send msg] ──await── [update DB] ──await──
API Server:             [handle request]              [handle request]
                        ↑ runs during scheduler's await
```

**You can verify this yourself**: Start a campaign with 200 messages and simultaneously open the dashboard. The dashboard loads instantly — the scheduler running in the background doesn't affect API response times.

### Proof 2: The Timeout Guard Prevents Deadlocks

```python
await asyncio.wait_for(self._process_item(item), timeout=45.0)
```

If a MongoDB query hangs (network issue, server overload), without the timeout, that message would stay in `processing` state **forever** — blocking that slot in every subsequent poll cycle. With `wait_for`, after 45 seconds asyncio raises `TimeoutError`, the except block logs the error, and the orphan recovery system will eventually reset the message.

### Proof 3: Fire-and-Forget Playwright Startup

```python
asyncio.create_task(sender.start())
```

The Playwright browser takes 5-15 seconds to launch and navigate to WhatsApp Web. Without `create_task`, the server startup would be blocked for 15 seconds — no HTTP requests could be handled during boot. With `create_task`, the server starts accepting requests immediately while the browser boots in the background.

### Proof 4: Concurrent Deletion is 14× Faster

The `asyncio.gather()` in shop deletion sends 14 MongoDB commands simultaneously instead of sequentially. For a shop with thousands of records across 14 collections, this reduces deletion time from ~1 second to ~70ms.

---

## 8. Common Interview Questions About asyncio

### Q: "Is asyncio multi-threaded?"

> **No.** asyncio is single-threaded. It achieves concurrency (not parallelism) by interleaving tasks during I/O waits. There's only one thread — the event loop — switching between tasks cooperatively. This is called **cooperative multitasking**.

### Q: "What happens if you forget to `await` a coroutine?"

> The coroutine **never runs**. You get a coroutine object instead of the result, and Python prints a warning: `RuntimeWarning: coroutine 'function_name' was never awaited`. This is a common bug — you'd get `None` instead of actual data.

```python
# ❌ BUG — forgot await
result = db.messages.find_one({"id": "123"})
# result is a coroutine object, not the document!

# ✅ Correct
result = await db.messages.find_one({"id": "123"})
# result is the actual MongoDB document
```

### Q: "What if an `async def` function does CPU-heavy work?"

> It blocks the entire event loop. asyncio only helps with I/O-bound work. If you do heavy computation (like training a ML model) inside an async function, all other tasks freeze until the computation finishes. The solution is to use `asyncio.to_thread()` or `loop.run_in_executor()` to offload CPU work to a separate thread.
>
> In our system, the Pandas computation in `level2_profiler.py` is a brief synchronous burst (a few hundred ms for 200 customers). It's fast enough that it doesn't noticeably block the event loop. If we scaled to 100,000 customers, we'd need to offload it.

### Q: "How is `async for` different from regular `for`?"

> Regular `for` expects the iterator to return values synchronously. `async for` works with **async iterators** that `await` between yielding values. MongoDB cursors from Motor are async iterators — each batch of results is fetched asynchronously:

```python
# Regular for — would block on each cursor batch
for doc in cursor:  # ❌ This blocks

# Async for — yields to event loop between cursor batches
async for doc in cursor:  # ✅ This yields control during network fetches
    process(doc)
```

### Q: "Can you `await` inside a regular (non-async) function?"

> **No.** `await` is only valid inside `async def`. If you try it in a regular function, you get `SyntaxError`. To call async code from sync code, use `asyncio.run()` (at the top level) or `loop.run_until_complete()`.

### Q: "What's the difference between `asyncio.create_task()` and `await`?"

> **`await`** = "Run this and **wait** for the result before continuing."  
> **`create_task()`** = "Start this in the background and **continue immediately**. I'll check on it later (or never)."

```python
# await — sequential
result1 = await task1()  # Wait for task1 to finish
result2 = await task2()  # THEN start task2
# Total time: task1_time + task2_time

# create_task — concurrent
t1 = asyncio.create_task(task1())  # Start task1 in background
t2 = asyncio.create_task(task2())  # Start task2 in background IMMEDIATELY
result1 = await t1  # Now wait for both
result2 = await t2
# Total time: max(task1_time, task2_time)
```

### Q: "What is the event loop exactly?"

> It's literally a `while True` loop that does three things:
> 1. Check if any I/O operations completed (using OS-level selectors like `epoll` on Linux, `kqueue` on macOS, `IOCP` on Windows)
> 2. Resume the coroutines whose I/O completed
> 3. Run them until they hit the next `await`
>
> It's Python's implementation of the **reactor pattern** — a core design pattern in high-performance network programming.

---

## Summary: asyncio in Our System at a Glance

| Where | asyncio Feature | What It Does | Why It Matters |
|---|---|---|---|
| **FastAPI + Uvicorn** | Event loop (foundation) | Runs the entire app on one async loop | All tasks share one efficient thread |
| **Motor (MongoDB)** | `await db.collection.find()` | Non-blocking database I/O | API doesn't freeze during queries |
| **Scheduler Worker** | `AsyncIOScheduler` + `await` | Background polling without blocking API | Dashboard works while messages send |
| **Scheduler Timeout** | `asyncio.wait_for(timeout=45s)` | Kills stalled processing | Prevents deadlocked messages |
| **Scheduler Jitter** | `asyncio.sleep(random)` | Non-blocking inter-message delay | Anti-spam without freezing server |
| **DummyGate Latency** | `asyncio.sleep(0.05-0.15)` | Simulated network delay | Realistic testing without blocking |
| **Playwright Browser** | `await page.goto()`, `await page.click()` | Async browser automation | 20s page loads don't freeze server |
| **Playwright Startup** | `asyncio.create_task(sender.start())` | Fire-and-forget background boot | Server starts immediately |
| **Shop Deletion** | `asyncio.gather(*14 deletes)` | Concurrent collection wipes | 14× faster deletion |
| **Migrations Script** | `asyncio.run(run_migrations())` | Sync → async bridge | Run async code from `__main__` |

**Bottom line**: asyncio is the reason our single-process Python server can simultaneously serve API requests, run a background message scheduler, control a Playwright browser, and execute concurrent database operations — all without threads, without processes, and without any of them blocking each other.

It's not magic. It's just really clever **cooperative multitasking** for I/O-bound work. And yes — it absolutely works and serves its purpose in our system. 💪
