`primaryTargetUnitId`

Meaning: What the student asked about (semantic mapping result).

Source: Mapping step (LLM or embedding classifier).

It should NOT be influenced by prereqs.

Why it exists: So you always preserve the student’s intent, even if you can’t address it immediately.

prereqBlockingUnitId

Meaning: The single prerequisite unit (or nearest prerequisite) that deterministically blocks progress on primaryTargetUnitId.

Source: DAG + snapshot mastery state.

Why it exists: To force correct sequencing when the student is trying to jump ahead.

focusUnitId

Meaning: What we will actually tutor right now.

Source: Deterministic policy computation.

Rule:

If prereqBlockingUnitId != null → focusUnitId = prereqBlockingUnitId

Else → focusUnitId = primaryTargetUnitId (or current thread focus if you’re scope-locking)

So:
primaryTargetUnitId = student intent
prereqBlockingUnitId = why we can’t go there yet
focusUnitId = what we do now
