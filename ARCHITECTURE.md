# Parsley — Architecture & Evolution Blueprint

> A from-scratch distributed task queue, grown one patch at a time by
> replaying the real evolution of Celery (originally `crunchd` → `celery`).
> We re-derive design from history + the knowledge graph. We do **not** copy code.

Status: **v0 draft — living document.** Updated at every milestone.

---

## 1. Vision

Build a small, honest distributed task queue that can evolve into a mighty
codebase, in the same way Celery evolved from a tiny feed-fetcher into one of
the most-used task queues in Python. Each milestone is a small, verified step.
**The bug/feature the previous milestone leaves open IS the plan for the next.**

Overarching rule: **grow by uncommenting + hardening the seams we leave behind.**

---

## 2. Non-goals (deliberate choices)

- **No wholesale code copying.** We study celery's *design* and re-write it in our own words.
- **No "recreate modern Celery in one shot."** We get there by milestones, never by assembly-from-the-top.
- **Django is a plugin, not a core dependency.** The seed's growth must not be coupled to `django.conf.settings` or a DB engine (a documented deviation from celery M0).
- **Messaging transport is pluggable from day one.** No fixed AMQP. In-memory first; real transports later.

---

## 3. Intentional Deviation Ledger

Every place we knowingly diverge from celery's path. Kept honest, never hidden.

| # | Milestone/subsystem | Real celery did | Parsley does | Why |
|---|--------------------|-----------------|--------------|-----|
| D-1 | M0 | Django-coupled config + discovery | Pluggable config, no Django import at core | Testable without a DB; matches celery's *later* destination |
| D-2 | M0 | Fixed on carrot/AMQP | Transport is an interface (`BaseTransport`) | Lets us run workers in `pytest` from the first commit |
| D-3 | all | Identity/brand "celery" | Identity/brand "parsley" | This is our system, not a clone |

---

## 4. The Subsystem Map

How a tiny seed composes into the mighty codebase. Parentheses = **not built yet**.

```
parsley/
├── task.py          # Public API: delay_task(), discard_all(), Task()   [M0→M2]
├── registry.py      # TaskRegistry + NotRegistered / AlreadyRegistered   [M0]
├── messaging.py     # BaseTransport, TaskPublisher/TaskConsumer          [M0]
├── transports/      # in_memory (M0) → redis → amqp/kombu (later)       [M0+]
├── conf.py          # Settings + defaults (pluggable, not Django-locked) [M0]
├── discovery.py     # autodiscover(): find `<pkg>.tasks` modules         [M0]
├── worker.py        # TaskDaemon: consume → process-pool → results       [M0]
├── process.py       # ProcessQueue: ordered result collection            [M0]
├── platform.py      # PIDFile, daemonize                                 [M0]
├── bin/parsleyd     # CLI daemon (optparse → later argparse/click)       [M0]
├── app/             # (Celery facade, Task base class)                   [M2]
├── canvas.py        # (Signature, group, chain, chord)                   [later]
├── result.py        # (AsyncResult, ResultSet)                           [M2+]
├── beat.py          # (periodic scheduler)                               [M3]
└── backends/        # (result backends)                                  [M2+]
```

### Core data flow (the round trip every milestone keeps working)

```
[app code]  delay_task(name, **kw)
   │  contains {crunchTASK:name, crunchID:uuid} + kw        [M0 wire contract,
   │                                                        renamed in our DSL]
   ▼
[TaskPublisher] --(message)--> [BaseTransport]
   │
   ▼
[TaskDaemon] fetch_next_task() → registry[name] → pool.apply_async(...)
   │  ack() / reject on unknown
   ▼
[ProcessQueue] ordered completion → result
```

---

## 5. Milestone Roadmap

Bound to real celery commit milestones for grounding; all names are our own.

| Ms | Theme | Real anchor (for study) | Parsley deliverable | Verify-gate |
|----|-------|------------------------|---------------------|-------------|
| **M0** | **Seed** | `crunchd` (`8dd7ac781`) | registry + publish/fetch + in-memory transport + `parsleyd` | unit tests: register→delay→run→result round-trip, no broker |
| **M1** | **Task classes** | `462d47068` | `Task` class + `tasks.register()`, `Task.delay()` shortcut | test: define `@task`-equivalent, delay, eager-run |
| **M2** | **Results** | `da7a3f4ee`+ | `delay_task` returns id; result store + retrieve | test: poll result, check executed flag |
| **M3** | **Periodic** | `88a5aede4` | beat-style periodic dispatch | test: schedule fires N times |
| **M4** | **ceremonial rename** | `71face9ab` | (n/a, we are already "parsley") | — |
| **M5** | **Worker daemon** | `742f9bad6`+ | `parsleyd` daemonize, pidfile, real process pool | test: daemon runs, processes queue |
| **M6+** | **Growth** | later celery | broker abstraction (redis→amqp), prefork/greenlet pools, bootsteps, canvas/chords, backends | per-subsystem tests + graph cycle-check |

---

## 6. Interface Contracts (per milestone, verified)

Written before implementation; the implementation must satisfy them or the
milestone fails its gate. **M0 contracts follow here** (shortest complete set
to prove the round trip):

### M0 — `TaskRegistry` (`registry.py`)
- `TaskRegistry` holds `name -> callable`.
- `register(name, func)`: raises `AlreadyRegistered` if `name` present.
- `unregister(name)`: raises `NotRegistered` if absent; else removes.
- `get_task(name)`, `get_all()`, `NotRegistered`, `AlreadyRegistered`.
- global singleton `tasks = TaskRegistry()`.

### M0 — `delay_task(name, **kwargs)` (`task.py`)
- Returns a fresh **task id (string uuid)**.
- **Must raise `NotRegistered` if `name` not in the registry** (this closes the
  gap that real celery left commented-out at M0 and hardened in `fad3646d5`).
- Publishes a message carrying the task name + id + kwargs.

### M0 — transport (`messaging.py` / `transports/in_memory.py`)
- `BaseTransport` interface: `publish(msg)`, `fetch() -> msg|None`, `ack(id)`, `reject(id)`.
- In-memory transport supports concurrent producer/consumer within one process
  (thread/proc safe enough for tests).
- Wire payload: JSON object with `name`, `id`, unrolled kw params.

### M0 — `TaskDaemon` (`worker.py`)
- `concurrency` worker processes (via `multiprocessing.Pool`).
- `fetch_next_task()`: pull message, look up task, `UnknownTask`→reject/requeue,
  else `pool.apply_async`.
- `run()`: loop; on empty queue sleep `queue_wakeup_after`, throttle empty-logs.

### M0 — `parsleyd` CLI (`bin/parsleyd`)
- Flags (optparse): `-c/--concurrency`, `-f/--logfile`, `-l/--loglevel`,
  `-p/--pidfile`, `-w/--wakeup-after`, `-d/--daemon`.
- `main()`: autodiscover → `TaskDaemon(...).run()`.

---

## 7. The Verify-Gate (how a milestone is proven)

A milestone is **done** only when all pass:

1. **Contracts hold** — every interface in section 6 is exercised by a test.
2. **Round-trip passes** — a test that registers → delays → runs → retrieves a
   result without a broker.
3. **No new import cycles** — confirmed via the knowledge-graph query
   (`analyze_code_dependencies` / graph cycle check) on the milestone's files.
4. **Deviation ledger updated** — any divergence logged in section 3.
5. **Doc + mermaid updated** — subsystem map & diagram reflect reality.

---

## 8. Conventions

- **Token-light searching**: resolve via the local knowledge graph first;
  ship only the minimal contract/source slice needed. No bulk dumps.
- **One commit per milestone**, message naming the milestone (e.g. `M0: seed`).
- **Name**: package `parsley`. Worker daemon binary `parsleyd`.
- **Django never imported at the core.** 
- Honesty: if we cannot verify a claim, we say so.

---

*Next action: settle the remaining M0 design choice (in-memory transport layout),
then scaffold `parsley/` + first test under the M0 contract.*
