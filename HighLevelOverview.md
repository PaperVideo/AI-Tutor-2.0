## Condensed Process Flow Scaffolding (high-level, no code)

### 0) Setup assumptions

* **Course DAG** is static (units + prereq edges).
* **Thread** has a *locked scope* (scoped units) and a thread focus.
* **Student snapshot** tracks progress evidence + mastery tiers.
* **Exam questions** are prebuilt (MCQ-driven), with lockouts and revisit via a simple queue.

---

## 1) Student interaction starts a “turn”

A turn begins when the student:

* sends a message (text/voice → transcript/LaTeX), **or**
* clicks a UI button (continue / start new thread / submit MCQ, etc.)

---

## 2) Backend loads state (deterministic)

Backend loads:

* StudentSnapshot (progress + evidence)
* ThreadState (threadScopedUnitIds, threadFocusUnitId)
* Course DAG
* Exam lockout / eligibility state (from existing exam system)

---

## 3) Determine “what this message is about”

* Use **embeddings + Pinecone unit anchors** to map the message to likely units.
* Pick the **primaryTargetUnitId** (student intent) if confidence is high enough.
* If confidence is low: ask a clarifying question or route to “start new thread” (since scope is locked).

---

## 4) Compute the Policy (deterministic)

Policy is computed *before* generating the tutor reply. It decides:

* **prereqBlockingUnitId** (if direct prereqs aren’t satisfied)
* **focusUnitId** (blocker wins, else the student’s target)
* **scopedUnitIds** (thread-locked fence)
* **allowedActions** for this turn (socratic / concept / drill / exam)
* exam availability + tier choice (tier fallback if no inventory)

---

## 5) Retrieve content (bounded by policy)

Backend retrieves only what’s in scope:

* unit lesson content (transcripts + PDF frameworks)
* candidate exam questions (metadata + IDs) for allowed tier, excluding locked ones

This retrieved content is what anchors the tutor so it doesn’t drift.

---

## 6) Generate the tutor response (LLM “composer” call)

* LLM receives: **Policy + Retrieved Context + Minimal SnapshotLite + Student input**
* LLM outputs a structured response: either

  * Socratic question, or
  * Concept / worked example, or
  * Drill question, or
  * Exam block suggestion (ID only, from candidates)

---

## 7) Validate + enforce (deterministic, no retries)

Backend checks the LLM output against Policy.

* If valid → proceed.
* If invalid → return a deterministic fallback card (no “try again” with LLM).

---

## 8) Update state (deterministic) + respond

Backend persists:

* snapshot evidence updates (drill counters, mastery tier progress, etc.)
* exam queue rotation + lockout timers (via existing exam logic)
* the thread messages and cards returned to the UI

UI renders the new card + available buttons.

---

# Special note: when do we use 2 LLM calls?

Only on **drill-answer turns**, because drill grading isn’t deterministic:

* **Call 1 (Analyzer):** judge the student’s drill answer (correct/incorrect/partial) → evidence signal
* Policy + retrieval happens (or is recomputed) using that evidence
* **Call 2 (Composer):** generate the next tutoring action

For exam MCQs, grading is deterministic → **no analyzer call**.

---

## Mental checklist to sanity-check your understanding

* [ ] Thread scope is locked; topic switching happens by starting a new thread
* [ ] Policy is computed before the tutor LLM response
* [ ] LLM proposes; backend validates and enforces
* [ ] Pinecone is used for mapping + content anchoring, not for “thinking”
* [ ] Exams are prebuilt, MCQ graded, lockouts handled deterministically
* [ ] Two-call is only for drill grading turns
