# AGENTS.md

## Project

This is a Unity 2D top-down turn-based tactics board game with Go-like liberties, capture rules, territory control, resource income, and level-based victory conditions.

The project language for design documents is Chinese. Code, class names, enum names, folder names, and public APIs should use clear English names.

## Core coding rules

- Use C# for Unity.
- Keep core game rules independent from Unity scene objects.
- Do not put capture, liberty, territory, turn, resource, or victory logic directly inside `MonoBehaviour` when it can be implemented as pure C#.
- Use `Vector2Int` for board coordinates.
- Use four-direction adjacency by default unless `Docs/Rules.md` has been modified to say otherwise.
- Separate Model, Rule/System, Action, Turn, Victory, AI, View, and Input code.
- Do not introduce third-party packages unless explicitly requested.
- Avoid hidden side effects in public methods.
- Prefer small classes with a single responsibility.
- Keep rule calculation deterministic.

## Suggested folder structure

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

## Naming guidance

Use these names unless a task explicitly changes them:

- `Faction`
- `TerrainType`
- `ResourceType`
- `PieceType`
- `TileOwner`
- `TileData`
- `PieceData`
- `GridModel`
- `PlayerState`
- `GameState`
- `BoardGroupService`
- `CaptureRuleService`
- `OwnershipService`
- `ResourceService`
- `TurnManager`
- `IVictoryCondition`
- `GameAction`
- `PlacePieceAction`
- `UpgradePieceAction`
- `EndTurnAction`

## Default implementation assumptions

When a rule is marked as `待定` in `Docs/Rules.md`, use the default recommendation written in that section for MVP implementation. Do not stop the task only because a rule is marked as `待定`, unless the current task specifically asks to resolve design questions.

## Validation

After code changes:

- Add or update Unity EditMode tests for core rule changes when practical.
- Keep the core model and rules testable without loading a Unity scene.
- When Unity cannot be run in the current environment, state which tests should be run in Unity Test Runner.
- Avoid relying on PlayMode tests for pure board-rule correctness.

## Done when

A task is considered complete when:

- Code compiles in Unity.
- Requested systems are implemented in the appropriate folders.
- Core game logic has EditMode tests when practical.
- Tilemap is used only for initialization/rendering, not as the source of truth for gameplay rules.
- The implementation follows `Docs/TechnicalSpec.md` and `Docs/Rules.md`.
