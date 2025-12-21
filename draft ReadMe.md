Absolutely — below is the **same PF (Process Flow) structure**, but now it contains **all core functions, prompts, scoring, and validation logic** all the way through (including the “end” parts like revisit queue, UI timers, logging). I’ve also added your new requirement:

✅ **Exam questions have 3 difficulty levels** (Bronze/Silver/Gold), and each unit has **Bronze/Silver/Gold mastery** based on passing those levels.

I’m writing this as **Jira-ready guidance**: your dev should be able to build directly from it with minimal interpretation.

---

# Process Flow Overview (PF)

**PF-0** Shared contracts + schemas
**PF-1** Snapshot & DAG (cold start, sparse state, prerequisites)
**PF-2** Pinecone indexing (Unit Anchors + Exam Candidates with difficulty)
**PF-3** Turn Orchestrator (LLM analysis → state → policy → response)
**PF-4** Drill workflow (generated, not stored as entities; LLM grades)
**PF-5** Exam workflow (MCQ pass/fail + support lock + 24h lock + revisit + tier mastery)
**PF-6** Frontend workspace (cards, timers, actions)
**PF-7** Observability & tuning (logs, debug panel)

---

# EPIC A — Contracts & Canonical State

## PF-0.1 — Types and Schemas (single source of truth)

### PV-AI-001 — Shared types + Zod schemas for all objects

**Deliverables**

* `TutorPolicy`
* `StudentSnapshot` (sparse)
* `TutorMessage` + `TutorCard`
* `LlmTutorResponse` including `turn_analysis`
* `ExamQuestionState` including lockout & revisit
* `UnitMasteryTier` = none/bronze/silver/gold

### ✅ JSON Contracts

#### Unit mastery tier

```json
"masteryTier": "none | bronze | silver | gold"
```

#### Exam question metadata (indexed + DB)

```json
{
  "questionId": "EX-2019-ALG-14",
  "unitIds": ["ALG-01"],
  "difficultyTier": "bronze | silver | gold",
  "tags": ["linear_equations"],
  "year": 2019,
  "courseId": "MATH-G10"
}
```

#### Exam question state (tracked per student)

```json
{
  "questionId": "EX-2019-ALG-14",
  "unitId": "ALG-01",
  "status": "unseen | available | locked | passed",
  "lockedUntil": "2025-12-17T05:27:00+08:00",
  "lockReason": "wrong_attempt | support_viewed",
  "attemptCount": 1,
  "supportViewed": { "memo": true, "video": false },
  "needsRevisit": true,
  "revisitAfter": "2025-12-17T05:27:00+08:00",
  "passedAt": null
}
```

#### Unit progress (sparse)

```json
{
  "status": "not_started | in_progress | mastered",
  "masteryTier": "none | bronze | silver | gold",
  "lastTouchedAt": "ISO",
  "drill": { "attempts": 3, "correct": 2, "streakCorrect": 0 },
  "exam": {
    "passedByTier": { "bronze": 1, "silver": 0, "gold": 0 },
    "lastPassedAt": "ISO"
  },
  "confusionTags": { "inverse_operations": 2 }
}
```

#### Snapshot (canonical)

```json
{
  "studentId": "S1",
  "courseId": "MATH-G10",
  "focusUnitId": "ALG-01",
  "unitsInProgress": ["ALG-01","ALG-00"],
  "unitProgress": { "ALG-01": { "...": "..." } },
  "examQuestionState": { "EX-2019-ALG-14": { "...": "..." } },
  "lastTurnAnalysis": {
    "mappedUnits": [{"unitId":"ALG-01","confidence":0.92}],
    "studentIntent": "solve",
    "understandingSignal": "uncertain",
    "suggestedPrereqUnits": ["ALG-00"],
    "confusionTags": ["inverse_operations"]
  },
  "lastTurnAt": "ISO"
}
```

#### Policy (computed deterministically every turn)

```json
{
  "turnId": "uuid",
  "focusUnitId": "ALG-01",
  "prereqBlockingUnitId": "ALG-00",
  "scopedUnitIds": ["ALG-01","ALG-00"],
  "allowedActions": ["SOCRATIC_QUESTION","DRILL_CARD"],
  "stuck": false,
  "examReady": false,
  "desiredExamTier": "bronze",
  "examAvailability": "available | locked | none",
  "constraints": { "maxConceptWords": 170, "maxWorkedExamples": 1, "drillMaxSteps": 2 }
}
```

---

# EPIC B — Snapshot + DAG

## PF-1.1 — Snapshot storage + cold start defaults

### PV-AI-002 — getOrCreateSnapshot() + sparse unit progress

**Function**

```ts
export async function getOrCreateSnapshot(userId: string, courseId: string): Promise<StudentSnapshot>
```

**Cold start rules**

* `focusUnitId = ENTRY-00` (course-defined)
* no `unitProgress` entries yet
* policy initially allows only Socratic + Drill
* no exam until mapping + readiness

**Implementation snippet**

```ts
const snapshot: StudentSnapshot = {
  studentId: userId,
  courseId,
  focusUnitId: 'ENTRY-00',
  unitsInProgress: [],
  unitProgress: {},
  examQuestionState: {},
  lastTurnAnalysis: null,
  lastTurnAt: new Date().toISOString(),
};
```

---

## PF-1.2 — DAG graph + prerequisite utilities

### PV-AI-003 — LearningPathGraph + validation

**Required deterministic functions**

```ts
getPrereqs(unitId: string): string[]
isBlocked(unitId: string, snapshot: StudentSnapshot): boolean
assertAcyclic(): void
```

---

# EPIC C — Pinecone Guardrails

## PF-2.1 — Index unit anchors

### PV-AI-004 — Index `units` namespace

**Function**

```ts
indexUnitsToPinecone(units: UnitMeta[]): Promise<void>
```

---

## PF-2.2 — Index exam candidates with Bronze/Silver/Gold

### PV-AI-005 — Index `exam_questions` namespace (difficultyTier included)

**Function**

```ts
indexExamQuestionsToPinecone(questions: ExamMeta[]): Promise<void>
```

**Metadata must include**

* `difficultyTier: bronze|silver|gold`
* `unitIds[]`, `courseId`

---

# EPIC D — The Tutor Brain (Turn orchestration)

## PF-3.1 — Turn orchestrator endpoint

### PV-AI-006 — POST /threads/:id/turn (single entry point)

**Deterministic process sequence (fixed order)**

1. Load snapshot + DAG + unit registry
2. Compute **candidate unit set** (from snapshot)
3. Compute **initial policy** (pre-LLM mapping fallback)
4. Pinecone retrieval constrained by scoped units
5. Call LLM with policy + context
6. Parse JSON + extract `turn_analysis`
7. Write `turn_analysis` into snapshot soft state
8. Recompute policy deterministically (now informed by mapping)
9. Validate LLM action against policy; fallback if invalid
10. Persist messages; update touched units; return snapshotLite

---

## PF-3.2 — The “missing link” object: `turn_analysis`

### PV-AI-007 — LLM output schema includes `turn_analysis` always

**LLM must always output**

* mapped units + confidence
* student intent
* understanding signal
* suggested prereqs (optional)
* confusion tags (optional)

**Required fields (minimum)**

```json
"turn_analysis": {
  "student_intent": "solve|explain|check|stuck|unknown",
  "understanding_signal": "confident|uncertain|confused"
}
```

---

## PF-3.3 — Prompt builder (hard guardrails)

### PV-AI-008 — buildTutorPrompt() + callTutorLLM()

### Prompt: System (core rules)

```text
You are Paper Video Tutor. Behave like a private tutor.
You MUST obey the POLICY object.
You MUST output JSON only.

Rules:
- Default mode is Socratic: ask ONE question when clarification is needed.
- DRILL_CARD is for short assessment/practice. Make it 1–2 steps max.
- CONCEPT_CARD only if policy.stuck=true. Keep under policy max words.
- EXAM_BLOCK only if policy.examReady=true AND policy allows it.
- Exam questions are NOT generated. Choose from provided examCandidates only.
- Drills are generated and ephemeral. Do not assume drills are stored.
- Stay anchored to provided unitAnchors. Do not introduce unrelated syllabus topics.
- Never claim mastery. Mastery is set by backend only.
```

### Prompt: POLICY and CONTEXT injection (JSON)

* Inject `POLICY: { ... }`
* Inject `CONTEXT: { unitAnchors[], examCandidates[], snapshotLiteForLLM }`

### LLM output JSON skeleton (required)

```json
{
  "mapped_units": [{"unit_id":"ALG-01","confidence":0.92}],
  "action": "SOCRATIC_QUESTION|CONCEPT_CARD|DRILL_CARD|EXAM_BLOCK",
  "target_unit_id": "ALG-01",
  "tutor_text": "...",
  "drill_card": {"prompt":"...","question_latex":"..."},
  "concept_card": {"key_ideas":["..."], "worked_example": {...}},
  "exam_suggestion": {"question_id":"EX-..."},
  "turn_analysis": {
    "student_intent":"solve",
    "understanding_signal":"uncertain",
    "suggested_prereq_units":["ALG-00"],
    "confusion_tags":["inverse_operations"]
  }
}
```

---

## PF-3.4 — Candidate units + scoring + focus selection

### PV-AI-009 — computeCandidateUnits() + scoring + chooseNextFocusUnit()

#### Candidate set builder (deterministic)

```ts
export function computeCandidateUnits(snapshot: StudentSnapshot, mappedUnits: string[], dag: LearningPathGraph): string[]
```

Candidate union:

* mapped units (1–3)
* current focus
* unitsInProgress (≤3)
* prereqs (1 hop)
* optionally DAG neighbors (1 hop)

#### Scoring structure (relative ordering)

```ts
export function computeUnitPriorityScore(inputs: {
  isMapped: boolean;
  isFocus: boolean;
  isInProgress: boolean;
  prereqMissingCount: number;
  prereqMasteredRatio: number;
  recencyHours: number;
  evidence: { drillStreak: number; examPassedTier: "none|bronze|silver|gold" };
  confusionPenalty: number;
}): number
```

**Bronze/Silver/Gold affects scoring**

* If unit already Gold → score down (unless student insists)
* If unit is close to next tier → score up (finishing effect)
* If there’s a revisit due soon → score up (urgency)

---

## PF-3.5 — Deterministic policy recomputation (the “math proof step”)

### PV-AI-010 — computePolicy() fully specified

**Function**

```ts
export function computePolicy(args: {
  snapshot: StudentSnapshot;
  dag: LearningPathGraph;
  mappedUnits: Array<{unitId:string; confidence:number}>;
  nowIso: string;
}): TutorPolicy
```

**Policy reasoning sequence**

1. Select primary target:

   * If mappedUnits[0].confidence ≥ `CONF_OK` → primary target = mapped unit
   * Else primary target = snapshot.focusUnitId
2. Determine prereq blockers:

   * missingPrereqs = prereqs(target) not mastered
   * prereqBlockingUnitId = deterministic choose (see below)
3. focusUnitId:

   * if prereqBlockingUnitId exists → focus = prereqBlockingUnitId
   * else focus = target
4. stuck:

   * stuck = (attemptsDrill ≥ 2 AND streakCorrect == 0) on focus unit
5. desiredExamTier:

   * based on unit’s current `masteryTier`:

     * none → bronze
     * bronze → silver
     * silver → gold
     * gold → gold (or null)
6. examReady:

   * prereqs must be mastered AND drillStreak ≥ threshold (e.g. 2)
7. examAvailability (tier-aware):

   * compute whether there exists an **available** exam question at `desiredExamTier`
   * else if exists locked question needing revisit at that tier → locked
   * else none
8. allowedActions:

   * always allow Socratic and Drill
   * allow Concept only if stuck
   * allow Exam only if examReady AND availability != none
9. scopedUnitIds:

   * {focus, primary target, missing prereqs, unitsInProgress} capped

#### Deterministic prereq selection

```ts
choosePrereqToNudge(missingPrereqs): string
```

Strategy v1:

* choose the prereq with lowest masteryTier / lowest drill streak / oldest touched (simple tie-break)

---

## PF-3.6 — Validation + fallback (no hoping)

### PV-AI-011 — validateAgainstPolicy() + fallbackResponse()

**Rules**

* action must be allowed
* target_unit_id must be within scope
* exam_suggestion.question_id must be from candidates AND correct tier AND available/not locked
* concept card only if stuck
* drill card must be 1–2 steps (heuristic check; see below)

**Drill “step limit” heuristic**

* ask LLM to generate *simple* drills; backend can do a crude check:

  * reject if the LaTeX contains multiple “=” lines, or too many operators, or multi-part questions
  * if rejected, fallback to Socratic question

---

# EPIC E — Drill Workflow (generated, not stored as entities)

## PF-4.1 — Drill generation: in chat only, not stored as assets

### PV-AI-012 — Drill Cards are ephemeral; snapshot stores only evidence counters

**Clarification encoded in tickets**

* no `drills` table
* no drill library
* chat message may contain drill text (optional)
* grading request must carry drill text if you prefer zero persistence

---

## PF-4.2 — Drill grading endpoint (LLM grades)

### PV-AI-013 — POST /threads/:id/drill/grade

**Request**

```json
{
  "courseId": "MATH-G10",
  "unitId": "ALG-01",
  "drill": { "question_latex": "2x+3=11", "prompt": "Solve for x" },
  "studentAnswer": "x=4"
}
```

**Prompt for grading (short, deterministic style)**

```text
Grade this drill question.
Return JSON only: { "isCorrect": boolean, "feedbackText": string, "commonMistakeTag": string|null }.
The drill is intended to be simple and 1–2 steps.
Be strict but helpful.
```

**Snapshot update**

* attempts++, correct++, streakCorrect update
* update confusion tag counters if commonMistakeTag returned

---

## PF-4.3 — Stuck detection unlocks Concept Card

### PV-AI-014 — stuckFromEvidence() deterministic

```ts
stuck = attemptsDrill >= 2 && streakCorrect == 0
```

(Optionally: if LLM says “confused” you can lower attempts threshold, but evidence remains the anchor.)

---

# EPIC F — Exam Workflow (MCQ + lockout + revisit + Bronze/Silver/Gold)

## PF-5.1 — Exam question state machine (tier-aware)

### PV-AI-015 — ExamQuestionState transitions

**State transitions**

* unseen → available (when shown)
* available → passed (MCQ correct)
* available → locked (MCQ wrong OR support viewed)
* locked → available (once now ≥ lockedUntil, computed at read time)

---

## PF-5.2 — MCQ submit endpoint

### PV-AI-016 — POST /threads/:id/exam/mcq-submit

**Request**

```json
{
  "courseId":"MATH-G10",
  "unitId":"ALG-01",
  "questionId":"EX-2019-ALG-14",
  "difficultyTier":"bronze",
  "isCorrect": true
}
```

**Deterministic updates**

* if correct:

  * question status = passed
  * `unitProgress[unitId].exam.passedByTier[bronze]++`
  * update unit mastery tier (see PF-5.5)
* if incorrect:

  * status = locked
  * lockedUntil = now+24h
  * needsRevisit=true

---

## PF-5.3 — Support viewed endpoint (memo/video locks too)

### PV-AI-017 — POST /threads/:id/exam/support-viewed

**Request**

```json
{
  "courseId":"MATH-G10",
  "unitId":"ALG-01",
  "questionId":"EX-2019-ALG-14",
  "supportType":"video"
}
```

**Update**

* if not passed:

  * lock until max(existingLockedUntil, now+24h)
  * set needsRevisit=true
  * record supportViewed.video=true

---

## PF-5.4 — Tier-aware availability + revisit queue

### PV-AI-018 — computeExamAvailabilityForUnit()

**Function**

```ts
export function computeExamAvailabilityForUnit(args: {
  snapshot: StudentSnapshot;
  unitId: string;
  desiredTier: "bronze"|"silver"|"gold";
  nowIso: string;
  examCatalogForUnit: ExamMeta[]; // from DB or cached
}): {
  availability: "available"|"locked"|"none";
  candidateQuestionIds: string[];
  nextEligibleAt?: string;
  revisitQuestionIds: string[];
}
```

**Deterministic logic**

* Filter examCatalogForUnit by desiredTier
* Exclude passed questions
* Normalize locked→available if now >= lockedUntil
* If any available: availability=available, candidateQuestionIds=list
* Else if any locked with needsRevisit: availability=locked, nextEligibleAt=min(revisitAfter)
* Else none

This is what makes the tutor “remember” locked questions and drive revisit.

---

## PF-5.5 — Bronze/Silver/Gold mastery commit per unit (deterministic)

### PV-AI-019 — updateUnitMasteryTier()

**Function**

```ts
export function updateUnitMasteryTier(unitProgress: UnitProgress): "none"|"bronze"|"silver"|"gold"
```

**Suggested deterministic rule (v1)**

* Bronze mastery if:

  * drillStreak ≥ 2 AND passedByTier.bronze ≥ 1
* Silver mastery if:

  * bronze mastery AND passedByTier.silver ≥ 1
* Gold mastery if:

  * silver mastery AND passedByTier.gold ≥ 1

You can raise counts later (e.g., require 2 bronzes, etc.) without changing architecture.

**Policy uses mastery tier**

* desiredExamTier = next tier to unlock
* if already gold, policy may:

  * continue exam at gold for reinforcement, or
  * allow “review mode” but keep it optional

---

## PF-5.6 — LLM exam suggestion constrained by tier availability

### PV-AI-020 — Validate exam suggestion strictly

When LLM returns:

```json
"exam_suggestion": { "question_id": "EX-..." }
```

Backend checks:

* question_id ∈ `candidateQuestionIds` from computeExamAvailabilityForUnit()
* tier matches desiredExamTier
* not locked (unless explicitly allowing “locked card” display state)

Fallback:

* if locked: show locked revisit info + drill instead
* if none: drill instead

---

# EPIC G — Frontend (Unified Workspace)

## PF-6.1 — Tutor workspace page

### PV-AI-021 — Angular page: threads + chat + context strip

**SnapshotLite (expanded)**

```json
{
  "focus": { "unitId":"ALG-01", "title":"Linear equations", "masteryTier":"bronze" },
  "prereqNudge": { "unitId":"ALG-00", "title":"Inverse operations" },
  "revisit": { "lockedCount": 2, "nextQuestionId":"EX-2019-ALG-14", "nextEligibleAt":"ISO" },
  "progress": { "bronzeUnits": 12, "silverUnits": 4, "goldUnits": 1, "totalUnits": 40 }
}
```

---

## PF-6.2 — Cards and triggers (full function-level behavior)

### PV-AI-022 — Drill card component (answer → /drill/grade)

* Sends drill content back (since drills aren’t stored)
* Renders feedback text returned
* Updates context strip

### PV-AI-023 — Exam block component (MCQ + support lock + countdown)

* On MCQ submit → `/exam/mcq-submit`
* On memo/video click → `/exam/support-viewed`
* If locked: show countdown and “Revisit later” badge
* Bonus: show “Add to revisit list” (local UI list) even without background jobs

---

# EPIC H — Observability and “No Stuff Ups”

## PF-7.1 — Full reasoning chain logs

### PV-AI-024 — logTurn() records every key variable

Log:

* mapped units + confidence
* selected focusUnitId
* prereqBlockingUnitId
* allowedActions
* desiredExamTier + availability
* candidateQuestionIds count
* validation outcome + fallback reason

This makes debugging deterministic.

---

## PF-7.2 — Debug panel (dev-only)

### PV-AI-025 — Render policy + snapshotLite + exam state summary

Displays:

* policy JSON
* top candidates + scores
* revisit queue
* mastery tier progression per unit

---

# Exhaustive Input → Response Triggers (expanded with tier mastery)

## Input Class 1 — `/turn` message arrives

### 1A: Confidence high mapping; prereqs satisfied; not stuck

* allowed: Socratic, Drill
* if drill streak enough: examReady true
* desiredExamTier = next tier
* if exam available → allow exam

**Typical response**: Drill → then Exam at desired tier

### 1B: Confidence high; prereqs missing

* focus becomes prereqBlockingUnit
* exam disallowed for target
  **Response**: Drill on prereq (or Socratic)

### 1C: Confidence low/ambiguous

**Response**: 1 Socratic clarifier (no card)

### 1D: Student is “stuck” by evidence

* stuck=true → concept allowed
  **Response**: Concept Card (short) → Drill

---

## Input Class 2 — `/drill/grade`

### 2A: correct

* streak increments → may enable examReady
  **Next**: `/turn` often suggests exam

### 2B: incorrect twice

* stuck=true next policy
  **Next**: concept allowed

---

## Input Class 3 — `/exam/mcq-submit`

### 3A: correct

* increments passedByTier[tier]
* may upgrade masteryTier (bronze→silver→gold)
  **Next**: policy desiredExamTier advances

### 3B: incorrect

* locks question 24h, needsRevisit=true
* examAvailability might become locked
  **Next**: tutor drills or schedules revisit

---

## Input Class 4 — `/exam/support-viewed`

* locks question 24h, needsRevisit=true
  **Next**: drill + revisit queue updated

---

# Key Implementation Clarification (drills not stored)

The architecture is consistent with your constraint:

* **Drills:** generated, graded live, only evidence stored in snapshot
* **Exams:** DB-controlled, MCQ pass/fail, lockouts, revisit tracking, mastery tiers

---

If you want, I can now output this in a **pure Jira paste format** (each ticket as: Summary / Description / AC / Implementation / Subtasks / Code block attachments) so you can paste directly into Jira with minimal editing.
