# AI-Tutor-2.0

Below is a **full upgraded version** of the **Process Flow Overview (PF)** response (same structure, expanded), incorporating:

* **Confusion tags only from drill grading** (no LLM “guess tags”)
* **Drills are fully generated** (no drill entities stored; evidence only)
* **Pinecone context explicitly passed to LLM** (`unitAnchors[]`, `unitContentChunks[]`, `examCandidates[]`, optional `examSupportChunks[]`)
* **Exam lockouts come from your existing API** (authoritative)
* **Unit/exam transcripts + unit PDFs** included in Pinecone indexing plan (for the full update later, we include it now as requested)
* **Deterministic fallback only** (no retries)
* **Bronze/Silver/Gold exam difficulty → Bronze/Silver/Gold mastery tiers per unit**
* **Linear decay + explicit normalization functions for all scoring components**
* **EPIC G beefed up**: UI layout, buttons, and exactly where “Continue”, “Check”, “Try a Drill”, etc. live

Everything below is intended to be **pasteable into Jira** and readable as a working engineering document.

---

# Process Flow Overview (PF)

### One-line summary

Each student interaction produces a **deterministic Policy** from the **persistent Snapshot** and **static DAG**, retrieves bounded context from Pinecone, calls the LLM to produce a structured action, validates it, and updates snapshot using **hard evidence** (drill grading + exam MCQ events + lockout API).

---

# EPIC A — Shared Contracts, Schemas, and Data Shapes

## PF-0.1 — Shared contracts (Types) + Validation (Zod)

### PV-AI-001 — Shared TypeScript types + Zod schemas

**Concept explainer (doc style):**
We standardize all objects that flow between FE ↔ BE ↔ LLM. This reduces misunderstandings and makes the system testable.

### Core JSON Objects (with field glossaries)

#### 1) `TutorPolicy` (computed every turn; not stored)

**Purpose:** Guardrails for the LLM and deterministic control of allowed actions.

```json
{
  "turnId": "uuid",
  "focusUnitId": "ALG-01",
  "primaryTargetUnitId": "ALG-01",
  "prereqBlockingUnitId": "ALG-00",
  "scopedUnitIds": ["ALG-01","ALG-00"],
  "allowedActions": ["SOCRATIC_QUESTION","DRILL_CARD"],
  "stuck": false,
  "examReady": false,
  "desiredExamTier": "bronze",
  "examAvailability": "available",
  "constraints": { "maxConceptWords": 170, "maxWorkedExamples": 1, "drillMaxSteps": 2 }
}
```

**Field glossary**

* `turnId`: unique per assistant turn, for logging/debug.
* `focusUnitId`: the unit we are actively tutoring right now.
* `primaryTargetUnitId`: what the student likely asked about.
* `prereqBlockingUnitId`: missing prereq we must address first (deterministic from DAG).
* `scopedUnitIds`: small list of units the LLM is allowed to reference/target this turn.
* `allowedActions`: the only actions the LLM is permitted to output.
* `stuck`: true only when evidence indicates stuck (drill evidence).
* `examReady`: prereqs satisfied + drill readiness threshold met.
* `desiredExamTier`: next tier to progress (bronze→silver→gold).
* `examAvailability`: whether an exam question at desired tier is currently attemptable (`available|locked|none`).
* `constraints`: small limits used to keep output consistent and non-bloated.

---

#### 2) `StudentSnapshot` (stored; persistent)

**Purpose:** Canonical student progress record across sessions and DAG units (sparse).

```json
{
  "studentId": "S1",
  "courseId": "MATH-G10",
  "focusUnitId": "ALG-01",
  "unitsInProgress": ["ALG-01","ALG-00"],
  "unitProgress": {
    "ALG-01": {
      "status": "in_progress",
      "masteryTier": "bronze",
      "lastTouchedAt": "ISO",
      "drill": { "attempts": 3, "correct": 2, "streakCorrect": 0 },
      "exam": { "passedByTier": { "bronze": 1, "silver": 0, "gold": 0 } },
      "confusionTags": { "inverse_operations": 2 }
    }
  },
  "examTouched": { "EX-2019-ALG-14": true },
  "revisitQueue": { "EX-2019-ALG-14": { "unitId":"ALG-01", "tier":"bronze" } },
  "lastTurnAnalysis": {
    "mappedUnits": [{ "unitId":"ALG-01", "confidence":0.92 }],
    "studentIntent": "solve",
    "understandingSignal": "uncertain",
    "suggestedPrereqUnits": ["ALG-00"]
  },
  "lastTurnAt": "ISO"
}
```

**Field glossary**

* `focusUnitId`: last known focus; updated deterministically each turn.
* `unitsInProgress`: small list of units actively being worked on.
* `unitProgress`: sparse per-unit evidence & mastery.
* `examTouched`: sparse marker so you don’t store state for 15,000 questions; only those interacted with.
* `revisitQueue`: sparse list of questions to revisit (questionId → unit/tier). Lockout timing comes from existing API.
* `lastTurnAnalysis`: LLM-provided mapping + intent metadata (NOT confusion tags).
* `lastTurnAt`: timestamp for recency scoring.

> **ConfusionTags rule (your instruction):** Only updated from drill grading results.

---

#### 3) `TutorMessage` and `TutorCard` (stored as conversation log)

**Purpose:** UI renders timeline items; cards are interactive.

```json
{
  "id": "uuid",
  "threadId": "T1",
  "role": "assistant",
  "text": "Let’s do one quick practice first.",
  "latex": null,
  "card": {
    "type": "DRILL",
    "unitId": "ALG-01",
    "prompt": "Solve for x",
    "questionLatex": "2x+3=11"
  },
  "createdAt": "ISO"
}
```

**Card types**

* `CONCEPT`: short key ideas + optional worked example (LaTeX).
* `DRILL`: generated question (ephemeral), answer input + check button.
* `EXAM`: pre-existing exam block with MCQ + links.

---

#### 4) `LlmTutorResponse` (LLM output, validated)

**Purpose:** Structured assistant decision. Must obey policy.

```json
{
  "mapped_units": [{ "unit_id":"ALG-01", "confidence":0.92 }],
  "action": "DRILL_CARD",
  "target_unit_id": "ALG-01",
  "tutor_text": "What’s the first step to isolate x?",
  "drill_card": { "prompt":"Solve for x", "question_latex":"2x+3=11" },
  "turn_analysis": {
    "student_intent":"solve",
    "understanding_signal":"uncertain",
    "suggested_prereq_units":["ALG-00"]
  }
}
```

**Field glossary**

* `mapped_units`: LLM semantic mapping (unit IDs + confidence).
* `action`: one of the policy-allowed actions.
* `target_unit_id`: must be in `policy.scopedUnitIds`.
* `turn_analysis`: LLM-only metadata for mapping/intent (no confusion tags here).

---

# EPIC B — Snapshot Storage + DAG Graph

## PF-1.1 — Snapshot storage & cold start

### PV-AI-002 — getOrCreateSnapshot() + sparse default

**Concept explainer:**
We need a persistent student record from the first interaction. The “ENTRY-00” focus is just a safe default pointer; mapping will immediately move focus.

**Code**

```ts
export async function getOrCreateSnapshot(userId: string, courseId: string) {
  const existing = await db.snapshots.get(userId, courseId);
  if (existing) return existing;

  const snap = {
    studentId: userId,
    courseId,
    focusUnitId: 'ENTRY-00',
    unitsInProgress: [],
    unitProgress: {},
    examTouched: {},
    revisitQueue: {},
    lastTurnAnalysis: null,
    lastTurnAt: new Date().toISOString(),
  };
  await db.snapshots.upsert(userId, courseId, snap);
  return snap;
}
```

---

## PF-1.2 — DAG graph utilities

### PV-AI-003 — LearningPathGraph (static course graph)

**Concept explainer:**
The DAG is static course content (units and prerequisites). Student progress is separate and sparse.

**Example DAG JSON**

```json
{
  "courseId":"MATH-G10",
  "entryUnitId":"ENTRY-00",
  "prereqEdges": [
    ["ALG-00","ALG-01"],
    ["ALG-01","ALG-02"]
  ]
}
```

**Code**

```ts
export class LearningPathGraph {
  constructor(private prereqs: Map<string, string[]>) {}

  getPrereqs(unitId: string): string[] {
    return this.prereqs.get(unitId) ?? [];
  }

  isBlocked(unitId: string, masteryTierByUnit: Record<string, string>): boolean {
    return this.getPrereqs(unitId).some(p => (masteryTierByUnit[p] ?? 'none') === 'none');
  }
}
```

> Note: we pass `masteryTierByUnit` rather than whole snapshot if you prefer tighter signatures.

---

# EPIC C — Content Guardrails with Pinecone (Unit + Exam + Transcripts/PDF chunks)

## PF-2.1 — Pinecone indices

### PV-AI-004 — Index Unit Anchors + Unit Content Chunks

**Concept explainer:**
We separate **anchors** (short, canonical) from **content chunks** (transcripts, PDFs). Anchors are for navigation; chunks are for teaching/explaining.

**Namespaces**

* `unit_anchors` — unit title, tags, misconceptions, short anchor text
* `unit_content_chunks` — lesson transcripts + PDF framework chunks (chunked by section)

**Chunk metadata**

```json
{
  "courseId":"MATH-G10",
  "unitId":"ALG-01",
  "chunkType":"pdf|transcript",
  "sourceId":"lesson-ALG-01",
  "section":"Solving linear equations",
  "text":"..."
}
```

---

### PV-AI-005 — Index Exam Candidates + Exam Support Chunks (tiered)

**Concept explainer:**
We index exam **candidates** for selection (metadata) and exam **support chunks** for tutoring context (transcripts/memos). The actual exam question shown remains your controlled UI block.

**Namespaces**

* `exam_candidates` — metadata (questionId, unitIds, tier, tags)
* `exam_support_chunks` — memo/video transcript chunks (optional for tutoring context)

**Candidate metadata**

```json
{
  "courseId":"MATH-G10",
  "questionId":"EX-2019-ALG-14",
  "unitIds":["ALG-01"],
  "difficultyTier":"bronze",
  "tags":["linear_equations"]
}
```

---

# EPIC D — The Tutor Brain (Turn Loop: Policy → Retrieval → LLM → Validate → Persist)

## PF-3.1 — Turn Orchestrator endpoint

### PV-AI-006 — `POST /api/tutor/threads/:threadId/turn`

**Concept explainer:**
This is the single entry point for “student says something” → “tutor responds”. It produces a response and updates snapshot.

**Request**

```json
{
  "courseId":"MATH-G10",
  "messageText":"I need help with graphs",
  "clientEvent": { "type":"NONE" }
}
```

**Response**

```json
{
  "turnId":"uuid",
  "messages":[ "...timeline..." ],
  "snapshotLite": { "...UI summary..." }
}
```

---

## PF-3.2 — Scoped units: deterministic selection for this turn

### PV-AI-007 — computeCandidateUnits() + computeScopedUnitIds()

**Concept explainer:**
We first build a broad candidate set (what might matter), then distill it into a small `scopedUnitIds` for safe retrieval and LLM targeting.

**Code: computeCandidateUnits**

```ts
export function computeCandidateUnits(args: {
  snapshotFocusUnitId: string;
  unitsInProgress: string[];
  mappedUnits: Array<{ unitId: string; confidence: number }>;
  dagGetPrereqs: (unitId: string) => string[];
}): string[] {
  const set = new Set<string>();

  // mapped units (top 3)
  args.mappedUnits
    .sort((a,b)=>b.confidence-a.confidence)
    .slice(0,3)
    .forEach(m => set.add(m.unitId));

  // focus
  set.add(args.snapshotFocusUnitId);

  // in progress
  args.unitsInProgress.slice(0,3).forEach(u => set.add(u));

  // prereqs of mapped + focus (1 hop)
  for (const u of Array.from(set)) {
    for (const p of args.dagGetPrereqs(u)) set.add(p);
  }

  return Array.from(set);
}
```

**Code: computeScopedUnitIds (small + ordered cap)**

```ts
export function computeScopedUnitIds(args: {
  focusUnitId: string;
  primaryTargetUnitId: string;
  prereqBlockingUnitId?: string;
  mappedUnits: Array<{ unitId: string; confidence: number }>;
  unitsInProgress: string[];
  dagGetPrereqs: (unitId: string) => string[];
  maxScoped?: number; // default 6
}): string[] {
  const maxScoped = args.maxScoped ?? 6;
  const scoped = new Set<string>();

  // Always include focus + target
  scoped.add(args.focusUnitId);
  scoped.add(args.primaryTargetUnitId);

  // Blocker and its prereqs
  if (args.prereqBlockingUnitId) {
    scoped.add(args.prereqBlockingUnitId);
    args.dagGetPrereqs(args.prereqBlockingUnitId).forEach(p => scoped.add(p));
  }

  // Top mapped (2)
  const topMapped = args.mappedUnits
    .sort((a,b)=>b.confidence-a.confidence)
    .slice(0,2)
    .map(m => m.unitId);
  topMapped.forEach(u => scoped.add(u));

  // Prereqs of target
  args.dagGetPrereqs(args.primaryTargetUnitId).forEach(p => scoped.add(p));

  // A couple in-progress units
  args.unitsInProgress.slice(0,2).forEach(u => scoped.add(u));

  // Deterministic order
  const ordered: string[] = [];
  const pushIf = (u?: string) => { if (u && scoped.has(u) && !ordered.includes(u)) ordered.push(u); };

  pushIf(args.focusUnitId);
  pushIf(args.prereqBlockingUnitId);
  pushIf(args.primaryTargetUnitId);
  topMapped.forEach(pushIf);
  args.dagGetPrereqs(args.primaryTargetUnitId).forEach(pushIf);
  args.unitsInProgress.forEach(pushIf);

  // Fill any remaining
  Array.from(scoped).forEach(pushIf);

  return ordered.slice(0, maxScoped);
}
```

---

## PF-3.3 — Scoring (all values normalized, linear decay, explicit computations)

### PV-AI-008 — computeUnitPriorityScore() + normalization helpers

**Concept explainer:**
Scoring ranks candidate units when multiple are relevant. It does not override prereq gating.

### Normalization functions (simple + explicit)

**Recency (linear decay over 72h)**

```ts
export function recencyValue(hoursSinceTouched: number): number {
  if (!isFinite(hoursSinceTouched)) return 0;
  return Math.max(0, Math.min(1, 1 - (hoursSinceTouched / 72)));
}
```

**Mapped confidence (cap to 0..1, already)**

```ts
export function mappedValue(conf: number): number {
  return Math.max(0, Math.min(1, conf));
}
```

**In-progress boost**

```ts
export function inProgressValue(isInProgress: boolean): number {
  return isInProgress ? 1 : 0;
}
```

**Tier need (what’s next?)**

```ts
export function tierNeedValue(masteryTier: 'none'|'bronze'|'silver'|'gold'): number {
  // higher means more urgent to progress
  if (masteryTier === 'none') return 1.0;
  if (masteryTier === 'bronze') return 0.7;
  if (masteryTier === 'silver') return 0.4;
  return 0.1; // gold
}
```

**Evidence (drill streak mapped to 0..1)**

```ts
export function drillEvidenceValue(streakCorrect: number): number {
  // streak of 0→0, 1→0.5, 2+→1
  if (streakCorrect <= 0) return 0;
  if (streakCorrect === 1) return 0.5;
  return 1.0;
}
```

### Scoring function (weighted sum)

```ts
export function computeUnitPriorityScore(x: {
  mappedConfidence: number;        // 0..1
  isFocus: boolean;
  isInProgress: boolean;
  recencyHours: number;
  masteryTier: 'none'|'bronze'|'silver'|'gold';
  drillStreak: number;
}): number {
  const intent = 0.7 * mappedValue(x.mappedConfidence) + 0.3 * (x.isFocus ? 1 : 0);
  const rec = recencyValue(x.recencyHours);
  const prog = inProgressValue(x.isInProgress);
  const tierNeed = tierNeedValue(x.masteryTier);
  const drillEvd = drillEvidenceValue(x.drillStreak);

  // Weights (tune later)
  const score =
    0.35 * intent +
    0.15 * rec +
    0.15 * prog +
    0.20 * tierNeed +
    0.15 * drillEvd +

  return Math.max(0, Math.min(1, score));
}
```

---

## PF-3.4 — Deterministic policy computation (code-level)

### PV-AI-009 — computePolicy() + helper functions

**Concept explainer:**
Policy is the “what is allowed now” object. It is recomputed every turn from snapshot + DAG + exam lockout states.

### Key helpers

**Stuck detection (evidence only)**

```ts
export function isStuckByEvidence(unitProg: any): boolean {
  const attempts = unitProg?.drill?.attempts ?? 0;
  const streak = unitProg?.drill?.streakCorrect ?? 0;
  return attempts >= 2 && streak === 0;
}
```

**Desired exam tier from mastery tier**

```ts
export function desiredExamTierFromMastery(m: 'none'|'bronze'|'silver'|'gold'): 'bronze'|'silver'|'gold' {
  if (m === 'none') return 'bronze';
  if (m === 'bronze') return 'silver';
  return 'gold';
}
```

**Mastery tier update (deterministic)**

```ts
export function updateUnitMasteryTier(unitProg: any): 'none'|'bronze'|'silver'|'gold' {
  const streak = unitProg?.drill?.streakCorrect ?? 0;
  const passed = unitProg?.exam?.passedByTier ?? { bronze:0, silver:0, gold:0 };

  const bronzeOk = streak >= 2 && passed.bronze >= 1;
  const silverOk = bronzeOk && passed.silver >= 1;
  const goldOk   = silverOk && passed.gold >= 1;

  if (goldOk) return 'gold';
  if (silverOk) return 'silver';
  if (bronzeOk) return 'bronze';
  return 'none';
}
```

### Exam availability (authoritative lockout API)

We only query exam status for **candidate question IDs** (small list), not 15k.

```ts
export type ExamAvail = { availability: 'available'|'locked'|'none'; candidateIds: string[]; nextEligibleAt?: string };

export async function computeExamAvailabilityForTier(args: {
  studentId: string;
  courseId: string;
  unitId: string;
  desiredTier: 'bronze'|'silver'|'gold';
  candidateQuestionIds: string[]; // from Pinecone / DB selection
  examApiBatchGetStatus: (studentId: string, questionIds: string[]) => Promise<Record<string, { status:'available'|'locked'|'passed', lockedUntil?: string }>>;
  nowIso: string;
}): Promise<ExamAvail> {
  if (args.candidateQuestionIds.length === 0) return { availability: 'none', candidateIds: [] };

  const statusMap = await args.examApiBatchGetStatus(args.studentId, args.candidateQuestionIds);

  const available = args.candidateQuestionIds.filter(q => statusMap[q]?.status === 'available');
  if (available.length) return { availability: 'available', candidateIds: available };

  const locked = args.candidateQuestionIds
    .filter(q => statusMap[q]?.status === 'locked')
    .map(q => statusMap[q]?.lockedUntil)
    .filter(Boolean) as string[];

  if (locked.length) {
    const nextEligibleAt = locked.sort()[0];
    return { availability: 'locked', candidateIds: [], nextEligibleAt };
  }

  return { availability: 'none', candidateIds: [] };
}
```

### computePolicy (core)

```ts
export async function computePolicy(args: {
  snapshot: any;
  dag: LearningPathGraph;
  mappedUnits: Array<{ unitId: string; confidence: number }>;
  nowIso: string;
  examCandidateIdsForFocusTier: string[];
  examApiBatchGetStatus: (studentId: string, questionIds: string[]) => Promise<Record<string, { status:'available'|'locked'|'passed', lockedUntil?: string }>>;
}): Promise<any> {
  const { snapshot, dag } = args;

  const topMapped = [...args.mappedUnits].sort((a,b)=>b.confidence-a.confidence)[0];
  const primaryTargetUnitId = (topMapped && topMapped.confidence >= 0.55) ? topMapped.unitId : snapshot.focusUnitId;

  // Prereq blocker (deterministic)
  const prereqs = dag.getPrereqs(primaryTargetUnitId);
  const masteryTierByUnit: Record<string,string> = {};
  for (const [unitId, up] of Object.entries(snapshot.unitProgress ?? {})) masteryTierByUnit[unitId] = (up as any).masteryTier ?? 'none';

  const missing = prereqs.filter(p => (masteryTierByUnit[p] ?? 'none') === 'none');
  const prereqBlockingUnitId = missing[0]; // v1: first missing; can use smarter selection later

  const focusUnitId = prereqBlockingUnitId ?? primaryTargetUnitId;

  const focusProg = snapshot.unitProgress?.[focusUnitId];
  const stuck = isStuckByEvidence(focusProg);

  // Tier logic
  const focusTier: 'none'|'bronze'|'silver'|'gold' = focusProg?.masteryTier ?? 'none';
  const desiredExamTier = desiredExamTierFromMastery(focusTier);

  // Exam ready requires prereqs satisfied + drill readiness
  const prereqsOk = !dag.isBlocked(focusUnitId, masteryTierByUnit);
  const drillStreak = focusProg?.drill?.streakCorrect ?? 0;
  const examReady = prereqsOk && drillStreak >= 2 && !prereqBlockingUnitId;

  // Exam availability from existing API (tier candidates already filtered upstream)
  const examAvail = examReady
    ? await computeExamAvailabilityForTier({
        studentId: snapshot.studentId,
        courseId: snapshot.courseId,
        unitId: focusUnitId,
        desiredTier: desiredExamTier,
        candidateQuestionIds: args.examCandidateIdsForFocusTier,
        examApiBatchGetStatus: args.examApiBatchGetStatus,
        nowIso: args.nowIso,
      })
    : { availability:'none', candidateIds: [] as string[] };

  const allowedActions = ['SOCRATIC_QUESTION','DRILL_CARD'] as string[];
  if (stuck) allowedActions.push('CONCEPT_CARD');
  if (examReady && examAvail.availability === 'available') allowedActions.push('EXAM_BLOCK');

  return {
    turnId: crypto.randomUUID(),
    focusUnitId,
    primaryTargetUnitId,
    prereqBlockingUnitId,
    scopedUnitIds: [], // filled by PF-3.2 computeScopedUnitIds()
    allowedActions,
    stuck,
    examReady,
    desiredExamTier,
    examAvailability: examAvail.availability,
    constraints: { maxConceptWords: 170, maxWorkedExamples: 1, drillMaxSteps: 2 },
    _debug: { examNextEligibleAt: examAvail.nextEligibleAt }
  };
}
```

---

## PF-3.5 — Retrieval and passing context to the LLM (explicit)

### PV-AI-010 — ragRetrieve() + prompt injection of `unitAnchors[]`, `unitContentChunks[]`, `examCandidates[]`

**Concept explainer:**
The LLM does not query Pinecone. The backend retrieves and **injects** results into the prompt.

**Retrieval outputs (conceptual)**

* `unitAnchors[]` (scoped)
* `unitContentChunks[]` (scoped, topK small)
* `examCandidates[]` (focus unit + desired tier)
* optional: `examSupportChunks[]` (only for guidance, not to “reveal answers”)

**Code skeleton**

```ts
export async function ragRetrieve(args: {
  courseId: string;
  userText: string;
  scopedUnitIds: string[];
  focusUnitId: string;
  desiredExamTier: 'bronze'|'silver'|'gold';
}) {
  const queryVec = await embed(args.userText);

  const unitAnchors = await pinecone.query('unit_anchors', {
    topK: 6, vector: queryVec,
    filter: { courseId: args.courseId, unitId: { $in: args.scopedUnitIds } },
    includeMetadata: true,
  });

  const unitContentChunks = await pinecone.query('unit_content_chunks', {
    topK: 8, vector: queryVec,
    filter: { courseId: args.courseId, unitId: { $in: args.scopedUnitIds } },
    includeMetadata: true,
  });

  const examCandidates = await pinecone.query('exam_candidates', {
    topK: 10, vector: queryVec,
    filter: { courseId: args.courseId, unitIds: { $in: [args.focusUnitId] }, difficultyTier: args.desiredExamTier },
    includeMetadata: true,
  });

  return {
    unitAnchors: unitAnchors.matches.map(m => m.metadata),
    unitContentChunks: unitContentChunks.matches.map(m => m.metadata),
    examCandidates: examCandidates.matches.map(m => m.metadata),
  };
}
```

---

### PF-3.6 — Prompt Builder + LLM Call (FULL, implementation-ready)

**Goal:** Build a single, deterministic prompt bundle that (1) *forces* the model to obey your **Policy**, (2) keeps it anchored to **Pinecone-retrieved content**, and (3) returns **strict JSON only** that you can validate and render into UI cards.

> **Key rule:** The LLM does **not** query Pinecone. The backend retrieves and injects `unitAnchors[]`, `unitContentChunks[]`, and `examCandidates[]` into `CONTEXT`.

---

## PF-3.6.1 Inputs to `buildTutorPrompt()`

Backend constructs these per turn:

* `POLICY: TutorPolicy` (computed deterministically; not stored)
* `CONTEXT: { unitAnchors[], unitContentChunks[], examCandidates[], snapshotLiteForLLM, clientEvent }`
* `STUDENT_INPUT: { messageText | messageLatex, optional voiceTranscript }`

---

## PF-3.6.2 System Prompt — Core Rules (paste verbatim)

```text
You are Paper Video Tutor. Behave like a great private tutor.

You MUST:
- Obey the POLICY object. If POLICY disallows an action, you may not choose it.
- Stay anchored to CONTEXT only. Do not introduce topics that are not supported by unitAnchors/unitContentChunks/examCandidates.
- Default to Socratic style: ask ONE question at a time when clarification is needed.
- Avoid info dumps. Keep replies short and interactive.
- Never claim mastery or write to the database. Mastery is decided by the backend.
- Drills are ephemeral (generated live). Do NOT assume drills are stored or reusable.
- Exam questions are NOT generated. You may only reference question IDs from CONTEXT.examCandidates.
- Confusion tags are NOT produced by you. Do not output or infer confusion tags.

Output format:
- You MUST output JSON only.
- No markdown. No extra text. JSON must be parseable.
```

---

## PF-3.6.3 System Message — POLICY (backend injects JSON)

**You include the *actual* JSON here (not a placeholder).**

```json
POLICY: {
  "turnId": "uuid",
  "focusUnitId": "ALG-01",
  "primaryTargetUnitId": "ALG-01",
  "prereqBlockingUnitId": "ALG-00",
  "scopedUnitIds": ["ALG-01","ALG-00"],
  "allowedActions": ["SOCRATIC_QUESTION","CONCEPT_CARD","DRILL_CARD","EXAM_BLOCK"],
  "stuck": false,
  "examReady": false,
  "desiredExamTier": "bronze",
  "examAvailability": "available",
  "constraints": {
    "maxConceptWords": 170,
    "maxWorkedExamples": 1,
    "drillMaxSteps": 2
  }
}
```

---

## PF-3.6.4 System Message — CONTEXT (backend injects retrieval arrays + snapshotLite)

**Important:** `examCandidates[]` should already be filtered by:

* `POLICY.scopedUnitIds`
* `POLICY.desiredExamTier`
* lockout state / revisit needs (your existing exam API is authoritative)

```json
CONTEXT: {
  "unitAnchors": [
    { "unitId": "ALG-01", "title": "Linear Equations", "anchorText": "..." }
  ],
  "unitContentChunks": [
    { "unitId": "ALG-01", "chunkType": "transcript|pdf", "text": "...", "sourceId": "..." }
  ],
  "examCandidates": [
    { "questionId": "EX-2019-ALG-14", "unitIds": ["ALG-01"], "difficultyTier": "bronze", "tags": ["linear_equations"] }
  ],
  "snapshotLiteForLLM": {
    "focusUnitId": "ALG-01",
    "focusMasteryTier": "bronze",
    "unitsInProgress": ["ALG-01","ALG-00"],
    "revisit": { "lockedCount": 2, "nextEligibleAt": "ISO" }
  },
  "clientEvent": { "type": "NONE" }
}
```

---

## PF-3.6.5 User Message Wrapper — Student Input + JSON Contract (paste verbatim)

```text
Student input:
- messageText: <...> (or messageLatex: <...>)

Return JSON matching EXACTLY one of the allowed actions.

Required JSON shape:
{
  "mapped_units": [{"unit_id": string, "confidence": number}],
  "action": "SOCRATIC_QUESTION" | "CONCEPT_CARD" | "DRILL_CARD" | "EXAM_BLOCK",
  "target_unit_id": string,
  "tutor_text": string,
  "turn_analysis": {
    "student_intent": "solve" | "explain" | "check" | "stuck" | "unknown",
    "understanding_signal": "confident" | "uncertain" | "confused",
    "suggested_prereq_units": string[]
  },

  "concept_card"?: {
    "key_ideas": string[],
    "worked_example"?: {
      "problem_latex": string,
      "final_answer_latex": string,
      "steps_latex"?: string[]
    }
  },

  "drill_card"?: {
    "prompt": string,
    "question_latex": string
  },

  "exam_suggestion"?: {
    "question_id": string,
    "difficultyTier": "bronze" | "silver" | "gold"
  }
}

Hard constraints:
- action MUST be in POLICY.allowedActions.
- target_unit_id MUST be in POLICY.scopedUnitIds.
- If action="SOCRATIC_QUESTION": do NOT include concept_card/drill_card/exam_suggestion.
- If action="CONCEPT_CARD": include concept_card; key_ideas max 3; worked_example max 1.
- If action="DRILL_CARD": include drill_card; keep it 1–2 steps.
- If action="EXAM_BLOCK": exam_suggestion.question_id MUST be from CONTEXT.examCandidates,
  and exam_suggestion.difficultyTier MUST equal POLICY.desiredExamTier.
- If POLICY.prereqBlockingUnitId != null, your target_unit_id should normally be that unit.
- If POLICY.examAvailability is "locked" or "none", do not choose EXAM_BLOCK.
```

---

## PF-3.6.6 Output Handling (what happens next)

* Backend parses JSON.
* Backend validates **against Policy** and **Context constraints** (this is PF-3.7).
* If invalid → **no retries** → deterministic fallback action (PF-3.7).

---

## PF-3.7 — Validation + deterministic fallback (no second chances)

### PV-AI-012 — validateAgainstPolicy() + fallbackAction()

**Concept explainer:**
We never retry the LLM. If it violates policy, we override deterministically.

**Rules**

* `action` ∈ `allowedActions`
* `target_unit_id` ∈ `scopedUnitIds`
* If `EXAM_BLOCK`: questionId must be from `examCandidates` and not locked (per existing API)
* If `CONCEPT_CARD`: policy.stuck must be true
* Drill card must be short (we rely mostly on prompt; optional heuristic checks)

**Fallback logic**

* Prefer `SOCRATIC_QUESTION`, else `DRILL_CARD`.

---

# EPIC E — Drill Workflow (generated, not stored as entities)

## PF-4.1 — Drill cards are ephemeral (generated + graded live)

### PV-AI-013 — Drill grading endpoint: `POST /threads/:id/drill/grade`

**Concept explainer:**
Drills are not stored in a drill database or bank. They are generated and graded on demand. Snapshot stores only evidence counters + confusion tags from grading.

**Request (must include drill text)**

```json
{
  "courseId":"MATH-G10",
  "unitId":"ALG-01",
  "drill": { "prompt":"Solve for x", "question_latex":"2x+3=11" },
  "studentAnswer":"x=4"
}
```

**Drill grading prompt (LLM)**

```text
You are grading a short drill question (1–2 steps).
Return JSON only:
{ "isCorrect": boolean, "feedbackText": string, "commonMistakeTag": string|null }.
If incorrect, choose a single commonMistakeTag from a fixed list provided.
```

**Confusion tags rule**

* If `isCorrect=false` and `commonMistakeTag != null`: increment `snapshot.unitProgress[unitId].confusionTags[tag] += 1`
* Nowhere else generates tags.

**After grading**

* UI shows feedback + “Continue” button (details in EPIC G).
* Next tutoring step happens via `/turn` (clientEvent), or student types next message.

---

# EPIC F — Exam Workflow (MCQ + lockout API + revisit + Bronze/Silver/Gold mastery)

## PF-5.1 — Exam difficulty tiers and mastery tiers

### PV-AI-014 — Bronze/Silver/Gold mapping (deterministic)

**Concept explainer:**
Each unit can reach Bronze, Silver, Gold mastery by passing exam questions at that difficulty tier (plus drill readiness).

* Bronze mastery: pass bronze-tier exam (and drill readiness)
* Silver mastery: pass silver-tier exam (and already bronze)
* Gold mastery: pass gold-tier exam (and already silver)

---

## PF-5.2 — Exam question attempt + support view events (existing API authoritative)

### PV-AI-015 — MCQ submit endpoint (calls existing exam API)

**Concept explainer:**
The lockout time and pass/fail is already tracked by your exam system. We consume it and update snapshot.

**Endpoint**

* `POST /threads/:id/exam/mcq-submit`

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

**Snapshot updates**

* Mark question touched: `examTouched[questionId]=true`
* If incorrect OR support viewed causes lock: add/keep in revisitQueue `{questionId:{unitId,tier}}`
* If correct:

  * increment `passedByTier[tier]`
  * recompute `masteryTier`

> Lockout timing itself comes from the existing exam API, not from snapshot.

---

### PV-AI-016 — Support viewed endpoint (calls existing exam API)

**Endpoint**

* `POST /threads/:id/exam/support-viewed`

**Request**

```json
{
  "courseId":"MATH-G10",
  "unitId":"ALG-01",
  "questionId":"EX-2019-ALG-14",
  "supportType":"memo|video"
}
```

**Effect**

* existing API enforces lockout
* snapshot adds questionId to revisitQueue

---

## PF-5.3 — Exam availability per tier (batch status lookup)

### PV-AI-017 — `computeExamAvailabilityForTier()` uses candidate IDs only

**Concept explainer:**
We never track 15k states. We only query status for a small candidate set from Pinecone/DB.

Outputs:

* `available`: some candidates attemptable now
* `locked`: none attemptable, but some locked (nextEligibleAt returned)
* `none`: no candidates exist

---

# EPIC G — Frontend UI (Angular) — Full Interaction Spec (Beefed Up)

## PF-6.0 — UI layout: unified workspace

### PV-AI-018 — Tutor Workspace Page layout (threads + chat + right panel)

**Concept explainer:**
This page is the product. It must feel simple despite the DAG and policy engine.

### Layout (desktop)

* **Left sidebar:** Thread list + “New session”
* **Center:** Chat timeline (messages + cards)
* **Bottom center:** Composer (text input, voice button, send)
* **Right panel (Context Strip / SnapshotLite):**

  * Focus unit + mastery tier badge (Bronze/Silver/Gold)
  * Prereq nudge (if blocked)
  * Revisit queue summary (locked count + next eligible time)
  * Progress summary (bronze/silver/gold counts)
  * Optional: “Learning Path” shortcut to progress board page

### Layout (mobile)

* Left sidebar collapses to hamburger
* Right panel collapses to “Info” drawer

---

## PF-6.1 — SnapshotLite (why UI needs it)

### PV-AI-019 — SnapshotLite builder + UI binding

**Concept explainer:**
SnapshotLite is a small UI summary so the frontend doesn’t implement DAG logic.

**SnapshotLite example**

```json
{
  "focus": { "unitId":"ALG-01", "title":"Linear Equations", "masteryTier":"bronze" },
  "prereqNudge": { "unitId":"ALG-00", "title":"Inverse Operations" },
  "revisit": { "lockedCount": 2, "nextEligibleAt":"ISO", "nextQuestionId":"EX-2019-ALG-14" },
  "progress": { "bronze": 12, "silver": 4, "gold": 1, "total": 40 }
}
```

**Field glossary**

* `focus`: what the tutor is working on now.
* `prereqNudge`: what must be learned first (if any).
* `revisit`: exam revisit info surfaced as a nudge.
* `progress`: high-level mastery distribution across the course.

---

## PF-6.2 — Card UX specs with exact buttons and placement

### PV-AI-020 — Concept Card component

**Placement:** In chat timeline as a card message.

**Contents**

* Title: “Key idea”
* Bullet key ideas (max 3)
* Optional worked example section (collapsed by default)
* Buttons (bottom-right of card):

  * **Try a Drill** (primary)
  * **Ask follow-up** (secondary; focuses composer)

**Try a Drill action**

* Sends `/turn` with:

```json
{ "clientEvent": { "type":"REQUEST_DRILL", "unitId":"ALG-01" } }
```

---

### PV-AI-021 — Drill Card component (assessment)

**Placement:** In chat timeline as a card message.

**Contents**

* Prompt + LaTeX question
* Answer input box (inline)
* Buttons (bottom-right):

  * **Check Answer** (primary)
  * **I’m stuck** (secondary; triggers a /turn event hint)

**On “Check Answer”**

* Call `/drill/grade` with drill content + answer
* Show returned feedback in-card

**After grading, show**

* Feedback text
* Status pill: Correct / Incorrect
* Buttons (bottom-right):

  * **Continue** (primary)
  * **Try another** (secondary)

**Continue placement & behavior**

* **Continue** is inside the drill card, bottom-right.
* It triggers `/turn` with:

```json
{ "clientEvent": { "type":"DRILL_CONTINUE", "unitId":"ALG-01", "lastResult":"correct|incorrect" } }
```

This prompts the tutor to either:

* move to next drill,
* introduce an exam block (if policy allows),
* or provide a concept card (if stuck unlocked).

> This gives your dev a very concrete UI loop: drill → grade → continue.

---

### PV-AI-022 — Exam Block component (MCQ + lockout + tier)

**Placement:** In chat timeline as a card message.

**Contents**

* Header: Exam Question + tier badge (Bronze/Silver/Gold)
* Question element (your existing controlled renderer)
* MCQ options
* Buttons area (bottom):

  * **Submit** (primary)
  * **View Memo** (secondary)
  * **Watch Video** (secondary)
  * **MCQ Practice** (optional if your system has it)

**Lockout behavior (from existing API)**

* If locked:

  * Disable Submit and options
  * Show lock countdown (“Available in 18h 12m”)
  * Show button: **Add to revisit list** (optional, local-only)

**On Submit**

* Call `/exam/mcq-submit` and update snapshot
* If correct:

  * show “Passed” badge
  * show **Continue** button (bottom-right)
* If incorrect:

  * show “Locked for 24h” note + countdown
  * show **Continue** button (bottom-right) to return to drills

**Continue on exam**

* Same pattern: triggers `/turn`:

```json
{ "clientEvent": { "type":"EXAM_CONTINUE", "unitId":"ALG-01", "questionId":"EX-...", "result":"passed|locked" } }
```

---

## PF-6.3 — Composer + Voice

### PV-AI-023 — Composer interactions

* Send button → `/turn`
* Voice button:

  * record → transcribe → show confirm modal with detected LaTeX
  * confirm → `/turn {messageLatex: "..."}`

---

# EPIC H — Observability & Debugging (to prevent stuff-ups)

## PF-7.1 — Turn logging: full chain

### PV-AI-024 — logTurn() with policy + validation + exam availability

Log these each turn:

* mapped_units + confidence
* focusUnitId + blocker
* allowedActions
* desiredExamTier + examAvailability + nextEligibleAt
* selected action and whether it passed validation or fell back

---

## PF-7.2 — Dev debug drawer (feature flag)

### PV-AI-025 — Debug panel shows policy + snapshotLite + scores

* policy JSON
* top candidate units + scores
* revisit summary
* mastery tier per focus unit

---

# Exhaustive Input → Response Triggers (updated to match your final rules)

## Input A — `/turn` (student message)

### A1: Mapped unit confidence high, prereqs satisfied

* Policy likely allows Drill; exam may become allowed after readiness
* Typical response: Socratic or Drill Card

### A2: Mapped unit high, prereqs missing

* Policy sets blocker; focus shifts to prereq
* Response: Drill Card on prereq (or Socratic)

### A3: Mapped unit ambiguous/low confidence

* Response: Socratic clarifier (no card)

### A4: Evidence shows stuck (attempts≥2 and streak=0)

* Policy allows Concept Card
* Response: Concept Card → Drill

### A5: ExamReady true and examAvailability available (tier-aware)

* Policy allows Exam Block
* Response: Exam Block at desired tier (bronze/silver/gold)

---

## Input B — `/drill/grade` (generated drill)

### B1: Correct

* Update drill counters
* Confusion tags unchanged
* UI shows feedback + Continue → `/turn (DRILL_CONTINUE)`

### B2: Incorrect

* Update counters; if wrong repeatedly, stuck triggers next policy
* Confusion tag increments from grading only
* UI shows feedback + Continue → `/turn`

---

## Input C — Exam events (existing API)

### C1: MCQ submit correct

* passedByTier[tier] increments
* mastery tier may upgrade
* Continue → `/turn (EXAM_CONTINUE passed)`

### C2: MCQ submit incorrect OR support viewed

* existing API locks for 24h
* snapshot marks revisitQueue entry
* UI shows countdown + Continue → `/turn`

---

# Quick “developer sanity” notes (to avoid misunderstandings)

* **LLM never generates confusion tags**; only drill grading can.
* **LLM never queries Pinecone**; backend injects retrieved arrays into the prompt.
* **No retries**; invalid LLM outputs are overridden deterministically.
* **No drill persistence** beyond what appears in the message payload and what the client sends back for grading.
* **Exam lockouts are authoritative from your existing exam API**; snapshot just tracks “needs revisit”.

---
