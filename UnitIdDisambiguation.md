`primaryTargetUnitId`

Meaning: What the student asked about (semantic mapping result).

Source: Mapping step (LLM or embedding classifier).

It should NOT be influenced by prereqs.

Why it exists: So you always preserve the student’s intent, even if you can’t address it immediately.

`prereqBlockingUnitId`

Meaning: The single prerequisite unit (or nearest prerequisite) that deterministically blocks progress on primaryTargetUnitId.

Source: DAG + snapshot mastery state.

Why it exists: To force correct sequencing when the student is trying to jump ahead.

`focusUnitId`

Meaning: What we will actually tutor right now.

Source: Deterministic policy computation.

**Rule:**

If `prereqBlockingUnitId` != null → `focusUnitId` = `prereqBlockingUnitId`

Else → `focusUnitId` = `primaryTargetUnitId` (or current thread focus if you’re scope-locking)

So:
`primaryTargetUnitId` = student intent

`prereqBlockingUnitId` = why we can’t go there yet

`focusUnitId` = what we do now

That’s the whole relationship.

**Does prereqBlockingUnitId “override” the other fields?**

It doesn’t overwrite them — it controls focus.

You keep `primaryTargetUnitId` as the “north star”.

You set `focusUnitId` to the blocker until it’s cleared.

This is how you avoid losing the student’s goal while still enforcing prerequisites.

In practice, the LLM does not override anything — the policy does.

**Where scopedUnitIds ties in**

`scopedUnitIds` is the hard fence for the turn/thread. It must contain the units the LLM is allowed to target.

A clean deterministic rule:

Always include `focusUnitId`

Always include `primaryTargetUnitId` (so the tutor can reference it: “we’re working toward X”)

Include prereq chain / neighborhood needed to support the next steps

Example:
```
scopedUnitIds = unique([
  focusUnitId,
  primaryTargetUnitId,
  prereqBlockingUnitId,
  ...immediatePrereqs(focusUnitId),
  ...smallNeighborhood(focusUnitId)
]).slice(0, MAX_SCOPE)
```

Then the LLM constraint is simple:

`target_unit_id` ∈ `scopedUnitIds`

“Usually target `focusUnitId`” (unless your policy explicitly allows otherwise)

**How prereqBlockingUnitId is derived from DAG + Snapshot**

Assume your DAG edges mean:

```// ["ALG-00","ALG-01"] means ALG-00 is a prerequisite of ALG-01
type PrereqEdge = [prereqUnitId: string, unitId: string];
```

You compute blocking prereq like this:

Start at `primaryTargetUnitId`

Find its direct prerequisites from the DAG

For each prereq, check the snapshot: is it mastered enough (e.g., at least bronze)?

If not mastered → that prereq is the `prereqBlockingUnitId`

If mastered → recurse into its prereqs (because chains exist)

A simple deterministic TS version:

```type MasteryTier = "none" | "bronze" | "silver" | "gold";
type UnitStatus = "not_started" | "in_progress" | "mastered";

type Snapshot = {
  unitProgress: Record<string, { status: UnitStatus; masteryTier: MasteryTier }>;
};

type DagIndex = {
  prereqsOf: Record<string, string[]>; // unitId -> prereq unitIds
};

function tierRank(t: MasteryTier): number {
  return t === "none" ? 0 : t === "bronze" ? 1 : t === "silver" ? 2 : 3;
}

function getUnitMastery(snapshot: Snapshot, unitId: string): MasteryTier {
  // Missing from unitProgress => treat as "none"
  return snapshot.unitProgress[unitId]?.masteryTier ?? "none";
}

// Return the *closest unmet prereq* (deterministic order), else null
export function computePrereqBlockingUnitId(
  primaryTargetUnitId: string,
  dag: DagIndex,
  snapshot: Snapshot,
  requiredTier: MasteryTier = "bronze"
): string | null {
  const visited = new Set<string>();

  // BFS from direct prereqs upward (closest prereq first)
  const queue: string[] = [...(dag.prereqsOf[primaryTargetUnitId] ?? [])];

  while (queue.length) {
    const prereq = queue.shift()!;
    if (visited.has(prereq)) continue;
    visited.add(prereq);

    const mastery = getUnitMastery(snapshot, prereq);
    if (tierRank(mastery) < tierRank(requiredTier)) {
      return prereq; // block here
    }

    // prereq is mastered enough -> check its prereqs too (for longer chains)
    const next = dag.prereqsOf[prereq] ?? [];
    queue.push(...next);
  }

  return null;
}
```

Then your policy logic becomes clean:
```
policy.primaryTargetUnitId = primaryTargetUnitId; // from mapping step

policy.prereqBlockingUnitId = computePrereqBlockingUnitId(primaryTargetUnitId, dag, snapshot, "bronze");

policy.focusUnitId = policy.prereqBlockingUnitId ?? policy.primaryTargetUnitId;
```
