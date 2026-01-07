So for the thread scope locking, do we lock it the moment we establish a student intent above a level of confidence, and thereafter all added units can only be added deterministically? I need to undersatnd how a thread "boots up"

Yes — that’s the clean MVP story. A thread “boots up” in two phases:

Bootstrap (unlocked / finding intent)

Locked scope (normal tutoring)

You can do this without adding new fields by treating “threadScopedUnitIds empty” (or missing) as “not bootstrapped yet”.

Thread boot-up: the deterministic sequence
Step 1 — Thread is created (empty scope)

When the student hits “New thread” (or first-ever message):

Create threadId

Store threadFocusUnitId = entryUnitId (or null)

Store threadScopedUnitIds = [] (or [entryUnitId] if you prefer)

At this point, the thread is not locked yet in practice, because you don’t know intent.

Step 2 — First student message arrives → map units

You run embeddings mapping against unit anchors (for the course).

You get:

topMapped.unitId

topMapped.confidence

Step 3 — If confidence is LOW → clarification loop (still unbooted)

If confidence < threshold (e.g. 0.55):

You do not lock scope yet.

Policy clamps to SOCRATIC_QUESTION only

LLM asks a discriminating question (choose between top candidates)

Student replies → repeat mapping

This is the “bootloader” phase: the thread is collecting enough signal to pick a starting unit.

Step 4 — Confidence is HIGH → lock scope (boot complete)

Once confidence ≥ threshold:

Set:

threadFocusUnitId = topMapped.unitId

threadScopedUnitIds = computeInitialScope(topMapped.unitId, dag)
(e.g., target + direct prereqs + small neighbourhood, or prereq closure)

Save thread state.

From now on, the thread is “locked”.

After boot: how units are allowed to appear

Once locked:

The LLM is constrained to policy.scopedUnitIds (which equals threadScopedUnitIds).

Any additional units can only enter scope deterministically (if you allow expansion), e.g.:

add prereq blockers as they are discovered

add direct prereqs of the current focus

(optional later) add unlocked children

But MVP-wise, the simplest is: include enough prereqs at lock time so you rarely need expansion.

What this looks like in practice (1–2 turns)

Turn 1: “I need help with graphs”

mapping confidence maybe low → Socratic clarifier:

“Do you mean straight-line graphs (y=mx+c) or quadratic graphs?”

Turn 2: “Straight lines”

mapping confidence high → lock scope around Linear Functions and prereqs → start tutoring.

Key point

So yes: scope locks the moment you establish intent with sufficient confidence.
Before that, the thread behaves like a short “intent discovery” funnel using Socratic-only turns.
