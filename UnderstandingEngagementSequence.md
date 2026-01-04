My original intent was **not** “two analyzers for everything.” It was a clean separation of responsibilities so you can reason about the loop like math:

* **Policy is deterministic and computed *before* you ask the tutor model to respond.**
* **Snapshot updates are deterministic and based on evidence.**
* The LLM is used mainly to **compose the next tutoring move** inside constraints.

The “two-call” idea only appears when you ask: *“Where does the evidence come from if it isn’t in Pinecone and isn’t deterministic?”* Drill grading is the main example.

Here’s the original loop intent, step-by-step:

---

## The intended turn loop (baseline)

### 1) Student input arrives

* text/voice transcript/latex
* plus any **clientEvent** (button clicks, MCQ result, “continue”, etc.)

### 2) Backend derives what it can deterministically

* If it’s an **exam MCQ**: pass/fail + lockout timing comes deterministically from your exam system.
* If it’s a UI action: intent is explicit (REQUEST_DRILL, CONTINUE, etc.)
* If it’s free text: map to units via **embeddings** (Pinecone unit anchors).

This produces a deterministic **TurnFacts** object:

* `mappedUnits[]` (from embeddings)
* `primaryTargetUnitId`
* `examResult?` (if applicable)
* `eventIntent?`

### 3) Backend updates snapshot using *evidence*

* For exams: use deterministic results (MCQ + lockout)
* For drills: you need a correctness signal (more on this below)

Snapshot update is a backend responsibility, not the LLM’s.

### 4) Backend computes **Policy**

Using:

* DAG + snapshot mastery
* thread scope lock (if enabled)
* exam availability (queue + lockouts)
* “evidence so far” to decide if we should drill more or move to exam

Policy answers: *what is allowed now*.

### 5) Backend retrieves bounded context

Pinecone retrieval based on `policy.scopedUnitIds`, `policy.focusUnitId`, chosen tier.

### 6) One LLM call: **Tutor Composer**

Given `POLICY + CONTEXT + student message`, LLM returns **LlmTutorResponse** (structured JSON).
Backend validates; if invalid → deterministic fallback (no retries).

---

## Where the “analysis call” comes in (only if needed)

The only place this loop *needs* an extra LLM “analysis” call is when a key evidence signal is **not deterministic**.

That’s basically:

* **Drill grading** (student types an answer to a generated drill)

Because drill questions are generated and not stored, the backend cannot reliably grade every math input deterministically (unless you build a CAS-like grader, which you don’t want right now).

So my intent was:

* **Default:** 1 LLM call per turn (Tutor Composer)
* **Exception:** add a small analysis/grading step **only when the turn requires non-deterministic evidence** (drill answer turns)

That’s it.

---

## In your mental model (“policy + llm call -> output -> student input -> llm analysis -> policy update”)

You can rewrite it as:

1. Student input
2. **(Optional) grading/analysis step** if the input is an answer that must update evidence
3. Snapshot update (backend)
4. Policy compute (backend)
5. Tutor response (LLM)

So the policy isn’t updated by the LLM. The LLM only provides either:

* tutoring output (always), and/or
* grading output (only on drill-answer turns)

---

## Why this separation matters

If you let the tutor call both:

* decide what to do next, and
* decide what evidence to record

…you get hidden coupling and harder debugging (“why did it mark mastery?” “why did it switch units?”).

So the intent was:

* **LLM proposes**
* **Backend decides and persists**

---

