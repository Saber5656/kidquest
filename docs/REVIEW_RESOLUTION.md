# Review resolution addendum

- PR: #1
- Base: `main`
- Resolution scope: the six existing review threads listed below.
- This document is a normative design addendum. It records the accepted resolution contract; it does not claim that implementation or tests have already run.
- Bot review is not retriggered.

## PRRT_kwDOTNkI_86OZCTT — repeat-once history

Problem: `repeat=once` must not be reset merely because the current day has changed.

Resolution: the completion decision MUST inspect all historical completion records, or an equivalent persisted terminal state, scoped to the quest and member. A prior terminal completion remains terminal across date boundaries; “today only” is not sufficient for a once-only quest.

Focused verification before resolving this thread: seed a completion on an earlier date, advance the clock across a date boundary, and assert that a second completion is rejected or returns the existing terminal record.

## PRRT_kwDOTNkI_86OZCTU — idempotent completion

Problem: retries and duplicate events can award the same quest more than once.

Resolution: completion persistence MUST be atomic and idempotent. Enforce a unique key such as `(questId, memberId, period)`, where period semantics match the quest repeat mode, and use an upsert/compare-and-set transaction. A duplicate request MUST return the existing pending or terminal record and MUST NOT create a second payout.

Focused verification before resolving this thread: submit the same completion concurrently and sequentially; assert one durable record and one award side effect.

## PRRT_kwDOTNkI_86OZCTV — points snapshot

Problem: approval after a quest edit could award a value different from the value shown when completion was requested.

Resolution: a pending completion MUST store `pointsSnapshot` at creation. Approval MUST award that snapshot, while later quest edits affect only future completions. The audit record MUST expose both quest identity and the captured value.

Focused verification before resolving this thread: create pending state, edit the quest points, approve, and assert the original snapshot is awarded.

## PRRT_kwDOTNkI_86OZCTW — iOS Safari/Home Screen storage

Problem: Safari storage and an installed Home Screen app are not implicitly the same storage container.

Resolution: the real iOS family setup MUST install or launch the target Home Screen app before setup is considered complete. If setup starts in Safari, the product MUST provide an explicit handoff or export/import path and MUST show the user which storage container owns the data. Silent reliance on shared storage is prohibited.

Focused verification before resolving this thread: perform setup in Safari and in the installed app separately, then verify the explicit handoff preserves the same family data without silent loss.

## PRRT_kwDOTNkI_86OZCTX — Vite/PWA base path

Problem: deployment under `/kidquest/` can break assets, manifest, and service-worker scope when paths assume root.

Resolution: production build configuration MUST set Vite `base: '/kidquest/'`. PWA `scope` and `start_url` MUST remain under `/kidquest/`, and registration/fetch paths MUST resolve under that prefix.

Focused verification before resolving this thread: build with the production configuration and inspect asset URLs, manifest fields, and service-worker registration/scope for the exact prefix.

## PRRT_kwDOTNkI_86OZCTY — empty assignee semantics

Problem: an empty `assigneeIds` list must not accidentally assign a child task to parents.

Resolution: when `assigneeIds` is empty, derive assignees only from members whose role is `kid`. Parent/guardian accounts MUST be excluded unless explicitly listed. Add a test for a family containing both roles.

Focused verification before resolving this thread: evaluate empty, explicit-kid, and explicit-parent inputs and assert the role rule and explicit-assignee behavior.

## Verification status

The checks above are required acceptance criteria for implementation. This addendum intentionally reports no test result and no implementation-complete status.