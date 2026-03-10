# 💓 心跳机制规范 (Heartbeat Specification)

**版本**: v1.0  
**生效日期**: 2026-03-10

---

## 一、概述

心跳机制用于检测主脑是否在线，确保触手能够在主脑失效时及时暂停待命。

---

## 二、心跳流程

```
┌─────────────────┐
│   主脑 (Zed)     │
│   (Main Brain)  │
└────────┬────────┘
         │
         │ 每 30 秒发送心跳
         │ [💓 HEARTBEAT] Zed is alive
         ↓
┌─────────────────┐
│  外部 Channel  │
│  (广播通道)      │
└────────┬────────┘
         │
         │ 所有触手接收
         ↓
┌────────────────────────────────────────┐
│              触手们                      │
│  ┌────────┐  ┌────────┐  ┌────────┐   │
│  │ Geoff  │  │  Eva   │  │  Leo   │   │
│  └────────┘  └────────┘  └────────┘   │
└────────────────────────────────────────┘
```

---

## 三、心跳参数

| 参数 | 值 | 说明 |
|------|-----|------|
| **心跳间隔** | 30 秒 | 主脑发送心跳的频率 |
| **超时阈值** | 90 秒 | 触手判定主脑离线的时长（3 次心跳间隔） |
| **重试次数** | 3 次 | 主脑重新上线后的状态同步重试次数 |
| **心跳格式** | JSON | 心跳消息的数据格式 |

---

## 四、消息格式

### 4.1 主脑发送心跳

```json
{
  "type": "heartbeat",
  "brain": "Zed",
  "timestamp": "2026-03-10T15:00:00+08:00",
  "status": "alive",
  "active_tentacles": 6,
  "uptime_seconds": 3600
}
```

**外部 Channel 消息格式**（人类可读）：
```
[💓 HEARTBEAT] Zed is alive | timestamp: 2026-03-10T15:00:00+08:00 | active_tentacles: 6 | uptime: 1h
```

### 4.2 触手响应（可选）

```json
{
  "type": "heartbeat_ack",
  "tentacle": "Geoff",
  "timestamp": "2026-03-10T15:00:01+08:00",
  "status": "busy",
  "current_task": "项目进度管理",
  "task_progress": 60
}
```

**外部 Channel 消息格式**（人类可读）：
```
[👋 ACK] Geoff received | status: busy | task: 项目进度管理 | progress: 60%
```

---

## 五、触手侧实现

### 5.1 心跳监测伪代码

```javascript
class TentacleHeartbeatMonitor {
  constructor(tentacleId) {
    this.tentacleId = tentacleId;
    this.lastHeartbeat = null;
    this.timeoutThreshold = 90000; // 90 秒
    this.timer = null;
  }

  // 收到心跳
  onHeartbeat(message) {
    this.lastHeartbeat = Date.now();
    this.resetTimer();
  }

  // 重置超时计时器
  resetTimer() {
    if (this.timer) {
      clearTimeout(this.timer);
    }
    this.timer = setTimeout(() => {
      this.onBrainOffline();
    }, this.timeoutThreshold);
  }

  // 主脑离线处理
  async onBrainOffline() {
    console.warn(`[⚠️] 主脑离线，已超时 ${this.timeoutThreshold/1000} 秒`);
    
    // 1. 保存当前状态
    await this.saveState();
    
    // 2. 暂停当前任务
    await this.pauseCurrentTask();
    
    // 3. 发送告警到外部 Channel
    await this.sendAlert('[⚠️] 主脑离线，已暂停待命');
    
    // 4. 进入等待状态
    this.enterWaitingMode();
  }

  // 保存状态
  async saveState() {
    const state = {
      taskId: this.currentTask?.id,
      taskName: this.currentTask?.name,
      progress: this.currentTask?.progress,
      pausedAt: new Date().toISOString(),
      reason: 'brain_offline'
    };
    
    await fs.writeFile(
      `/{$this.tentacleId}/workspace/tentacle-state/current-task.json`,
      JSON.stringify(state, null, 2)
    );
  }
}
```

---

## 六、主脑侧实现

### 6.1 心跳发送伪代码

```javascript
class BrainHeartbeatService {
  constructor(brainId) {
    this.brainId = brainId;
    this.startTime = Date.now();
    this.activeTentacles = new Set();
  }

  // 启动心跳服务
  start() {
    // 每 30 秒发送心跳
    setInterval(() => {
      this.sendHeartbeat();
    }, 30000);
  }

  // 发送心跳
  async sendHeartbeat() {
    const message = {
      type: 'heartbeat',
      brain: this.brainId,
      timestamp: new Date().toISOString(),
      status: 'alive',
      active_tentacles: this.activeTentacles.size,
      uptime_seconds: Math.floor((Date.now() - this.startTime) / 1000)
    };

    // 通过外部 Channel 发送
    await this.sendToChannel(message);
  }

  // 重新上线后的状态同步
  async onRecover() {
    console.log('[✅] 主脑重新上线，开始状态同步');
    
    // 1. 立即发送心跳
    await this.sendHeartbeat();
    
    // 2. 请求所有触手汇报状态
    await this.requestStatusReport();
    
    // 3. 重试 3 次（如有触手未响应）
    for (let i = 0; i < 3; i++) {
      const missing = this.getMissingTentacles();
      if (missing.length === 0) break;
      
      await sleep(5000);
      await this.requestStatusReport(missing);
    }
  }
}
```

---

## 七、状态转换

### 7.1 触手状态机

```
                    ┌─────────────┐
                    │    idle     │
                    │   (空闲)     │
                    └──────┬──────┘
                           │ 接收任务
                           ↓
                    ┌─────────────┐
          ┌────────│    busy     │────────┐
          │        │  (执行中)    │        │
          │        └──────┬──────┘        │
          │               │               │
    完成任务│               │主脑离线       │主脑上线
          │               ↓               │
          │        ┌─────────────┐        │
          │        │   paused    │        │
          │        │  (暂停)      │        │
          │        └──────┬──────┘        │
          │               │               │
          └───────────────┴───────────────┘
```

### 7.2 状态持久化时机

| 状态转换 | 持久化 | 说明 |
|----------|--------|------|
| idle → busy | ✅ | 保存任务信息 |
| busy → paused | ✅ | 保存进度和上下文 |
| paused → busy | ✅ | 更新恢复时间 |
| busy → idle | ✅ | 保存完成状态 |

---

## 八、异常处理

### 8.1 心跳丢失处理

| 场景 | 处理方式 |
|------|----------|
| 单次心跳丢失 | 忽略（网络波动） |
| 连续 2 次丢失 | 记录警告日志 |
| 连续 3 次丢失（90 秒） | 触发失效处理流程 |

### 8.2 主脑重新上线

```
1. 主脑发送心跳
   ↓
2. 触手收到心跳，退出暂停状态
   ↓
3. 主脑请求状态汇报
   ↓
4. 触手从 tentacle-state/ 恢复状态
   ↓
5. 触手汇报当前状态
   ↓
6. 主脑确认，任务继续
```

---

## 九、监控与告警

### 9.1 监控指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| `heartbeat.interval` | 心跳间隔 | > 40 秒 |
| `heartbeat.latency` | 心跳延迟 | > 5 秒 |
| `tentacle.pause.count` | 触手暂停次数 | > 3 次/小时 |
| `brain.offline.duration` | 主脑离线时长 | > 5 分钟 |

### 9.2 告警消息格式

```
[🚨 ALERT] 主脑离线检测
- 最后心跳时间：2026-03-10T15:00:00+08:00
- 离线时长：90 秒
- 受影响触手：Geoff, Eva, Leo, Anna, Mia, Kia
- 状态：已暂停待命
```

---

## 十、测试用例

### 10.1 正常心跳测试

```
给定：主脑正常运行
当：每 30 秒发送心跳
则：所有触手保持 busy/idle 状态，不触发暂停
```

### 10.2 主脑离线测试

```
给定：主脑突然离线
当：90 秒内无心跳
则：
  1. 所有触手保存状态到 tentacle-state/
  2. 所有触手进入 paused 状态
  3. 发送告警到外部 Channel
```

### 10.3 主脑恢复测试

```
给定：主脑重新上线
当：发送心跳并请求状态汇报
则：
  1. 所有触手退出 paused 状态
  2. 触手汇报当前状态
  3. 任务继续执行
```

---

**本文档由 ZedComp 团队维护**
