# Claude Code 源码解读系列专栏

深入解析 Anthropic Claude Code CLI 的核心架构与实现原理

---

## 专栏大纲

### 第一部分：基础入门

| 章节 | 文章 | 文件 | 状态 |
|------|------|------|------|
| 第一章 | [项目概述与背景](./01-introduction.md) | - | ✅ |
| 第二章 | [项目架构总览](./02-architecture.md) | - | ✅ |

### 第二部分：核心机制

| 章节 | 文章 | 文件 | 状态 |
|------|------|------|------|
| 第三章 | [入口与启动流程](./03-entrypoint.md) | cli.tsx, main.tsx | ✅ |
| 第四章 | [核心查询引擎](./04-query-engine.md) | query.ts | ✅ |
| 第五章 | [会话引擎](./05-query-engine-class.md) | QueryEngine.ts | ✅ |
| 第六章 | [工具系统](./06-tool-system.md) | Tool.ts, tools.ts | ✅ |
| 第七章 | [权限系统](./07-permissions.md) | permissions.ts | ✅ |
| 第八章 | [API 层与多 Provider](./08-api-layer.md) | claude.ts | ✅ |

### 第三部分：基础设施

| 章节 | 文章 | 文件 | 状态 |
|------|------|------|------|
| 第九章 | [Ink 终端 UI 系统](./09-ink-ui.md) | ink.ts, ink/* | ✅ |
| 第十章 | [状态管理](./10-state-management.md) | AppState.tsx, store.ts | ✅ |
| 第十一章 | [斜杠命令系统](./11-commands.md) | commands.ts, commands/* | ✅ |
| 第十二章 | [高级特性](./12-advanced-features.md) | MCP/压缩/定时任务 | ✅ |

---

## 阅读建议

### 按架构层次从浅入深

```
第一章 → 第二章 → 第三章 → 第四章/第五章 → 第六章 → ...
(项目背景)  (架构)   (启动)   (核心循环)    (工具)
```

### 按模块独立阅读

如果你对某个特定模块感兴趣，可以直接跳转到对应章节：

- 对 **UI 交互** 感兴趣 → 第九章 (Ink) → 第十章 (状态)
- 对 **工具开发** 感兴趣 → 第六章 (工具系统) → 第三章 (入口)
- 对 **API 集成** 感兴趣 → 第八章 (API 层)

---

## 前置知识

- TypeScript / JavaScript
- React (组件化思想)
- CLI 应用开发
- API 调用 (REST, Streaming)
- 基本的设计模式 (Factory, Observer, State Machine)
