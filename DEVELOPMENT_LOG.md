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
| A3.3 tests | Test class present | `RMarketJobTest_ConditionTree` tests nested logic and validation; test method reports 10 checks. Test outcomes were reported in prior work, not independently rerun by assistant |
| A3.4.1–A3.4.4.3 | Implemented | Action tree on `RMarketStrategy`, selection/editor, creating events/groups/conditions |
| A3.4.4.4 | Implemented in `a0b60e7` | Central caption formatter and `updateSelectedConditionTreeNode()`, `modified()` handlers for group and condition fields |

### Current UI code state, verified from GitHub 2026-11-18

- `RMarketStrategy.conditionTreeNodeCaption(Common)` formats group names as root/event or nested group, including AND/OR; condition captions currently show only `● Condition N`.
- `RMarketStrategy.updateSelectedConditionTreeNode(Common)` checks the selected tree item's table/record identity and updates its text through `FormTreeControl.setItem()`.
- `modified()` controls cover group name/operator, and condition source, indicator, candle value, value, interval, shift, comparison, and mirrored right-side fields.
- `RMarketStrategy.updateFormDesign()` currently toggles the group vs leaf editor, and disables deletion and Up/Down. Conditional visibility of individual What/With fields is not yet implemented.
- `RMarketStrategy.addConditionTreeNodes()` currently enumerates nested groups and leaves in **separate loops**, so mixed children do not yet display in a single global `Num` order.
- Latest committed `BuildProjectResult.xml` shows no compile errors and three existing warnings about external assemblies, generated 2026-11-18 00:26 MSK. It is not a substitute for functional form testing.

## Next work (ordered, incremental)

1. **A3.4.4.5 — Human-readable condition captions.** Replace `● Condition N` with a compact, deterministic English expression combining left operand, comparison sign and right operand. Reuse one formatter for initial tree construction and live field edits. Cover Const, Candle, Indicator first; unsupported/unconfigured sources should be visibly explicit, not silently presented as valid.
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
- User implemented updating captions when fields of groups/conditions change. Code review found correct targeted-update structure; condition text still placeholder, per-source field visibility pending.
- Decision: keep this Markdown file in GitHub as a durable **decision and task log**, maintained by assistant when actively working on the project, in addition to ordinary git commits. No autonomous/background updates are implied.
- Protocol: whenever a task concludes, record the outcome, technical decisions and exact next task here. Clearly mark `implemented` vs `tested` vs `planned`. Do not overwrite historical decisions without noting why they changed.

## How to use

- File path: `DEVELOPMENT_LOG.md` at repository root.
- Assistant may create/update it through the connected GitHub app. User can also edit via GitHub web UI, a local text editor, or Visual Studio.
- If this file is changed directly on GitHub, the local clone must incorporate the new commit before the next push (normally `git pull --rebase origin main`, after committing/stashing any local edits).
- This file is **not** an automatically recorded chat transcript. Irreplaceable ideas should be added as explicit decisions or questions when they arise.
