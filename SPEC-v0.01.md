# Parsley — v0.01 Specification

> A distributed task queue. This is the **seed spec** — the smallest honest
> description of what Parsley is, before it grows. Versioned and committed so
> the design never goes lost.

Status: **v0.01 — seed spec (happy-path).** Living document.

---

## 1. What Parsley is

Parsley is a task queue which:

- takes requests from a **Producer**,
- sends them to a **Worker**,
- the Worker, once it receives a task, starts working on it,
- tracks each task's lifecycle in a **TaskStatusRegister**,
- stores finished results in a small store called the **Backend**,
- lets the Producer collect ready results by polling with a **Heartbeat**.

There is **no retry and no failure-prevention mechanism** in this version.

---

## 2. Actors

| Actor | Responsibility |
|-------|----------------|
| **Producer** | Sends tasks to the queue; polls the Backend for ready results. |
| **Queue** | Holds tasks between the Producer and the Worker. |
| **Worker** | Receives a task, executes it, writes the result to the Backend. |
| **Backend** | Small store that holds finished results. |
| **TaskStatusRegister** | Tracks each task's lifecycle state. |
| **Heartbeat** | The Producer's polling timer. |

---

## 3. Task lifecycle (state machine)

```
IN_QUEUE  →  STARTED  →  RESULT_READY
```

1. When the Producer sends a task to the queue, it notifies the
   **TaskStatusRegister** to register the task as **IN_QUEUE**.
2. When the queue sends the task to the Worker, the state changes
   from **IN_QUEUE** to **STARTED**.
3. When the Worker completes the work, it stores the result in the
   **Backend** and changes the state from **STARTED** to **RESULT_READY**.

---

## 4. Result collection (the sweep)

- The Producer keeps polling the Backend for pending tasks at regular
  intervals using a **Heartbeat**.
- When the Heartbeat clicks, the Producer goes to the Backend, collects all
  tasks that are ready at that sweep, **packetizes** them, and consumes them.
- After each sweep, all entries in the Backend are **deleted**, making way for
  the next set of results.

---

## 5. Data flow (the round trip)

```
Producer → Queue → Worker → Backend → (Producer polls) → done
```

---

## 6. M0 decisions (to be confirmed)

These are the open choices in the seed spec. Each is a deliberate decision,
not an oversight. Confirm or change them before building.

| # | Decision | Options | Current lean |
|---|----------|---------|--------------|
| D-1 | **Crashed Worker** — a task stuck in `STARTED` forever? | (a) leave it (documented limitation) / (b) timeout resets to `IN_QUEUE` | (a) leave it |
| D-2 | **Sweep vs. poll race** — if the Producer misses a sweep, results are deleted before it reads them. Who owns sweep timing? | (a) Producer owns it / (b) Backend owns it | (a) Producer owns it |
| D-3 | **Backend vs. TaskStatusRegister** — one store or two? | (a) one store, two views / (b) two stores | (a) one store, two views |
| D-4 | **Backend is single point of truth** — and single point of failure. Acknowledged? | (a) yes, accepted for M0 / (b) plan a fallback | (a) accepted for M0 |
| D-5 | **No retry / no failure prevention** — consequence: a crashed Worker leaves a task stuck in `STARTED`. | (a) accepted / (b) add a timeout | (a) accepted |

---

## 7. Non-goals (this version)

- No retry mechanism.
- No failure-prevention / crash detection.
- No persistence guarantees beyond the current sweep.
- No multi-broker support.

---

## 8. What grows next (the seam)

The seed spec deliberately leaves a seam: **"no retry, no failure prevention."**
The natural next milestone is to close that seam — e.g. a timeout that resets a
stuck `STARTED` task back to `IN_QUEUE`, or a retry counter. That is the
evolution method: each version leaves a gap, and the next version fills it.

---

*Next action: confirm the M0 decisions (section 6), then scaffold the seed
implementation against this spec.*
