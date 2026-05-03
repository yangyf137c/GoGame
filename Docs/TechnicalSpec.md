# TechnicalSpec.md

## 1. 技术目标

本文件描述 Unity 项目的建议技术架构，使 Codex 能按照稳定、可测试、可扩展的方式实现游戏。

核心原则：

- 逻辑层与表现层分离。
- 规则计算使用纯 C# 类。
- Tilemap 只作为地图初始数据来源和显示层。
- 核心规则可通过 Unity EditMode Test 验证。
- 每个系统职责清晰，避免把所有逻辑集中在一个 `GameManager` 中。

## 2. Unity 版本

- Unity 版本：待项目实际创建后填写。
- 推荐：Unity 6 LTS 或当前项目实际使用版本。

```text
[待填写]
Unity Version: 
```

## 3. 坐标系统

- 逻辑坐标使用 `Vector2Int`。
- 默认使用二维正方形网格。
- 推荐约定：x 向右递增，y 向上递增。
- 地图外坐标视为无效，不计为气，不可放置棋子。

## 4. 目录结构

建议目录：

```text
Assets/
  Scripts/
    Core/
      Models/
      Rules/
      Actions/
      Turn/
      Victory/
      Resources/
    AI/
    Unity/
      Level/
      Views/
      Input/
      UI/
  Tests/
    EditMode/
```

职责说明：

- `Core/Models`：纯数据模型。
- `Core/Rules`：气、提子、占地等规则计算。
- `Core/Actions`：玩家操作命令对象。
- `Core/Turn`：回合推进。
- `Core/Victory`：胜负条件。
- `Core/Resources`：资源计算和消耗。
- `AI`：电脑玩家决策。
- `Unity/Level`：Tilemap 加载、关卡配置、ScriptableObject。
- `Unity/Views`：棋盘、棋子、归属显示。
- `Unity/Input`：鼠标点击、格子选择。
- `Unity/UI`：资源、回合、提示、胜负界面。
- `Tests/EditMode`：纯逻辑测试。

## 5. 核心数据模型

### 5.1 Faction

```csharp
public enum Faction
{
    None,
    Black,
    White
}
```

说明：

- `None` 用于无阵营场景。
- 地图归属不要直接使用 `Faction`，建议使用 `TileOwner`，因为存在 `Contested`。

### 5.2 TileOwner

```csharp
public enum TileOwner
{
    None,
    Black,
    White,
    Contested
}
```

### 5.3 TerrainType

```csharp
public enum TerrainType
{
    Plain,
    River,
    Mountain,
    Forest,
    Flag
}
```

MVP 只要求实现：

- `Plain`
- `Mountain`

### 5.4 ResourceType

```csharp
public enum ResourceType
{
    Food,
    Iron
}
```

### 5.5 PieceType

```csharp
public enum PieceType
{
    Normal,
    Upgraded
}
```

MVP 只要求实现 `Normal`。

### 5.6 TileData

建议字段：

```csharp
public sealed class TileData
{
    public Vector2Int Position { get; }
    public TerrainType TerrainType { get; set; }
    public ResourceType? YieldType { get; set; }
    public int YieldAmount { get; set; }
    public TileOwner Owner { get; set; }
    public int? OccupyingPieceId { get; set; }
    public bool IsPlaceable { get; set; }
    public int MoveCost { get; set; }
}
```

规则：

- `YieldAmount` 必须大于或等于 0。
- `YieldType == null` 或 `YieldAmount == 0` 表示无产出。
- `OccupyingPieceId == null` 表示无棋子。
- `Owner` 由 `OwnershipService` 计算，不建议手动修改。

### 5.7 PieceData

建议字段：

```csharp
public sealed class PieceData
{
    public int Id { get; }
    public Faction Faction { get; set; }
    public PieceType PieceType { get; set; }
    public Vector2Int Position { get; set; }
    public bool IsAlive { get; set; }

    public int ControlRadius => PieceType == PieceType.Upgraded ? 2 : 1;
}
```

### 5.8 GridModel

建议职责：

- 存储所有 `TileData`。
- 存储所有 `PieceData`。
- 提供坐标查询。
- 提供四方向邻居查询。
- 提供放置、移除、移动棋子的基础操作。

建议方法：

```csharp
public sealed class GridModel
{
    public bool ContainsPosition(Vector2Int position);
    public TileData? GetTile(Vector2Int position);
    public PieceData? GetPieceAt(Vector2Int position);
    public IEnumerable<Vector2Int> GetFourNeighbors(Vector2Int position);
    public IEnumerable<TileData> GetAllTiles();
    public IEnumerable<PieceData> GetAlivePieces();
    public bool IsEmpty(Vector2Int position);
}
```

### 5.9 ResourceWallet

建议使用字典支持扩展资源类型：

```csharp
public sealed class ResourceWallet
{
    public IReadOnlyDictionary<ResourceType, int> Amounts { get; }
    public int Get(ResourceType type);
    public bool CanAfford(ResourceCost cost);
    public void Add(ResourceType type, int amount);
    public bool TrySpend(ResourceCost cost);
}
```

### 5.10 ResourceCost

```csharp
public sealed class ResourceCost
{
    public Dictionary<ResourceType, int> Amounts { get; }
}
```

### 5.11 PlayerState

```csharp
public sealed class PlayerState
{
    public Faction Faction { get; }
    public ResourceWallet Resources { get; }
}
```

### 5.12 GameState

建议字段：

```csharp
public sealed class GameState
{
    public GridModel Grid { get; }
    public Dictionary<Faction, PlayerState> Players { get; }
    public Faction CurrentFaction { get; set; }
    public int FullRoundNumber { get; set; }
    public TurnPhase CurrentPhase { get; set; }
    public bool IsGameOver { get; set; }
    public VictoryResult? VictoryResult { get; set; }
}
```

默认：

- `CurrentFaction = Faction.Black`
- `FullRoundNumber = 1`

## 6. 规则服务

### 6.1 BoardGroupService

职责：

- 查找某个棋子所在棋块。
- 查找某个阵营的所有棋块。
- 计算棋块气数。
- 返回棋块的气坐标集合。

建议方法：

```csharp
public sealed class BoardGroupService
{
    public BoardGroup FindGroup(GridModel grid, Vector2Int startPosition);
    public IReadOnlySet<Vector2Int> FindLiberties(GridModel grid, BoardGroup group);
    public int CountLiberties(GridModel grid, BoardGroup group);
}
```

### 6.2 CaptureRuleService

职责：

- 在落子后检查相邻敌方棋块。
- 移除无气敌方棋块。
- 检查己方是否自杀。
- 提供可回滚的落子结算。

### 6.3 OwnershipService

职责：

- 根据棋子控制半径重新计算所有地图块归属。
- 处理双方争夺同一格的情况。
- 产出结果写入 `TileData.Owner`。

### 6.4 ResourceService

职责：

- 计算当前阵营回合开始收入。
- 给当前阵营增加资源。
- 计算操作费用。
- 执行资源扣除。

### 6.5 TurnManager

职责：

- 管理回合阶段。
- 调用回合开始资源结算。
- 接收并执行 `GameAction`。
- 切换行动方。
- 调用胜负条件判断。

## 7. 行动系统

所有玩家操作统一使用 `GameAction`。

建议接口：

```csharp
public interface IGameAction
{
    Faction Actor { get; }
    ActionResult Validate(GameState state);
    ResourceCost GetCost(GameState state);
    ActionResult Apply(GameState state);
}
```

### 7.1 ActionResult

```csharp
public sealed class ActionResult
{
    public bool Success { get; }
    public string ErrorMessage { get; }
}
```

### 7.2 PlacePieceAction

字段：

- `Faction Actor`
- `Vector2Int TargetPosition`
- `PieceType PieceType`

职责：

- 检查目标格合法。
- 计算费用。
- 扣除资源。
- 放置棋子。
- 执行提子结算。
- 重新计算地图归属。

### 7.3 UpgradePieceAction

字段：

- `Faction Actor`
- `int PieceId`

职责：

- 检查目标棋子是否属于当前行动方。
- 检查资源是否足够。
- 将普通棋子升级为强化棋子。
- 重新计算地图归属。

MVP 可暂不实现。

### 7.4 EndTurnAction

职责：

- 结束当前行动方回合。
- 触发胜负判断。
- 未结束时切换行动方。

## 8. 回合状态机

```csharp
public enum TurnPhase
{
    StartTurn,
    AwaitingAction,
    ResolvingAction,
    EndTurn,
    GameOver
}
```

流程见 `Docs/Rules.md`。

## 9. 胜负条件系统

建议接口：

```csharp
public interface IVictoryCondition
{
    void Initialize(GameState state, LevelDefinition level);
    VictoryResult Evaluate(GameState state, TurnContext context);
}
```

### 9.1 VictoryResult

```csharp
public sealed class VictoryResult
{
    public bool IsGameOver { get; }
    public Faction? Winner { get; }
    public string Reason { get; }
}
```

### 9.2 TraditionalTerritoryVictoryCondition

- 达到最大完整回合数时结算。
- 统计黑白双方拥有的地图块数量。
- 数量多者胜。

### 9.3 CaptureFlagVictoryCondition

- 追踪旗帜点连续占有回合数。
- 满足指定连续占有回合数时胜利。
- MVP 可暂不实现完整逻辑。

## 10. 关卡配置

建议使用 ScriptableObject。

### 10.1 LevelDefinition

字段建议：

```csharp
public sealed class LevelDefinition : ScriptableObject
{
    public string LevelId;
    public string DisplayName;
    public string MapSceneName;
    public VictoryConditionType VictoryType;
    public int MaxRound;
    public List<InitialPieceData> InitialPieces;
    public List<InitialResourceData> InitialResources;
    public List<FlagPointData> FlagPoints;
    public AIProfile WhiteAIProfile;
}
```

### 10.2 InitialPieceData

```csharp
[Serializable]
public sealed class InitialPieceData
{
    public Faction Faction;
    public PieceType PieceType;
    public Vector2Int Position;
}
```

### 10.3 InitialResourceData

```csharp
[Serializable]
public sealed class InitialResourceData
{
    public Faction Faction;
    public ResourceType ResourceType;
    public int Amount;
}
```

### 10.4 FlagPointData

```csharp
[Serializable]
public sealed class FlagPointData
{
    public Vector2Int Position;
    public int RequiredHoldTurns;
    public Faction TargetFaction;
}
```

## 11. Tilemap 加载

地图场景建议包含多个 Tilemap：

- `TerrainTilemap`：存储地形。
- `ResourceTilemap`：存储资源产出信息，可选。
- `MarkerTilemap`：存储初始点、旗帜点、不可放置点等关卡标记，可选。
- `OwnerOverlayTilemap`：运行时显示归属，可选。

加载流程：

1. 遍历 `TerrainTilemap.cellBounds`。
2. 对每个非空 Tile 生成 `TileData`。
3. 根据 Tile 类型映射 `TerrainType`。
4. 根据 `ResourceTilemap` 中同坐标 Tile 映射 `YieldType` 和 `YieldAmount`。
5. 根据 `MarkerTilemap` 读取初始点、旗帜点、不可放置点等信息。
6. 生成 `GridModel`。
7. 后续规则系统只读取 `GridModel`，不直接查询 Tilemap。

## 12. Tile 映射配置

建议创建 `TileDefinition` ScriptableObject 或配置表，用于把 Unity Tile 映射为逻辑数据。

字段建议：

- `TileBase Tile`
- `TerrainType TerrainType`
- `bool IsPlaceable`
- `ResourceType? YieldType`
- `int YieldAmount`

## 13. 表现层

### 13.1 BoardView

职责：

- 根据 `GridModel` 刷新棋盘显示。
- 显示地图块归属覆盖层。
- 显示选中格高亮。

不得实现：

- 气计算。
- 提子规则。
- 资源结算。
- 胜负判断。

### 13.2 PieceView

职责：

- 生成、移动或删除棋子显示对象。
- 根据 `PieceData` 刷新 Sprite。

### 13.3 GameHudView

职责：

- 显示当前方。
- 显示资源。
- 显示回合数。
- 显示选中格信息。
- 显示错误提示。

## 14. 输入层

### 14.1 BoardInputController

职责：

- 处理鼠标点击。
- 把屏幕坐标转换为 Tilemap cell 坐标。
- 生成对应的 `GameAction` 请求。
- 把请求交给 `TurnManager`。

## 15. AI 系统

### 15.1 IPlayerAgent

```csharp
public interface IPlayerAgent
{
    Faction Faction { get; }
    IGameAction DecideAction(GameState state);
}
```

### 15.2 HumanPlayerAgent

- 等待玩家 UI 输入。

### 15.3 RandomAIPlayerAgent

- 获取所有合法操作。
- 随机选择一个操作。
- 若无合法操作，则结束回合。

### 15.4 GreedyAIPlayerAgent

- 模拟候选落子。
- 选择能提升最多占地数量的行动。
- 可作为第二阶段实现。

## 16. 测试策略

核心规则必须优先使用 EditMode Test。

建议测试文件：

```text
Assets/Tests/EditMode/
  GridModelTests.cs
  BoardGroupTests.cs
  CaptureRuleTests.cs
  OwnershipTests.cs
  ResourceTests.cs
  TurnFlowTests.cs
  VictoryConditionTests.cs
```

测试原则：

- 不依赖真实 Tilemap 场景测试气和提子。
- 使用小型手工构造地图验证规则。
- 对非法操作测试状态不变。
- 对资源扣除和回合推进测试精确数值。

## 17. 不建议的实现方式

避免：

- 在 Tilemap 上直接保存所有规则状态。
- 在一个 `GameManager` 中实现全部逻辑。
- 用 GameObject 查找邻接棋子。
- 把资源、占地、提子混在 UI 点击事件中。
- 没有测试就实现复杂 AI。

## 18. 推荐开发顺序

1. 核心数据模型。
2. 棋块和气计算。
3. 放置与提子。
4. 占地重算。
5. 资源系统。
6. 回合系统。
7. 胜负条件。
8. Tilemap 加载。
9. 表现层和输入。
10. 简单 AI。
11. UI。
