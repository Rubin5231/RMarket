# RMarket — development log and decision register

> A durable project memory alongside Git history. Updated by the assistant during RMarket development sessions; not a transcript. Source of truth for implemented code is the `main` branch. Items labelled "planned" have not been verified as implemented.
>
> Last reconciliation: 2026-11-18, repository head before this document: `a0b60e7a1a5c321fba5a028e74c7eaadae96dd5c` (A3.4.4.4).

## Product goal and working principles

- Product: RMarket — X++ crypto trading robot on Dynamics 365 Finance & Operations.
- Initial operating concept: ~50 USDT in one exchange account, allocated virtually between 3–5 subaccounts; each can run a separate strategy and multiple subaccounts may use the same instrument.
- First trading iteration should be deliberately simple, workable with a small deposit, support both long and short, and avoid premature complexity. Some modes may be hardcoded initially and generalized later.
- **Complete analytics before operational trading**; adjust analytics when required by trading execution.
- Workflow: one small task, implement and test locally, commit/push, then next task. User owns local AX environment and builds; assistant reviews GitHub sources and proposes X++/SQL.
- Keep SQL for business data scoped to `Partition = 5637144576` and `DataAreaId = 'dat'`; resolve X++ enum numeric values through `ENUMIDTABLE` + `ENUMVALUETABLE`, rather than hardcoding enum numbers.
- X++ code comments in English. English labels/captions are acceptable temporarily. Preserve the user's chosen form structure instead of imposing Microsoft standard layout.
- Important: **Orders tab uses `RMarketConditionTable` for arithmetic price expressions**, not the new boolean event-tree model. Do not change that logic incidentally.

## Architecture and decisions (retain even when chat context is lost)

### Strategy events and conditions — A3

- `RMarketStrategyAction`: an action can be triggered by any of several alternative **root events** (logical OR between root events).
- `RMarketConditionGroup`: a named root event (`ParentRecId=0`) or nested boolean group; `RMarketConditionOperator` is AND/OR; nesting can be arbitrarily deep.
- Child group: same `RMarketStrategyActionRecId`, `ParentRecId` points to parent group.
- Leaf `RMarketConditionTable`: reuses existing polymorphic `Relation` + `RefRecId`. For group leaves: `Relation=RMarketConditionRelation::RMarketConditionGroup`, `RefRecId=group.RecId`. **Do not add a separate `GroupRecId` field.**
- Sample supported expression: `e AND ((a AND b) OR (c AND d))`.
- Empty group evaluates false; empty action does not signal; if root groups exist, they are evaluated with OR. Legacy direct action conditions remain a fallback, joined with AND when no root events exist.
- Validate parent existence, same action ownership and absence of cycles at write time. XOR is deferred.
- Condition evaluates `What` vs `With` using `Comparison`; possible sources are Const, Indicator, Candle, LosingTradesInARow and DepositPercentDrawdown. The last two were not implemented in `calcValueW()` as of 2026-11-18 and must not be mistaken for working calculations.
- At candle boundaries use an appropriate negative shift when relying on the previous **closed** candle, to avoid lookahead. Confirm strategy-specific timing during integration.
- The strategy-form top overview and Actions/Orders tabs must remain. Under Actions: left event/group/condition tree; right editor. Selecting group shows Name and AND/OR; selecting leaf shows What / Comparison / With.

### Why the design is shaped this way

- Root events represent alternative market triggers; nested groups express compound logic, so actions do not need a fixed one-level conditions list.
- `Relation + RefRecId` already exists and is suitable for polymorphic parent links; no redundant foreign-key column.
- The event tree belongs only to **Actions**; order pricing expressions in **Orders** remain a separate concern.
- Favor focused updates in the tree (preserve selection and expansion) over rebuilding it for each edit.

## Completed milestones

| Area | State | Notes |
| --- | --- | --- |
| A0 API and instruments | Completed earlier | Analytics foundation |
| A1 candle history | Completed earlier | Candle ingestion/history |
| A2 indicators | Completed earlier | Incremental recovery test: 400 deleted indicator rows restored; 60 configurations checked with zero missing slots |
| A3.1 | Completed earlier | Corrected missing-indicator check in `RMarketConditionTable.calcValueW()` |
| A3.3.1 | Implemented | `RMarketConditionGroup` table, operator enum, group relation |
| A3.3.2 | Implemented | Recursive group `calc()`, parent/cycle `validateWrite()` |
| A3.3.3 | Implemented | Action `calcSignal()`: root event OR; fallback legacy AND |
| A3.3 tests | Test class present | `RMarketJobTest_ConditionTree` tests nested logic and validation; test method reports 10 checks. User confirms all work through A3.4.4.4 has been tested in AX; assistant has not rerun those tests |
| A3.4.1–A3.4.4.3 | Implemented | Action tree on `RMarketStrategy`, selection/editor, creating events/groups/conditions |
| A3.4.4.4 | Implemented and user-tested in `a0b60e7` | Central caption formatter and `updateSelectedConditionTreeNode()`, `modified()` handlers for group and condition fields; user confirmed correct and convenient live editing on 2026-11-18 |

### Current UI code state, verified from GitHub 2026-11-18

- `RMarketStrategy.conditionTreeNodeCaption(Common)` formats group names as root/event or nested group, including AND/OR; condition captions currently show only `● Condition N`.
- `RMarketStrategy.updateSelectedConditionTreeNode(Common)` checks the selected tree item's table/record identity and updates its text through `FormTreeControl.setItem()`.
- `modified()` controls cover group name/operator, and condition source, indicator, candle value, value, interval, shift, comparison, and mirrored right-side fields.
- `RMarketStrategy.updateFormDesign()` currently toggles the group vs leaf editor, and disables deletion and Up/Down. Conditional visibility of individual What/With fields is not yet implemented.
- `RMarketStrategy.addConditionTreeNodes()` currently enumerates nested groups and leaves in **separate loops**, so mixed children do not yet display in a single global `Num` order.
- Latest committed `BuildProjectResult.xml` shows no compile errors and three existing warnings about external assemblies, generated 2026-11-18 00:26 MSK. It is not a substitute for functional form testing.

## Next work (ordered, incremental)

1. **A3.4.4.5 — Human-readable condition captions (code implemented in `599784d8`, integration bug under investigation).** Replace `● Condition N` with a compact, deterministic English expression combining left operand, comparison sign and right operand. Reuse one formatter for initial tree construction and live field edits. Cover Const, Candle, Indicator first; unsupported/unconfigured sources should be visibly explicit, not silently presented as valid.
2. **A3.4.4.6 — Dynamic field visibility.** Make each operand editor show only fields relevant to its source: Const → Value; Indicator → IndicatorId, Interval, Shift; Candle → CandleValue, Interval, Shift; other future sources → their relevant controls after their semantics exist. Keep Comparison visible. Refresh on tree selection and Source/SecondSource modification. **Scope this to Actions, not the Orders expression editor.**
3. Make tree deletion safe: leaf deletion; group/event deletion with intended child handling and selection updates. Confirm UX before implementing destructive operations.
4. Reconcile mixed group+condition sibling `Num` ordering; implement Up/Down with shared sibling order.
5. Improve validation for changing source, invalid indicator IDs, missing data and parent ownership changes.
6. Exercise the UI with a nested expression and verify that captions, selection, persistence, and computation agree.
7. Finish A3 strategy signaling and subsequent analytic integration before touching live operational trading.

## Open risks and deferred questions

- Source types `DepositPercentDrawdown` and `LosingTradesInARow` currently return their default value in `calcValueW()` (no explicit calculation). Implement or explicitly reject before those options become actionable.
- Do not assume all edited form fields have persisted simply because caption changed: verify save/refresh behavior in AX.
- `RMarketConditionGroup.validateWrite()` guards parent cycles but review behavior when a group with descendants is assigned a different action.
- Condition records can represent both boolean signals and order formulas; changes must not break the Orders tab.
- Existing strategy “VoltTrend Rider” (`001`) has placeholder conditions and should not be treated as production-ready or profit-tested.
- Business model (initial small balance, virtual subaccount sizing, costs/fees, risk) still needs explicit operational rules before any live orders.

## Session notes

### 2026-11-18 — Conversation-history recovery and A3.4.4.4 review

- User reported loss of a substantial chat segment following a computer restart. **The cause of the missing history is unknown**; do not attribute it to local code changes.
- Recovered architecture and work milestones from conversation context and checked actual repository commits and source code. Work was preserved through `a0b60e7`.
- User implemented updating captions when fields of groups/conditions change and confirmed live-edit testing is successful. Code review found correct targeted-update structure; condition text still placeholder, per-source field visibility pending.
- Decision: keep this Markdown file in GitHub as a durable **decision and task log**, maintained by assistant when actively working on the project, in addition to ordinary git commits. No autonomous/background updates are implied.
- Protocol: whenever a task concludes, record the outcome, technical decisions and exact next task here. Clearly mark `implemented` vs `tested` vs `planned`. Do not overwrite historical decisions without noting why they changed.

### 2026-11-18 — Confirmed validation and A3.4.4.5 started

- User explicitly confirmed testing of all completed work through A3.4.4.4, including live caption refresh on both group and condition fields.
- A3.4.4.5 is the current implementation task; no code change for that task is committed yet. Code should use existing `RMarketConditionTable` fields and `RMarketStrategy.conditionTreeNodeCaption()`.
- User prefers Visual Studio Git UI over commands for syncing the root-level log. `Fetch` alone does not update the working tree; `Pull` must target the correct clone and `main`. The root-level Markdown file need not appear in Solution Explorer unless included as a solution item.

### 2026-11-18 — A3.4.4.5 implemented; blocking UI selection bug

- GitHub main commit `599784d8cc4832015327d902ef08bad135e9822c` implements `conditionOperandCaption()` and full leaf captions; user tested caption updates.
- **Bug confirmed with user screenshots:** tree selection can point at condition RecId `5637155827` while editor cursor is `5637153576`; on reopening an action the rebuilt tree shows `0 > 0` while selected editor shows `0 < 0`. The user committed `4c7ef3ed` and merged to `9a79b025805168ce80d38010f263f9a00b8818bb` after adding `LinkType=Active` and checking `findRecord()` results. This did not fully fix the problem.
- Investigation narrowed to stale/mismatched editor datasource: parent/child linkage makes `RMarketConditionTable_DS.findRecord()` unreliable during tree selection even with `LinkType=Active`. **Proposed, not yet implemented:** make the editor datasource independent (remove `JoinSource`), filter its query directly by selected leaf `RecId` with `SysQuery::findOrCreateRange` and `executeQuery()`, and verify cursor `RecId` matches tree selection. Preserve edits using `leaveRecord()` before changing the query. Update `createCondition()` positioning to use the same helper. Do not mark A3.4.4.5 done until selecting arbitrary leaves and reloading actions produces matching captions/editor values.
- Separate warning: a lookup on `With_SecondIndicatorId` was opened when `SecondSource=Candle`; right indicator's table relation is source-qualified, so hidden irrelevant control should prevent that action. Still separately verify lookup when `SecondSource=Indicator`.
- User confirmed Git UI successfully synchronized development log; local file is not needed as an AX solution item.

### 2026-11-18 — Further datasource repro (still unresolved)

- User changed `RMarketConditionTable_DS.LinkType` to `Active` and instrumented failed `findRecord()`.
- On first opening the strategy form, selecting a real leaf reports: `Cannot position condition 5637155827. Current: 5637153576.`
- After that startup failure, traversing leaves within/across groups and editing fields appears to work; this supports a transient child datasource query/positioning issue at first load, but it is **not yet proven**.
- User also reported an unexpected result after editing a comparison operator on a newly created condition in another action, switching to a different action and back. Only the first screenshot (failed positioning) was available to the assistant in that turn; do **not** guess the second screenshot's contents.
- Next diagnosis: retry failed `findRecord()` after one child datasource `research(false)`, saving pending edits with `leaveRecord()` before switching leaves/actions; separately verify comparison-field persistence after action switches. Do not re-query on every selection or assume tree captions mean DB writes succeeded.
- A3.4.4.5 remains blocked pending reliable editing, persistence and startup positioning.

### 2026-11-18 — Query ownership refinement before implementing editor selection fix

- User requested that **all editor datasource record selection be driven through `SysQuery::findOrCreateRange()` / `QueryBuildRange` and `executeQuery()`, not through `FormDataSource.findRecord()`**. Agreed: the tree's selected `RecId` uniquely identifies the editor record; datasource should be unlinked from `RMarketConditionGroup` and filtered directly by that `RecId`.
- User requested initial `RecId=0` range in **`RMarketConditionTable_DS.init()` after `super()`**, not in `FormRun.init()`. Agreed: the datasource owns query initialization.
- A form-level selection helper may apply the range and `executeQuery()`; guard unsaved changes via `leaveRecord()`. Initial state and group/no-leaf selection should select no record. Distinguish direct table `findRecId()` (read/validation) from `FormDataSource.findRecord()` (positioning) — only the latter is to be avoided.
- This is a **revised design, not implemented yet**; user explicitly had not made changes at this point. Validate initial open, same-group leaf switching, cross-group and cross-action switching, and editing persistence.

### 2026-11-18 — Code review of RecId-filtered editor datasource (user implementation)

- Confirmed in `main` at `4dd4a5155f51a7fc09857bd2c5961f478a91fdc1`: standalone `RMarketConditionTable` datasource (no JoinSource), `init()` sets `RecId=0`, `selectConditionForEditing(RecId)` changes datasource QueryBuildRange via `SysQuery::findOrCreateRange`, runs `executeQuery()`, verifies cursor `RecId`; `selectionChanged()` and `createCondition()` call the helper. Latest committed build has no errors, 3 pre-existing assembly warnings. **Compiled, not yet reported functionally retested.**
- Review found two omissions: selecting a group/event does not call `selectConditionForEditing(0)`, and `RMarketStrategyAction_DS.active()` rebuilds/clears the tree without clearing the now-independent editor datasource. Propose calling the helper to clear editor on both transitions, then retest.
- Follow-up robustness: the helper returns `false` for failed `leaveRecord()` or editor cursor mismatch, but current callers ignore this. Add failure handling if reproducible; don't silently allow selection and editor to diverge. Do not close A3.4.4.5 before behavioral verification.

### 2026-11-18 — Editor responsiveness observation

- User reported visible 0.2–0.5 second delay when selecting a leaf: tree highlight changes before old right-side fields refresh. Current `selectConditionForEditing()` calls `leaveRecord()`, changes a `RecId` range, then `executeQuery()` on every leaf click. No profiler measurements yet, so precise cost attribution is unverified.
- **Optimization proposal (not implemented):** load conditions for the selected **action** in one datasource query (restrict to the group records belonging to that action, preserving `Relation=RMarketConditionGroup`), then navigate among loaded rows via datasource positioning, potentially `positionToRecord()`. Refresh query only when action changes or records are added/removed. Verify that editing, save/validation, and cross-group navigation remain correct.
- Small safe interim guard: return early from `selectConditionForEditing()` when datasource cursor already matches requested RecId, eliminating redundant `executeQuery()` after creating and selecting a new leaf. This does **not** remove the per-leaf latency for different records.
- Do not substitute `refresh()` alone for a query change; it only redraws currently loaded data. Do not assume `setRecord()` can replace queries for ordinary persisted records without verifying edit/save semantics.

## How to use

- File path: `DEVELOPMENT_LOG.md` at repository root.
- Assistant may create/update it through the connected GitHub app. User can also edit via GitHub web UI, a local text editor, or Visual Studio.
- If this file is changed directly on GitHub, the local clone must incorporate the new commit before the next push (normally `git pull --rebase origin main`, after committing/stashing any local edits).
- This file is **not** an automatically recorded chat transcript. Irreplaceable ideas should be added as explicit decisions or questions when they arise.
