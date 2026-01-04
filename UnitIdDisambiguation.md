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
