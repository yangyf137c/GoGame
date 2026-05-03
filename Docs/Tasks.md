# Tasks.md

## 1. 使用方式

本文件用于把项目拆成适合 Codex 执行的小任务。

建议每次只让 Codex 执行一个任务，并在任务完成后检查：

- 代码是否符合 `AGENTS.md`。
- 是否符合 `Docs/Rules.md`。
- 是否新增或更新了测试。
- 是否没有把规则写进 Tilemap 或 UI 代码里。

## 2. 任务 1：建立核心数据模型

### Goal

Implement the pure C# core data models for the board game.

### Context

Read:

- `AGENTS.md`
- `Docs/TechnicalSpec.md`
- `Docs/Rules.md`

### Requirements

- Create enums:
  - `Faction`
  - `TerrainType`
  - `ResourceType`
  - `PieceType`
  - `TileOwner`
  - `TurnPhase`
- Create models:
  - `TileData`
  - `PieceData`
  - `GridModel`
  - `ResourceWallet`
  - `ResourceCost`
  - `PlayerState`
  - `GameState`
- Use `Vector2Int` for coordinates.
- Core models must not depend on `MonoBehaviour` or Tilemap.
- Add helper methods for:
  - bounds checking,
  - tile lookup,
  - piece lookup,
  - four-direction neighbor lookup,
  - alive piece enumeration.

### Done when

- Code compiles in Unity.
- Add EditMode tests for `GridModel` bounds and neighbor lookup.
- Tests do not require a scene.

## 3. 任务 2：实现棋块与气计算

### Goal

Implement group and liberty calculation.

### Requirements

- Add `BoardGroupService`.
- Add `BoardGroup` result model.
- Find connected same-faction pieces using four-direction adjacency by default.
- Count liberties without duplicates.
- Empty map cells count as liberties.
- Occupied cells do not count as liberties.
- Map-outside cells do not count as liberties.

### Done when

Add tests for:

- Single piece has correct liberties.
- Edge piece has correct liberties.
- Two adjacent same-faction pieces form one group.
- Diagonal same-faction pieces do not form one group by default.
- Shared liberty is counted only once.

## 4. 任务 3：实现放置与提子

### Goal

Implement `PlacePieceAction` with capture resolution.

### Requirements

- Validate:
  - actor is current faction,
  - target tile exists,
  - target tile is placeable,
  - target tile has no piece,
  - player can afford the action cost.
- Apply placement.
- Resolve adjacent enemy captures first.
- Then check own group liberties.
- Reject pure suicide moves and rollback state.
- Recalculate ownership after successful placement.
- Return `ActionResult` for success/failure.
- Invalid action must not mutate `GameState`.

### Done when

Add tests for:

- Place on empty valid tile succeeds.
- Place on occupied tile fails.
- Place outside map fails.
- Single enemy piece with no liberties is captured.
- Multi-piece enemy group with no liberties is captured.
- Pure suicide placement fails.
- Capture-then-survive placement succeeds.
- Failed placement does not spend resources.
- Failed placement does not leave temporary pieces on board.

## 5. 任务 4：实现占地重算

### Goal

Implement ownership calculation.

### Requirements

- Add `OwnershipService`.
- Normal piece controls Manhattan radius 1.
- Upgraded piece controls Manhattan radius 2.
- If only one faction controls a tile, assign that owner.
- If both factions control a tile, mark `Contested`.
- If no faction controls a tile, mark `None`.
- Contested and None produce no resources.
- Ownership calculation must not affect liberties.

### Done when

Add tests for:

- Normal piece controls self and four neighboring tiles.
- Upgraded piece controls Manhattan radius 2.
- Opposing control creates `Contested`.
- No control creates `None`.
- Ownership recalculates after captured pieces are removed.

## 6. 任务 5：实现资源系统

### Goal

Implement resource wallet, income, and action cost calculation.

### Requirements

- Add `ResourceService`.
- Support at least `Food` and `Iron`.
- Resource amount cannot be negative.
- Income is gained at current faction's turn start.
- Only tiles owned by current faction produce income.
- `None` and `Contested` tiles do not produce income.
- Place piece cost uses default formula in `Docs/Rules.md`:
  - `FoodCost = BaseFoodCost + DistanceCostFactor * d`
  - `BaseFoodCost = 1`
  - `DistanceCostFactor = 1`
  - `d` is Manhattan distance to nearest friendly piece.

### Done when

Add tests for:

- Owned Food tile increases Food.
- Owned Iron tile increases Iron.
- Contested tile gives no income.
- None tile gives no income.
- Spending resources succeeds when affordable.
- Spending resources fails when not affordable.
- Failed spend does not reduce resources.
- Placement cost increases with distance.

## 7. 任务 6：实现回合流程

### Goal

Implement turn flow and action execution.

### Requirements

- Add `TurnManager`.
- Current faction starts as `Black`.
- On `StartTurn`, current faction gains income.
- A player may perform one or more actions.
- `EndTurnAction` switches faction.
- Full round count increases after White ends turn by default.
- Game cannot accept actions after `GameOver`.
- Invalid action returns failure and does not mutate state.

### Done when

Add tests for:

- Black acts first.
- Black end turn switches to White.
- White end turn switches to Black.
- Full round number increments after White ends turn.
- Start turn income is applied once.
- Invalid action does not advance phase unexpectedly.

## 8. 任务 7：实现传统占地胜利条件

### Goal

Implement `TraditionalTerritoryVictoryCondition`.

### Requirements

- Add `IVictoryCondition`.
- Add `VictoryResult`.
- At `MaxRound`, count owned tiles.
- `Black` owned tiles count for Black.
- `White` owned tiles count for White.
- `None` and `Contested` are ignored.
- More owned tiles wins.
- Tie uses default rule in `Docs/Rules.md`: draw.

### Done when

Add tests for:

- Game does not end before `MaxRound`.
- Black wins with more owned tiles.
- White wins with more owned tiles.
- Tie returns draw result.
- Contested tiles are ignored.

## 9. 任务 8：建立关卡配置 ScriptableObject

### Goal

Implement level configuration assets.

### Requirements

- Add `LevelDefinition` ScriptableObject.
- Add serializable data classes:
  - `InitialPieceData`
  - `InitialResourceData`
  - `FlagPointData`
- Add `VictoryConditionType` enum.
- Add `AIProfile` placeholder or simple ScriptableObject.
- Provide a `LevelInitializer` that creates `GameState` from `LevelDefinition` and `GridModel`.

### Done when

- Code compiles.
- A designer can create a level asset in Unity Inspector.
- Initial pieces and initial resources are applied to `GameState`.

## 10. 任务 9：连接 Unity Tilemap 加载

### Goal

Implement Unity Tilemap loader and convert scene tiles to `GridModel`.

### Requirements

- Add `TilemapBoardLoader` under `Assets/Scripts/Unity/Level`.
- Read `TerrainTilemap`.
- Optionally read `ResourceTilemap`.
- Optionally read `MarkerTilemap`.
- Convert non-empty terrain cells into `TileData`.
- Map Unity tiles to logical terrain/resource data using a configuration asset.
- Do not store gameplay state in Tilemap.
- Return a populated `GridModel`.

### Done when

- A test scene can load a board.
- The loaded board has correct tile count.
- Tile data includes terrain and resource info.
- Core rules still use `GridModel`, not Tilemap.

## 11. 任务 10：实现棋盘和棋子显示

### Goal

Implement basic board view and piece view.

### Requirements

- Add `BoardView`.
- Add `PieceView` or `PieceViewFactory`.
- Show pieces based on `PieceData`.
- Refresh owner overlay based on `TileData.Owner`.
- Provide selected tile highlight.
- View code must not calculate captures or resources.

### Done when

- Placing a piece updates its visual representation.
- Captured pieces disappear visually.
- Owner overlay updates after ownership recalculation.

## 12. 任务 11：实现玩家输入与基础 UI

### Goal

Implement player input and minimum playable UI.

### Requirements

- Add `BoardInputController`.
- Convert mouse click to grid coordinate.
- On clicking empty tile, create `PlacePieceAction` for current faction.
- Add End Turn button.
- Add HUD showing:
  - current faction,
  - full round number,
  - Black resources,
  - White resources,
  - selected tile info,
  - error message.

### Done when

- Player can place pieces by clicking.
- Player can end turn.
- Invalid click shows error but does not mutate state.
- HUD updates after actions and turns.

## 13. 任务 12：实现简单白方 AI

### Goal

Implement a simple AI for White.

### Requirements

- Add `IPlayerAgent`.
- Add `HumanPlayerAgent` placeholder.
- Add `RandomAIPlayerAgent`.
- AI enumerates legal placement actions.
- AI chooses one legal action randomly.
- If no legal action exists, AI ends turn.
- AI must use `TurnManager` and `GameAction`, not bypass rules.

### Done when

- White can act automatically.
- AI does not make illegal moves.
- AI ends turn if no legal placement exists.

## 14. 任务 13：整合 MVP 对局流程

### Goal

Integrate map loading, level initialization, player input, AI, turn flow, and victory result.

### Requirements

- Add one test scene.
- Add one test level asset.
- Initialize map and game state on scene start.
- Start with Black player turn.
- Allow Black player input.
- Allow White AI turn.
- Show game over result.

### Done when

- The MVP can be played from start to game over.
- Traditional territory victory condition triggers at `MaxRound`.
- No core rules are implemented in UI or Tilemap components.

## 15. 任务 14：后续扩展强化棋子

### Goal

Implement upgraded pieces.

### Requirements

- Add `UpgradePieceAction`.
- Only friendly normal pieces can be upgraded.
- Cost follows `Docs/Rules.md` default unless modified.
- Upgraded piece has `ControlRadius = 2`.
- Upgrading recalculates ownership.
- Upgrading does not change liberties by default.

### Done when

- Upgrade action succeeds with enough resources.
- Upgrade action fails without enough resources.
- Upgraded control radius affects ownership.
- Upgrading does not affect group liberty calculation.

## 16. 任务 15：后续扩展夺旗模式

### Goal

Implement capture flag victory condition.

### Requirements

- Add `CaptureFlagVictoryCondition`.
- Track hold turns per flag and faction.
- Owner must be exactly `Black` or `White` to count.
- `None` and `Contested` reset hold progress by default.
- Reaching required hold turns wins.
- Timeout behavior follows `Docs/Rules.md`.

### Done when

- Holding a flag for required turns wins.
- Contested flag resets progress.
- Enemy taking the flag resets previous faction progress.
- Timeout behavior works as specified.

## 17. Codex 提示词模板

复制以下模板并替换任务名称：

```md
Goal:
请执行 Docs/Tasks.md 中的「任务 X：……」。

Context:
请先阅读 AGENTS.md、Docs/TechnicalSpec.md、Docs/Rules.md，以及 Docs/Tasks.md 中对应任务。

Constraints:
- 不要把核心规则写进 MonoBehaviour。
- Tilemap 只能作为初始化和显示层。
- 核心规则需要能用 EditMode Test 验证。
- 若 Rules.md 中有待定规则，MVP 按默认建议实现。

Done when:
- 满足任务中的 Done when。
- 说明新增/修改了哪些文件。
- 说明应在 Unity Test Runner 中运行哪些测试。
```
