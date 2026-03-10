# 🐙 八爪鱼架构 (Octopus Architecture)

**多 Agent 协作架构规范**

![Octopus Logo](./logo.png)

**团队 Emoji**: 🐙  
**Logo 位置**: 
- 本地：`docs/octopus-architecture/logo.png`
- Nextcloud: `botshare/temp/logo-像素.png`

---

## 快速开始

八爪鱼架构是一个**集中决策、分布执行**的多 Agent 协作架构。

### 核心概念

```
🧠 主脑 (Brain) → 唯一决策中心
    │
    └── 🦑 触手 (Tentacle) → 专业执行者
            │
            └── 🔵 吸盘 (Sucker) → 任务执行单元（最末层级）
```

### 核心规则

| 规则 | 说明 |
|------|------|
| **主脑** | 唯一决策中心，不可接管，有心跳机制 |
| **触手** | 同时只能处理一个任务，可动态增删 |
| **吸盘** | 最末层级，不允许再派生 |
| **通讯** | 所有主脑↔触手通讯同步到外部 Channel |
| **失效** | 主脑离线时触手暂停待命，状态持久化 |

---

## 文档索引

| 文档 | 说明 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 📘 架构总览（必读） |
| [COMMUNICATION_PROTOCOL.md](./COMMUNICATION_PROTOCOL.md) | 📡 通讯协议规范 |
| [TENTACLE_ROLES.md](./TENTACLE_ROLES.md) | 🦑 触手角色定义 |
| [TASK_MANAGEMENT.md](./TASK_MANAGEMENT.md) | 📋 任务管理规范 |
| [FAILURE_HANDLING.md](./FAILURE_HANDLING.md) | ⚠️ 失效处理机制 |
| [HEARTBEAT_SPEC.md](./HEARTBEAT_SPEC.md) | 💓 心跳机制规范 |
| [STATE_PERSISTENCE.md](./STATE_PERSISTENCE.md) | 💾 状态持久化规范 |

---

## 当前实例

### ZedComp

- **主脑**: Zed (`main`)
- **触手**: 6 个 (Geoff, Eva, Leo, Anna, Mia, Kia)
- **激活日期**: 2026-03-10
- **通讯渠道**: 外部 Channel 群聊

---

## 核心特点

✅ **集中决策** — 主脑是唯一决策中心  
✅ **分布执行** — 触手各自独立工作空间  
✅ **单任务约束** — 触手同时只处理一个任务  
✅ **弹性扩展** — 触手数量不固定，可动态增删  
✅ **透明通讯** — 所有通讯同步到外部 Channel  
✅ **失效安全** — 主脑失效时触手暂停待命  

---

## 快速参考

### 触手状态

```
空闲 (idle) → 接收任务 → 执行中 (busy) → 完成任务 → 空闲 (idle)
                    ↓
              主脑离线 → 暂停 (paused) → 主脑上线 → 恢复
```

### 心跳参数

| 参数 | 值 |
|------|-----|
| 心跳间隔 | 30 秒 |
| 超时阈值 | 90 秒 |

### 状态持久化位置

```
/{tentacle-workspace}/tentacle-state/
├── current-task.json
├── progress.json
└── context.json
```

---

**版本**: v1.0  
**生效日期**: 2026-03-10  
**维护者**: ZedComp 团队
