# Unity Go-like Tactics Docs

这是一套面向 Codex 使用的 Unity 回合制战棋项目文档。

建议把以下文件复制到 Unity 项目根目录：

```text
AGENTS.md
Docs/
  GameDesign.md
  TechnicalSpec.md
  Rules.md
  Tasks.md
  OpenQuestions.md
```

## 阅读顺序

1. `AGENTS.md`：给 Codex 的项目级工作约定。
2. `Docs/GameDesign.md`：玩法设计总览。
3. `Docs/Rules.md`：核心规则规格。
4. `Docs/TechnicalSpec.md`：技术架构和类设计。
5. `Docs/Tasks.md`：分阶段开发任务。
6. `Docs/OpenQuestions.md`：待确认的规则歧义。

## 推荐使用方式

第一次让 Codex 开始开发时，可以使用：

```md
请阅读 AGENTS.md、Docs/GameDesign.md、Docs/Rules.md、Docs/TechnicalSpec.md 和 Docs/Tasks.md。
然后执行 Docs/Tasks.md 中的「任务 1：建立核心数据模型」。
若 Rules.md 中存在待定规则，MVP 按默认建议实现。
```

后续每次只推进一个任务，降低返工风险。
