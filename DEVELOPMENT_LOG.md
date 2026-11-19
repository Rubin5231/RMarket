# RMarket — development log and decision register

> A durable project memory alongside Git history. Updated by the assistant during RMarket development sessions; not a transcript. Source of truth for implemented code is the `main` branch. Items labelled "planned" have not been verified as implemented.
>
> Last reconciliation: 2026-11-19, code reviewed at `df1fb61ebf19b31bf1a959f38f6c1952a8188c65` (A3.4.4.6).

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
| A3.4.4.5 | Implemented; normal editor use accepted | Human-readable operand captions; independent condition datasource filtered by RecId. User deferred latency optimization to backlog; explicit full regression results not recorded |
| A3.4.4.6 | Implemented and user-tested in `c77a24ff` | Independent source-dependent visibility in What/With; user confirmed it works excellently on 2026-11-19 |

### Current UI code state, verified from GitHub 2026-11-19

- `RMarketStrategy.conditionTreeNodeCaption(Common)` formats group names as root/event or nested group, including AND/OR; condition captions display English operand expressions with comparison, interval and shift; missing indicator/candle selection is displayed explicitly with `?`.
- `RMarketStrategy.updateSelectedConditionTreeNode(Common)` checks the selected tree item's table/record identity and updates its text through `FormTreeControl.setItem()`.
- `modified()` controls cover group name/operator, and condition source, indicator, candle value, value, interval, shift, comparison, and mirrored right-side fields.
- `RMarketStrategy.updateFormDesign()` currently toggles the group vs leaf editor, and disables deletion and Up/Down. Individual What/With field visibility now follows the selected operand sources, and both source modified() handlers refresh the form. Condition editor uses an independent RecId-filtered datasource.
- `RMarketStrategy.addConditionTreeNodes()` currently enumerates nested groups and leaves in **separate loops**, so mixed children do not yet display in a single global `Num` order.
- Latest committed `BuildProjectResult.xml` shows no compile errors and three existing warnings about external assemblies, generated 2026-11-18 14:57 MSK. It is not a substitute for functional form testing.

## Next work (ordered, incremental)

1. **A3.4.4.7 — Tree node deletion (current task).** Permanently prohibit cascade deletion; only leaf conditions and empty groups/events may be deleted. Protect in UI and at table level (`hasChildren`, `validateDelete`, `delete`). Delete by selected tree node identity, not legacy `currentCondition_DS()`. Preserve expanded branches and select parent or adjacent root after successful deletion. Do not renumber siblings in this task.
2. Reconcile mixed group+condition sibling `Num` ordering; implement Up/Down with shared sibling order.
3. Exercise the UI with a nested expression and verify captions, selection, persistence, and computation agree.
4. Finish A3 strategy signaling and subsequent analytic integration. Complete safety-critical configuration validation and runtime fault handling (backlog below) before enabling actual trading.

## Backlog

- **Optional / low priority — condition editor responsiveness.** User accepted the current per-leaf query approach despite approximately 0.2–0.5 second delay. Potential future optimization: action-scoped preload and local positioning. Do not block current UI work.
- **Deferred, mandatory before operational trading — incomplete/invalid configuration in an active action.** User explicitly wants to permit incomplete draft conditions in the editor for now, leaving responsibility to the user during configuration. Screenshot on 2026-11-19 shows action OpenLong marked Active with a condition caption `Candle(?) H4 [0] > 0` (Candle source with CandleValue=None). This is a concrete risk, not cosmetic: current `RMarketConditionTable.calcValueW()` invokes `RMarketCandleTable.getValue(_candleValue)`; its default branch returns Close, including for None, so this incomplete condition can silently evaluate as Close > 0. No evidence of any actual order here.
  - Plan a strategy/action readiness check before start/activation (recursively check every applicable active-action root event, nested group and leaf; validate both operands; distinguish configuration errors from missing runtime market data). Do not confuse Active on an action with a validated/running strategy.
  - Check CandleValue=None or unsupported base (Range/Median also currently lack explicit supported cases), missing/unknown IndicatorId, Interval=None or unavailable interval, unsupported source types, invalid hierarchy, and other incomplete required parameters according to source. Do not validate hidden irrelevant stale fields. Define allowable shift semantics to avoid lookahead in evaluation.
  - At runtime, invalid/unavailable operands must not be silently substituted with Close or zero, treated as a valid false condition, or allow another OR branch to trigger an affected invalid action. Fail closed for the affected signal/action, log diagnostics with strategy/action/group/condition identity and cause; integration design must address safe handling of already-open positions separately.
  - Keep existing UI behavior for now. Do not introduce table-wide restrictions that break the shared Orders arithmetic configuration; design validation for each semantic context.
  - Add tests for an incomplete condition within an otherwise true OR expression and for legitimate zero/false values, so invalid is distinguished from an ordinary false/zero result.

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

### 2026-11-18 — A3.4.4.6 starts; performance improvement deferred

- User explicitly classified the 0.2–0.5-second per-leaf `executeQuery()` delay as **optional / backlog** and requested work to proceed. Do not block main A3 UI development on this optimization.
- Current next task: **A3.4.4.6 dynamic visibility of condition operand fields**, Actions editor only. `What` and `With` source independently control field visibility: Const -> Value; Indicator -> IndicatorId, Interval, Shift; Candle -> CandleValue, Interval, Shift; LosingTradesInARow/DepositPercentDrawdown -> Source only (calculation not implemented). Comparison operator remains visible.
- Implement visibility centrally in existing `RMarketStrategy.updateFormDesign()`, called on tree selection and on successful `What_Source.modified()` and `With_SecondSource.modified()`. For direct control references, source form metadata currently lacks `AutoDeclaration=Yes` on operand controls; ensure visibility-controlled controls are declared or resolved explicitly. Do not change Orders tab.
- **Proposed and not yet committed by user**; verify after implementation by switching source types on both sides, switching leaves/groups/actions, and reopening the form.

### 2026-11-19 — A3.4.4.6 completed; incomplete active action risk deferred

- User confirmed dynamic operand visibility works excellently; code committed in `c77a24ffddb868594318637cc2fea871d17f4cba` and merged into `df1fb61ebf19b31bf1a959f38f6c1952a8188c65`. Verified `updateFormDesign()` and the two source-control modified() handlers.
- User highlighted active OpenLong action with an incomplete candle operand (`Candle(?) H4 [0]`). Agreed to allow draft configuration for now and make handling invalid active-action conditions a pre-operational requirement; see Backlog. Source review confirmed `getValue(None)` falls through to Close, so the risk is real.
- Next task: A3.4.4.7 tree node deletion; clarify deletion policy before implementing. No further X++ changes made by assistant.

### 2026-11-19 — A3.4.4.7 deletion policy and implementation proposal

- User explicitly rejected cascading deletion entirely, not merely for this release. **Only nodes with no children may be deleted**, including empty root events and empty nested groups. This is an architectural decision, not a backlog item.
- Proposed `RMarketConditionGroup.hasChildren()` checks both direct child groups (by ParentRecId, intentionally regardless of action ownership for safety) and conditions (Relation=RMarketConditionGroup + RefRecId). `validateDelete()` and ordinary `delete()` guard nonempty groups. No cascading delete action.
- UI Delete enabled only with a selected valid leaf/empty group/event and no tree child (getChild). Database deletion uses selected node TableId/RecId, verifies action ownership, validates, deletes within transaction, updates editor and deletes only the single item from tree; select parent if available, otherwise an adjacent root. Do not reuse legacy `currentCondition_DS()` for the tree.
- No Num renumbering at this stage; gaps can be handled in shared sibling ordering task. Confirm deletion UX via a Yes/No confirmation; verify leaves, empty groups/events, and nonempty groups including collapsed branches.
- Proposal only: actual implementation and test are pending. Table checks apply to ordinary X++ delete paths, not raw SQL or deliberate doDelete/skip-method overrides; concurrent configuration editing and trading will need a separate consistency policy.

## How to use

- File path: `DEVELOPMENT_LOG.md` at repository root.
- Assistant may create/update it through the connected GitHub app. User can also edit via GitHub web UI, a local text editor, or Visual Studio.
- If this file is changed directly on GitHub, the local clone must incorporate the new commit before the next push (normally `git pull --rebase origin main`, after committing/stashing any local edits).
- This file is **not** an automatically recorded chat transcript. Irreplaceable ideas should be added as explicit decisions or questions when they arise.
