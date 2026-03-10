# 💾 状态持久化规范 (State Persistence Specification)

**版本**: v1.0  
**生效日期**: 2026-03-10

---

## 一、概述

状态持久化确保触手在主脑失效或其他异常情况下能够保存任务进度，并在恢复后继续执行。

---

## 二、存储位置

### 2.1 目录结构

```
/{tentacle-workspace}/
├── tentacle-state/           # 状态持久化目录（必需）
│   ├── current-task.json     # 当前任务状态
│   ├── progress.json         # 进度信息
│   ├── context.json          # 上下文信息
│   └── history/              # 历史状态归档（可选）
│       └── 2026-03-10/
│           └── task-001.json
├── tentacle-id/              # 触手 ID 目录
└── ...                       # 其他工作文件
```

### 2.2 各触手状态目录示例

| 触手 | 状态目录 |
|------|----------|
| Geoff | `/geoff/workspace/tentacle-state/` |
| Eva | `/eva/workspace/tentacle-state/` |
| Leo | `/leo/workspace/tentacle-state/` |
| Anna | `/anna/workspace/tentacle-state/` |
| Mia | `/mia/workspace/tentacle-state/` |
| Kia | `/kia/workspace/tentacle-state/` |

---

## 三、状态文件结构

### 3.1 current-task.json

**用途**: 存储当前任务的完整信息

```json
{
  "taskId": "task-20260310-001",
  "taskName": "爬取文档 V3 章节",
  "assignedBy": "Zed",
  "assignedAt": "2026-03-10T14:30:00+08:00",
  "status": "in_progress",
  "priority": "normal",
  "deadline": "2026-03-11T18:00:00+08:00",
  "subtasks": [
    {
      "id": "sucker-1",
      "name": "爬取第 1-5 章",
      "status": "completed",
      "result": "5 篇文档",
      "completedAt": "2026-03-10T15:00:00+08:00"
    },
    {
      "id": "sucker-2",
      "name": "爬取第 6-10 章",
      "status": "in_progress",
      "result": null,
      "startedAt": "2026-03-10T15:05:00+08:00"
    },
    {
      "id": "sucker-3",
      "name": "爬取第 11-15 章",
      "status": "pending",
      "result": null
    }
  ],
  "metadata": {
    "source": "外部文档平台",
    "url": "https://vip.kingdee.com/knowledge/specialDetail/218022218066869248",
    "totalChapters": 15
  }
}
```

### 3.2 progress.json

**用途**: 存储任务进度和统计信息

```json
{
  "lastUpdate": "2026-03-10T15:45:00+08:00",
  "completionPercentage": 60,
  "estimatedCompletion": "2026-03-10T17:00:00+08:00",
  "statistics": {
    "totalSubtasks": 3,
    "completedSubtasks": 1,
    "inProgressSubtasks": 1,
    "pendingSubtasks": 1,
    "failedSubtasks": 0
  },
  "notes": "已完成 2/3 吸盘任务，预计 1 小时内完成",
  "checkpoints": [
    {
      "time": "2026-03-10T15:00:00+08:00",
      "event": "sucker-1 completed",
      "details": "成功获取 5 篇文档"
    },
    {
      "time": "2026-03-10T15:05:00+08:00",
      "event": "sucker-2 started",
      "details": "开始爬取第 6-10 章"
    }
  ]
}
```

### 3.3 context.json

**用途**: 存储任务执行的上下文信息

```json
{
  "variables": {
    "baseUrl": "https://vip.kingdee.com",
    "cookies": "...",
    "sessionToken": "..."
  },
  "conversation": [
    {
      "timestamp": "2026-03-10T14:30:00+08:00",
      "from": "Zed",
      "to": "Eva",
      "message": "爬取文档 V3 章节"
    },
    {
      "timestamp": "2026-03-10T14:31:00+08:00",
      "from": "Eva",
      "to": "Zed",
      "message": "收到，预计拆分为 3 个子任务"
    }
  ],
  "resources": [
    {
      "type": "file",
      "path": "/eva/workspace/output/doc-001.md",
      "description": "第 1 章文档"
    },
    {
      "type": "file",
      "path": "/eva/workspace/output/doc-002.md",
      "description": "第 2 章文档"
    }
  ],
  "errors": [],
  "warnings": []
}
```

---

## 四、持久化时机

### 4.1 必须持久化的场景

| 场景 | 触发时机 | 说明 |
|------|----------|------|
| **任务接收** | 收到新任务时 | 保存任务基本信息 |
| **任务开始** | 开始执行时 | 保存初始状态 |
| **状态变更** | 子任务完成/失败时 | 保存进度更新 |
| **定期保存** | 每 5 分钟 | 防止意外丢失 |
| **主脑失效** | 检测到心跳超时 | 紧急保存，准备暂停 |
| **任务完成** | 任务结束时 | 保存最终状态 |

### 4.2 持久化伪代码

```javascript
class StatePersistence {
  constructor(tentacleId) {
    this.tentacleId = tentacleId;
    this.stateDir = `/${tentacleId}/workspace/tentacle-state`;
    this.autoSaveInterval = 300000; // 5 分钟
  }

  // 启动自动保存
  startAutoSave() {
    setInterval(() => {
      this.saveState();
    }, this.autoSaveInterval);
  }

  // 保存状态
  async saveState() {
    const timestamp = new Date().toISOString();
    
    // 1. 保存当前任务
    await this.saveCurrentTask();
    
    // 2. 保存进度
    await this.saveProgress();
    
    // 3. 保存上下文
    await this.saveContext();
    
    // 4. 记录保存时间
    await this.saveMetadata({
      lastSave: timestamp,
      reason: 'auto_save'
    });
  }

  // 紧急保存（主脑失效时）
  async emergencySave(reason) {
    console.log(`[💾] 紧急保存状态，原因：${reason}`);
    
    await this.saveState();
    
    // 额外保存一个快照
    await this.saveSnapshot(reason);
  }

  // 保存快照
  async saveSnapshot(reason) {
    const date = new Date().toISOString().split('T')[0];
    const taskId = this.currentTask?.id || 'unknown';
    const snapshotPath = `${this.stateDir}/history/${date}/${taskId}-${reason}.json`;
    
    await fs.ensureDir(path.dirname(snapshotPath));
    await fs.writeJson(snapshotPath, {
      timestamp: new Date().toISOString(),
      reason: reason,
      task: this.currentTask,
      progress: this.progress,
      context: this.context
    });
  }
}
```

---

## 五、状态恢复

### 5.1 恢复流程

```
1. 主脑重新上线
   ↓
2. 主脑请求状态汇报
   ↓
3. 触手检查 tentacle-state/ 目录
   ↓
4. 读取 current-task.json
   ↓
5. 验证状态有效性
   ↓
6. 汇报状态给主脑
   ↓
7. 主脑确认后，继续执行
```

### 5.2 恢复伪代码

```javascript
async function restoreState() {
  const stateDir = `/${tentacleId}/workspace/tentacle-state`;
  
  try {
    // 1. 读取状态文件
    const currentTask = await fs.readJson(`${stateDir}/current-task.json`);
    const progress = await fs.readJson(`${stateDir}/progress.json`);
    const context = await fs.readJson(`${stateDir}/context.json`);
    
    // 2. 验证状态
    if (!currentTask || !currentTask.taskId) {
      console.log('[ℹ️] 无有效状态，等待新任务');
      return null;
    }
    
    // 3. 检查是否已过期（超过 24 小时）
    const lastUpdate = new Date(progress.lastUpdate);
    const hoursSinceUpdate = (Date.now() - lastUpdate) / (1000 * 60 * 60);
    
    if (hoursSinceUpdate > 24) {
      console.warn(`[⚠️] 状态已过期 (${hoursSinceUpdate.toFixed(1)} 小时)`);
      // 可以选择归档旧状态
      await archiveOldState();
      return null;
    }
    
    // 4. 恢复状态
    this.currentTask = currentTask;
    this.progress = progress;
    this.context = context;
    
    console.log(`[✅] 状态恢复成功：${currentTask.taskName}`);
    console.log(`[📊] 进度：${progress.completionPercentage}%`);
    
    return {
      restored: true,
      task: currentTask,
      progress: progress
    };
    
  } catch (error) {
    console.error('[❌] 状态恢复失败:', error);
    return null;
  }
}
```

---

## 六、状态清理

### 6.1 清理策略

| 场景 | 操作 |
|------|------|
| **任务完成** | 归档到 history/，保留 30 天 |
| **任务取消** | 归档到 history/，保留 7 天 |
| **状态过期** | 超过 24 小时未更新，归档 |
| **磁盘空间不足** | 清理 30 天前的历史状态 |

### 6.2 清理伪代码

```javascript
async function cleanupOldStates() {
  const historyDir = `${stateDir}/history`;
  const now = Date.now();
  const maxAge = 30 * 24 * 60 * 60 * 1000; // 30 天
  
  const dates = await fs.readdir(historyDir);
  
  for (const date of dates) {
    const datePath = `${historyDir}/${date}`;
    const dateTimestamp = new Date(date).getTime();
    const age = now - dateTimestamp;
    
    if (age > maxAge) {
      console.log(`[🗑️] 清理过期状态：${date}`);
      await fs.remove(datePath);
    }
  }
}
```

---

## 七、文件格式规范

### 7.1 编码格式

- **文件编码**: UTF-8
- **JSON 格式**: 缩进 2 空格，便于人工阅读
- **换行符**: LF (Unix 风格)

### 7.2 时间格式

- **ISO 8601**: `2026-03-10T15:00:00+08:00`
- **时区**: 使用本地时区（Asia/Shanghai）

### 7.3 文件权限

```bash
# 状态目录权限
chmod 700 /{tentacle-id}/workspace/tentacle-state/

# 状态文件权限
chmod 600 /{tentacle-id}/workspace/tentacle-state/*.json
```

---

## 八、安全考虑

### 8.1 敏感信息处理

| 信息类型 | 处理方式 |
|----------|----------|
| **API 密钥** | 加密存储或引用外部密钥管理 |
| **Cookie/Token** | 加密存储，设置过期时间 |
| **密码** | 禁止明文存储 |
| **个人隐私** | 脱敏处理 |

### 8.2 备份策略

```
/{tentacle-workspace}/
├── tentacle-state/           # 主状态目录
└── .backup/
    └── tentacle-state/       # 定期备份（可选）
        └── 2026-03-10/
```

---

## 九、监控指标

| 指标 | 说明 | 告警阈值 |
|------|------|----------|
| `state.save.latency` | 保存延迟 | > 1 秒 |
| `state.restore.latency` | 恢复延迟 | > 5 秒 |
| `state.size` | 状态文件大小 | > 10 MB |
| `state.age` | 状态年龄 | > 24 小时 |
| `state.save.errors` | 保存错误次数 | > 3 次/小时 |

---

## 十、测试用例

### 10.1 正常保存测试

```
给定：触手正在执行任务
当：每 5 分钟自动保存
则：
  1. current-task.json 更新
  2. progress.json 更新
  3. 文件权限正确 (600)
```

### 10.2 紧急保存测试

```
给定：主脑突然离线
当：触手检测到心跳超时
则：
  1. 立即保存当前状态
  2. 创建历史快照
  3. 状态包含暂停原因
```

### 10.3 状态恢复测试

```
给定：触手之前保存了状态
当：主脑重新上线并请求恢复
则：
  1. 成功读取状态文件
  2. 状态验证通过
  3. 任务从断点继续
```

---

**本文档由 ZedComp 团队维护**
