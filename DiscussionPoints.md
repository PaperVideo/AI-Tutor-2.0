***Additional Points for Discussion ***

---

## 1) Streamline exam lockout: queue rotation + tier fallback

This is much simpler and works well.

### Minimal exam selection logic

Per unit + tier:

* Maintain a **queue** (or “recently seen list”) for `questionId`s.
* When a question is attempted OR support viewed → it becomes locked for 24h → push it to **back**.
* When selecting next:

  * scan queue from front, pick first **eligible** (not locked)
  * if none eligible → `examAvailability="locked"` + `nextEligibleAt=min(lockouts)`

### When queue exhausted / no questions

If there are **no questions for tier**:

* auto-fallback: bronze → silver → gold (or whichever direction you want)
* policy sets `desiredExamTier` to the next tier that has inventory

So policy computation becomes:

1. try tier demanded by current mastery goal
2. if none exist → move to next tier that exists
3. if still none → `examAvailability="none"` and remove EXAM action

This removes the “complex revisit scoring”.

---

## 2) Determining confidence: model-native or LLM-generated?

You don’t get a reliable “under the hood confidence” from the LLM that you can treat as truth.

So you have 2 practical options:

### Option A (recommended): derive confidence from retrieval scores

Use your mapping retrieval step (Pinecone query against unit anchors):

* take top-k unit matches with similarity scores
* normalize into `[0..1]` confidence values
* pass those to policy + LLM as “mapped units”

This is deterministic and stable.

### Option B: ask the LLM to provide confidence

You *can* ask, but it’s subjective and can drift. Good as a secondary signal, not primary.

Best: **Confidence comes from embeddings retrieval**, not LLM self-reporting.

---

## 3) Unit “comprehensive concept coverage” + drill count not static

Agree that a fixed drill count is too rigid.

What you really want is **evidence-based stopping**, not “N drills”.

Simple approach:

* Each unit has **concept tags** (from your unit content: transcripts/PDF frameworks)
* Track per unit:

  * `drill.attempts`, `drill.correct`, `streakCorrect`
  * optionally per-tag counters (lightweight)

### Progress rule (simple)

* Continue drills until **either**:

  * `streakCorrect >= 2` *and* `correct/attempts >= 0.75` for the last few attempts
  * or the tutor decides to move to exam because policy says `examReady=true`

### “Comprehensive coverage” without overengineering

Don’t try to force the bot to enumerate every concept.
Instead:

* when generating drills, instruct it to **rotate** across the top 3 concept tags in scope
* backend can pass `unitTopConceptTags[]` in context
* stop based on evidence thresholds above

This gives breadth without complex concept graphs.

---

## 4) PF-3.7 deterministic override: what does it look like?

It’s a **non-LLM response builder** that returns a safe card/message when LLM output violates policy.

Examples:

### If LLM chooses `target_unit_id` outside scope

Return deterministic card:

* Text: “That’s outside this thread’s scope. Start a new thread to switch topics.”
* Buttons: Start new thread / Stay here

### If LLM tries EXAM but `examAvailability="locked"`

Return deterministic card:

* Text: “That exam question is locked for 24h. Let’s do a quick drill instead.”
* Card: DRILL (backend requests drill from LLM or uses a canned drill prompt)
* Or: if you want zero extra calls, return a “concept + 1 socratic question” template.

**Key point:** PF-3.7 is not “regenerate”. It’s “reject and replace with deterministic fallback”.

---

## 5) SnapshotLite: for LLM vs for frontend?

Yes—two different “lite” projections. Name them differently to avoid confusion:

### `SnapshotLiteForLLM`

* Minimal, safe, token-cheap state:

  * focus unit, mastery tier, in-progress list, exam lockout summary, recent errors
* Purpose: help LLM choose tone/action without giving it the whole history.

### `SnapshotLiteForUI` (or `ProgressViewModel`)

* What the UI needs to render:

  * progress board, mastery badges, locked exam timers, units in progress
* Purpose: frontend display.

They can share fields, but they’re different consumers, so keep separate types.

---

If you want, I can rewrite PF-3.3–PF-3.7 as a single crisp sequence that uses:

* embeddings-only mapping (no extra LLM call)
* queue-based exam selection with tier fallback
* evidence-based drill continuation
* deterministic fallback cards (no retries)
