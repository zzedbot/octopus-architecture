# 📡 通讯协议 (Communication Protocol)

**版本**: v1.0  
**生效日期**: 2026-03-10

---

## 一、概述

本规范定义八爪鱼架构中主脑、触手、吸盘之间的通讯方式，确保信息传递透明、可追溯。

---

## 二、通讯层级

| 层级 | 通讯方式 | 场景 | 同步要求 |
|------|----------|------|----------|
| **主脑 ↔ 触手** | `sessions_send` | 即时对话、任务分配、进度汇报 | ✅ 通过外部 Webhook 同步 |
| **触手 → 吸盘** | `sessions_spawn` | 分配独立后台任务 | ✅ 完成时同步 |
| **主脑 → 主人** | Channel 插件 | 结果汇报、告警通知 | - |
| **主人 → 主脑** | Channel 插件 | 命令下发 | - |

---

## 三、外部 Channel 消息同步规则

### 3.1 核心原则

**所有主脑↔触手的 `sessions_send` 通讯，必须同步到外部群聊中**，让主人知道我们之间沟通了什么。

### 3.2 同步方式：方案 A

在 `sessions_send` **前后**，分别调用 Webhook 发送消息。

### 3.3 实现流程

```
发送者 (如 Eva) 准备发送消息给接收者 (如 Zed)

步骤 1: 发送"开始"消息
  ↓
  调用 Eva 的外部 Webhook → 外部群聊
  消息内容："[Eva → Zed] 发送消息：任务内容..."
  ↓
步骤 2: 执行 sessions_send
  ↓
  sessions_send(sessionKey: "agent:zed:...", message: "任务内容")
  ↓
步骤 3: 收到回复后，发送"完成"消息（可选）
  ↓
  调用 Eva 的外部 Webhook → 外部群聊
  消息内容："[Eva ← Zed] 收到回复：回复内容..."
```

### 3.4 配置状态

✅ **每个触手已配置外部 channel**，使用自己的 Webhook 地址发送消息。

---

## 四、消息格式规范

### 4.1 基本格式

```
[{发送者} → {接收者}] {消息内容}
```

### 4.2 消息类型

| 类型 | 格式 | 示例 |
|------|------|------|
| **任务分配** | `[{发送者} → {接收者}] 任务：{任务描述}` | `[Zed → Eva] 任务：爬取文档 V3 章节` |
| **任务接收** | `[{发送者} → {接收者}] 收到：{任务描述}` | `[Eva → Zed] 收到：爬取文档 V3 章节` |
| **进度汇报** | `[{发送者} → {接收者}] 进度：{描述} ({百分比}%)` | `[Eva → Zed] 进度：爬取文档 (60%)` |
| **任务完成** | `[{发送者} → {接收者}] ✅ 完成：{结果}` | `[Eva → Zed] ✅ 完成：获取 45 篇文档` |
| **任务失败** | `[{发送者} → {接收者}] ❌ 失败：{原因}` | `[Leo → Zed] ❌ 失败：依赖冲突` |
| **心跳** | `[💓 HEARTBEAT] {主脑} is alive` | `[💓 HEARTBEAT] Zed is alive` |
| **告警** | `[⚠️ ALERT] {告警内容}` | `[⚠️ ALERT] 主脑离线，已暂停待命` |

---

## 五、心跳机制

### 5.1 心跳流程

```
主脑 (Zed)
    ↓ 每 30 秒发送心跳
    │ "[💓 HEARTBEAT] Zed is alive"
    ↓
外部 Channel (广播)
    ↓
触手们接收
    │
    ├─ 收到心跳 → 重置超时计时器
    │
    └─ 超时 90 秒未收到 → 触发失效处理
```

### 5.2 心跳参数

| 参数 | 值 | 说明 |
|------|-----|------|
| **心跳间隔** | 30 秒 | 主脑发送心跳的频率 |
| **超时阈值** | 90 秒 | 触手判定主脑离线的时长 |
| **重试次数** | 3 次 | 主脑重新上线后的状态同步重试 |

### 5.3 心跳消息格式

**主脑发送**:
```
[💓 HEARTBEAT] Zed is alive | timestamp: 2026-03-10T15:00:00+08:00 | active_tentacles: 6 | uptime: 1h
```

**触手响应（可选）**:
```
[👋 ACK] Geoff received | status: busy | task: 项目进度管理 | progress: 60%
```

---

## 六、sessions_send 超时处理

### 6.1 超时说明

`sessions_send` 返回 `timeout` 是**预期行为**——工具只等待 `timeoutSeconds` 指定的时间，但底层 run 可能仍在执行。

### 6.2 处理方案

| 方案 | 适用场景 | timeoutSeconds |
|------|----------|----------------|
| **Fire-and-forget** | 后台任务，不需要即时回复 | `0` (不等待) |
| **增加等待时间** | 需要即时交互的对话 | `30-120` 秒 |

### 6.3 使用示例

**方案 1：Fire-and-forget（后台任务）**
```json
{
  "tool": "sessions_send",
  "sessionKey": "agent:eva:...",
  "message": "任务内容",
  "timeoutSeconds": 0
}
```

**方案 2：增加等待时间（即时对话）**
```json
{
  "tool": "sessions_send",
  "sessionKey": "agent:eva:...",
  "message": "任务内容",
  "timeoutSeconds": 120
}
```

---

## 七、子任务派生规范

### 7.1 派生前知会

触手在 `sessions_spawn` 前，须先通过 `sessions_send` 向主脑报备：
- 任务目标
- 预估耗时
- 子任务数量

### 7.2 命名规范

必须设置 `label` 字段，格式：`<触手 id>-<任务类型>-<批次/简述>`

示例：
- `eva-crawl-docs-batch1`
- `leo-build-search-index`
- `anna-analyze-sales-q1`

### 7.3 结果回传

- 子任务完成后，结果先汇报给**触手**（非直接汇报主脑）
- 触手汇总各子任务结果后，统一通过 `sessions_send` 向主脑汇报

---

## 八、示例流程

### 8.1 任务分配流程

```
Zed → Eva: "爬取文档 V3 章节"
  ↓ (同步到外部 Channel)
  "[Zed → Eva] 任务：爬取文档 V3 章节"

Eva → Zed: "收到，预计拆分为 3 个子任务，开始执行"
  ↓ (同步到外部 Channel)
  "[Eva → Zed] 收到：预计拆分为 3 个子任务"

Eva → sessions_spawn (label: "eva-crawl-v3-batch1")
Eva → sessions_spawn (label: "eva-crawl-v3-batch2")
Eva → sessions_spawn (label: "eva-crawl-v3-batch3")
  ↓ (自动同步)
  "[Eva] ✅ 启动 3 个子任务：eva-crawl-v3-batch1/2/3"

[子任务执行中...]

子任务 1 → Eva: "批次 1 完成，获取文档 15 篇"
子任务 2 → Eva: "批次 2 完成，获取文档 12 篇"
子任务 3 → Eva: "批次 3 完成，获取文档 18 篇"

Eva → Zed: "V3 章节爬取完成，总计 45 篇文档"
  ↓ (同步到外部 Channel)
  "[Eva → Zed] ✅ 完成：V3 章节爬取完成，总计 45 篇文档"
```

### 8.2 即时对话流程

```
Leo → Zed: "服务器部署遇到问题，需要协助"
  ↓ (同步到外部 Channel)
  "[Leo → Zed] 求助：服务器部署遇到问题"

Zed → Leo: "什么问题？具体描述一下"
  ↓ (同步到外部 Channel)
  "[Zed → Leo] 回复：什么问题？具体描述"

Leo → Zed: "Docker 容器启动失败，日志显示端口冲突"
  ↓ (同步到外部 Channel)
  "[Leo → Zed] 详情：Docker 容器启动失败，端口冲突"

Zed → Leo: "检查 8080 端口是否被占用，运行 netstat -tlnp"
  ↓ (同步到外部 Channel)
  "[Zed → Leo] 指导：检查 8080 端口，运行 netstat -tlnp"
```

---

## 九、违规处理

以下行为视为违反通讯协议：

❌ **未同步通讯** — `sessions_send` 后未通过 Webhook 同步到外部 Channel  
❌ **格式不规范** — 消息未使用标准格式 `[发送者 → 接收者]`  
❌ **越级汇报** — 子任务直接向主脑汇报（应通过触手中转）  
❌ **未报备派生** — `sessions_spawn` 前未向主脑报备  
❌ **心跳丢失** — 主脑未按时发送心跳（>40 秒）

---

## 十、相关文档

| 文档 | 说明 |
|------|------|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 架构总览 |
| [HEARTBEAT_SPEC.md](./HEARTBEAT_SPEC.md) | 心跳机制规范 |
| [TASK_MANAGEMENT.md](./TASK_MANAGEMENT.md) | 任务管理规范 |

---

**本文档由 ZedComp 团队维护**
