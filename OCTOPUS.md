# 🐙 八爪鱼架构 (Octopus Architecture)

**多 Agent 协作架构规范**

![Octopus Logo](./logo.png)

---

## 概述

八爪鱼架构是一个**集中决策、分布执行**的多 Agent 协作架构。

**团队 Emoji**: 🐙  
**团队名称**: ZedComp

---

## 核心概念

```
🧠 主脑 (Brain) → 唯一决策中心
    │
    └── 🦑 触手 (Tentacle) → 专业执行者
            │
            └── 🔵 吸盘 (Sucker) → 任务执行单元（最末层级）
```

---

## 文档索引

| 文档 | 说明 |
|------|------|
| [README.md](./README.md) | 📘 快速入门 |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 📐 架构总览 |
| [COMMUNICATION_PROTOCOL.md](./COMMUNICATION_PROTOCOL.md) | 📡 通讯协议 |
| [TENTACLE_ROLES.md](./TENTACLE_ROLES.md) | 🦑 触手角色 |
| [TASK_MANAGEMENT.md](./TASK_MANAGEMENT.md) | 📋 任务管理 |
| [FAILURE_HANDLING.md](./FAILURE_HANDLING.md) | ⚠️ 失效处理 |
| [HEARTBEAT_SPEC.md](./HEARTBEAT_SPEC.md) | 💓 心跳机制 |
| [STATE_PERSISTENCE.md](./STATE_PERSISTENCE.md) | 💾 状态持久化 |

---

## 核心特点

✅ **集中决策** — 主脑是唯一决策中心  
✅ **分布执行** — 触手各自独立工作空间  
✅ **单任务约束** — 触手同时只处理一个任务  
✅ **弹性扩展** — 触手数量不固定，可动态增删  
✅ **透明通讯** — 所有通讯同步到外部 Channel  
✅ **失效安全** — 主脑失效时触手暂停待命  

---

## 版本

**当前版本**: v1.0  
**生效日期**: 2026-03-10  
**维护者**: ZedComp 团队

---

## License

MIT License
