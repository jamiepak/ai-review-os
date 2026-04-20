# AI Review OS — Automation 自动化系统文档

> 版本：v3.0 | 更新时间：2026-04-20
> 
> 本文档完整梳理 AI Review OS 中所有 WorkBuddy Automation 自动化任务的产品逻辑、技术配置、数据流和运行机制。

---

## 一、系统总览

### 1.1 架构概要

AI Review OS 的自动化系统基于 **WorkBuddy Automation** 平台，采用 **事件驱动 + 动态调度** 架构。核心思想：不再按固定时间触发，而是以「会议主干表」为数据源，根据每场会议的实际开始/结束时间动态计算触发窗口。

```
┌─────────────────────────────────────────────────────────┐
│                    WorkBuddy Automation                  │
│                                                         │
│   automation-3（评审事件调度器）                           │
│   ┌─────────────┐    ┌──────────────┐                   │
│   │ 每小时运行   │───▶│ 扫描会议表    │                   │
│   └─────────────┘    └──────┬───────┘                   │
│                             │                           │
│              ┌──────────────┼──────────────┐            │
│              ▼              ▼              ▼            │
│     📋 报名提醒      📊 评审汇总      📝 评审总结       │
│    (开始前4h)       (开始前10min)    (结束后1h)          │
│              │              │              │            │
│              ▼              ▼              ▼            │
│         企微群 Webhook（template_card 卡片推送）         │
└─────────────────────────────────────────────────────────┘

          ▲ 数据源                    ▲ 防重复
          │                          │
 .review_meetings.json     .review_automation_state.json
 .review_records_cache.json
```

### 1.2 演进历程

| 阶段 | 时间 | 方案 | 状态 |
|------|------|------|------|
| v1 | 2026-04-16 | 4 个固定时间 Automation（10:30 提醒 / 14:30 汇总 / 16:30 归档 / 每小时概述） | ❌ 已废弃 |
| v2 | 2026-04-17 | 重建 4 个 Automation，优化 prompt | ❌ 已废弃 |
| **v3** | **2026-04-20** | **1 个统一调度器，会议数据驱动，动态窗口触发** | ✅ 当前方案 |

v1→v3 的核心变化：
- **固定时间** → **会议驱动**：评审时间不再写死为"周一二四 15:00"，完全由会议表决定
- **4 个 Automation** → **1 个统一调度器**：减少配置碎片化，逻辑内聚
- **markdown 消息** → **template_card 卡片**：视觉更专业，支持跳转链接

---

## 二、当前活跃 Automation 详情

### 2.1 automation-3：评审事件调度器

这是系统中 **唯一的活跃 Automation**，承担了报名提醒、评审汇总的全部调度职责。

#### 基本配置

| 属性 | 值 |
|------|------|
| **ID** | `automation-3` |
| **名称** | 评审事件调度器 |
| **状态** | PAUSED（截至 2026-04-20 15:30，被人工暂停） |
| **调度策略** | `FREQ=HOURLY;INTERVAL=1;BYDAY=MO,TU,WE,TH,FR,SA,SU`（每小时，每天） |
| **调度类型** | recurring（循环执行） |
| **模型** | claude-opus-4.6-1m |
| **推送到微信** | 否（由 Automation 自身通过 Webhook 推送） |
| **工作目录** | `/Users/jamiepak/WorkBuddy/20260415164609` |
| **创建时间** | 2026-04-20 10:59:36 |
| **上次运行** | 2026-04-20 15:03:54 |
| **下次运行** | 2026-04-20 16:03:54（暂停中不会执行） |

#### 触发规则

| 事件 | 触发时机 | 窗口容差 | 前置条件 |
|------|----------|---------|----------|
| 📋 **报名提醒** | 会议开始前 4 小时 | ±30 分钟 | 未取消、未归档、未发过提醒 |
| 📊 **评审汇总** | 会议开始前 10 分钟 | ±30 分钟 | 未取消、有报名、未发过汇总 |

> **注意**：原设计中还有「📝 评审总结」（会议结束后 1h），但当前 prompt 中未包含此事件，实际仅实现了报名提醒和评审汇总两个动作。

#### 完整 Prompt

```
你是 AI Review 系统的事件调度器。每小时运行一次，扫描会议表，根据每场会议的实际
时间触发对应的推送动作。

## 触发规则

| 事件 | 触发时机 | 条件 |
|------|---------|------|
| 📋 报名提醒 | 会议开始前 4 小时（±30min 窗口） | 未取消、未归档、未发过提醒 |
| 📊 评审汇总 | 会议开始前 10 分钟（±30min 窗口） | 未取消、有报名、未发过汇总 |

## 步骤

### 1. 读取数据
- 会议表：.review_meetings.json
- 报名缓存：.review_records_cache.json
- 调度状态：.review_automation_state.json

### 2. 遍历会议，判断触发
- 跳过 cancelled: true 或 archived: true
- 解析 start_time 和 end_time（ISO 8601 带时区）
- 计算当前时间与各触发点的差值
- 检查状态文件中是否已发送

### 3. 执行动作
- 报名提醒：通过 Webhook 发送 template_card
- 评审汇总：通过 Webhook 发送 template_card（含报名列表）

### 4. 保存状态
更新 .review_automation_state.json

### 5. 清理过期状态
删除超过 7 天的会议记录
```

（完整 prompt 含 Webhook JSON 模板，约 3000 字，详见数据库原文）

#### 推送渠道

| 渠道 | 方式 | 地址 |
|------|------|------|
| 企微群机器人 | Webhook POST | `https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=c83d814f-a7d5-40bd-b0ec-e58ca3bedcb8` |
| 消息格式 | `template_card` → `text_notice` 类型 | — |
| 跳转链接 | 评审智能表格 | `https://doc.weixin.qq.com/smartsheet/s3_AZ4AlQYlAKY...` |

---

## 三、已废弃的 Automation（历史记录）

以下 Automation 已被删除（数据库中不存在），仅在执行 memory 中保留记录。

### 3.1 automation（评审报名提醒）

| 属性 | 值 |
|------|------|
| **原调度** | 每天 10:30 |
| **功能** | 执行 `review-automation.py remind`，评审日推送报名提醒 |
| **废弃原因** | 被 automation-3 统一调度替代 |
| **最后执行** | 2026-04-20 10:30（已成功） |

### 3.2 automation-2（评审汇总推送）

| 属性 | 值 |
|------|------|
| **原调度** | 每天 14:30 |
| **功能** | 执行 `review-automation.py summary`，读取概述推送群汇总 |
| **废弃原因** | 被 automation-3 统一调度替代 |
| **最后执行** | 2026-04-16 14:30（空汇总，已成功） |

### 3.3 automation-4（需求概述自动总结）

| 属性 | 值 |
|------|------|
| **原调度** | 每小时 |
| **功能** | 扫描缓存中 summary 为空的记录，AI 生成概述并更新 |
| **废弃原因** | 数据库中已删除（可能因 Webhook 更新持续失败 errcode=40058） |
| **最后执行** | 2026-04-20 10:02（无需处理） |
| **已知问题** | Webhook `update_records` 对部分 record_id 返回 errcode=40058 |

---

## 四、数据文件规格

### 4.1 会议主干表 `.review_meetings.json`

这是 Automation 调度的**核心驱动数据**。

```jsonc
{
  "meetings": {
    "<meeting_key>": {
      "meeting_id": "",          // 腾讯会议 ID（import/manual 方式创建时有值）
      "meeting_code": "",        // 会议号
      "subject": "",             // 会议主题
      "review_type": "",         // 评审类型：设计评审 / 纯产品评审
      "start_time": "",          // ISO 8601 带时区，如 "2026-04-21T15:00:00+08:00"
      "end_time": "",            // ISO 8601 带时区
      "source": "",              // 来源：recurring / manual / import
      "stories": [],             // 关联的 record_id 数组
      "archived": false,         // 是否已归档
      "cancelled": false,        // 是否已取消
      "created_at": ""           // 创建时间
    }
  }
}
```

**当前数据统计**（截至 2026-04-20）：

| 状态 | 数量 | 说明 |
|------|------|------|
| 活跃 | 5 | 可触发调度的会议 |
| 已取消 | 5 | 跳过，不触发任何事件 |
| 已归档 | 1 | 跳过，不触发任何事件 |
| **合计** | **11** | — |

### 4.2 报名缓存 `.review_records_cache.json`

```jsonc
{
  "<YYYY-MM-DD>": {             // 评审日期
    "stories": [
      {
        "record_id": "",         // 智能表格 record_id
        "name": "",              // 需求名称
        "creator": "",           // 产品经理
        "designer": "",          // 设计师
        "story_id": null,        // TAPD story ID
        "summary": "",           // 需求概述（AI 生成 / 待补充）
        "tapd_url": "",          // TAPD 链接
        "review_type": "",       // 评审类型
        "duration": "",          // 预估时长（分钟）
        "submitted_at": ""       // 提交时间
      }
    ],
    "review_time": "15:00-16:00",
    "review_type": "",
    "ft": ""                     // FT（功能团队）
  }
}
```

**当前数据统计**：4 个日期，5 条报名记录。

### 4.3 调度状态 `.review_automation_state.json`

```jsonc
{
  "<meeting_key>": {
    "remind_sent": true,         // 报名提醒是否已发送
    "summary_sent": true         // 评审汇总是否已发送
  }
}
```

**防重复机制**：每次 Automation 运行前读取此文件，已发送的事件不会重复触发。超过 7 天的记录会被自动清理。

---

## 五、旧版脚本 `review-automation.py`

这是 v1 时期的辅助脚本，被 automation-3 的 AI prompt 直接替代。保留作为参考和降级方案。

### 5.1 功能概览

| 模式 | 原定时间 | 功能 |
|------|----------|------|
| `remind` | 10:30 | 判断是否评审日（周一/二/四）→ 发送 markdown 报名提醒 |
| `summary` | 14:30 | 读取当日缓存 → 构建汇总 → 推送群 |
| `archive` | 16:30 | 调用 `review-archive.py` 归档脚本 |

### 5.2 关键配置

```python
BOT_WEBHOOK = 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=c83d814f-a7d5-40bd-b0ec-e58ca3bedcb8'
SIGNUP_PAGE_URL = 'http://localhost:8766/'
REVIEW_DAYS = [0, 1, 3]  # 周一、周二、周四
REVIEW_TIME = '15:00-16:00'
```

### 5.3 与 automation-3 的对比

| 维度 | review-automation.py（旧） | automation-3（新） |
|------|---------------------------|-------------------|
| 评审时间 | 固定 15:00-16:00 | 每场会议独立时间 |
| 评审日 | 固定周一/二/四 | 按会议表动态 |
| 触发方式 | 固定时间执行 | 窗口匹配（±30min） |
| 消息格式 | markdown | template_card 卡片 |
| 防重复 | 无（依赖不重复调用） | 状态文件防重复 |
| 报名方式 | 网页表单 | @Review bot 发 TAPD 链接 |

---

## 六、执行记录与监控

### 6.1 数据库执行记录

执行记录存储在 SQLite 数据库中：

```
~/Library/Application Support/WorkBuddy/automations/automations.db
  └── automation_runs 表
```

| 字段 | 说明 |
|------|------|
| `thread_id` | 执行线程 ID |
| `automation_id` | 关联的 Automation ID |
| `status` | 执行状态：`PENDING_REVIEW` / `COMPLETED` / `FAILED` |
| `result_success` | 是否成功（1=成功，0=失败） |
| `thread_title` | 执行摘要（AI 生成） |
| `created_at` / `updated_at` | 时间戳 |

**automation-3 执行统计**（截至 2026-04-20 15:30）：

| 指标 | 值 |
|------|------|
| 总执行次数 | 4 次 |
| 成功次数 | 4 次 |
| 失败次数 | 0 次 |
| 成功率 | 100% |

### 6.2 Memory 执行日志

每个 Automation 在 `.codebuddy/automations/<id>/memory.md` 中记录详细的执行日志。

| Automation | Memory 文件大小 | 最后更新 |
|------------|----------------|----------|
| automation | 522 B | 2026-04-20 10:30 |
| automation-2 | 233 B | 2026-04-16 14:30 |
| automation-3 | 2.45 KB | 2026-04-20 15:03 |
| automation-4 | 3.78 KB | 2026-04-20 10:02 |

### 6.3 最近执行时间线

| 时间 | Automation | 动作 | 结果 |
|------|-----------|------|------|
| 04-20 15:02 | automation-3 | 扫描 11 会议 → 命中汇总窗口 → 发送 2 条汇总 | ✅ errcode=0 |
| 04-20 14:01 | automation-3 | 扫描 10 会议 → 汇总窗口未到 | ✅ 安静结束 |
| 04-20 13:00 | automation-3 | 扫描 10 会议 → 无事件触发 | ✅ 安静结束 |
| 04-20 11:59 | automation-3 | 首次扫描 → 创建空状态文件 | ✅ 安静结束 |
| 04-20 10:30 | automation | 推送报名提醒 | ✅ errcode=0 |
| 04-20 10:02 | automation-4 | 扫描缓存 → 所有 summary 非空 | ✅ 无需处理 |

---

## 七、外部依赖

### 7.1 企微群机器人 Webhook

| 配置项 | 值 |
|--------|------|
| Webhook URL | `https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=c83d814f-a7d5-40bd-b0ec-e58ca3bedcb8` |
| 消息类型 | `template_card`（text_notice） |
| 用途 | 报名提醒、评审汇总推送 |

### 7.2 企微智能表格 Webhook

| 配置项 | 值 |
|--------|------|
| Webhook URL | `key=64OEkYaHn4TCprkYWj3QYV4wZrnYTIikJ9RRE2pXzj5kZvrBKYYbjjFzWsHPWdLM9src3NHDtI8uHyRvZytK4wPJYgwmQl5rvGfrVLhEUd7C` |
| 支持操作 | `add_records`、`update_records` |
| 不支持操作 | `get_records`、`delete_records` |
| 用途 | 写入/更新评审记录 |

### 7.3 评审智能表格

| 配置项 | 值 |
|--------|------|
| URL | `https://doc.weixin.qq.com/smartsheet/s3_AZ4AlQYlAKYCNKAwzfGunQ0KeeACd` |
| 字段数量 | 13 个 |
| 读取方式 | 本地缓存 `.review_records_cache.json`（Webhook 不支持读取） |

### 7.4 WorkBuddy 平台

| 配置项 | 值 |
|--------|------|
| Automation 存储 | `~/Library/Application Support/WorkBuddy/automations/automations.db`（SQLite） |
| 执行 Memory | `<workspace>/.codebuddy/automations/<id>/memory.md` |
| AI 模型 | claude-opus-4.6-1m |

---

## 八、已知问题与待优化

### 8.1 已知问题

| # | 问题 | 严重程度 | 状态 |
|---|------|---------|------|
| 1 | Webhook `update_records` 对部分 record_id 返回 errcode=40058 | 🟡 中 | 待排查 |
| 2 | automation-4（需求概述）已删除，该功能缺失 | 🟡 中 | 需决策是否恢复 |
| 3 | 「评审总结」（会后归档）事件在 prompt 中未实现 | 🟡 中 | 需补充 |
| 4 | 部分测试数据（test001-006）残留在缓存中 | 🟢 低 | 可清理 |
| 5 | automation-3 当前状态为 PAUSED | 🔴 高 | 需确认是否应恢复 |

### 8.2 待优化方向

| # | 优化项 | 说明 |
|---|--------|------|
| 1 | **补充评审总结事件** | 会议结束后 1h，读取转写记录，生成评审总结并归档 |
| 2 | **恢复需求概述功能** | 新报名无 summary 时自动读取 TAPD/文档生成概述 |
| 3 | **RRULE 优化** | 当前全天候每小时运行，可限制为工作日工作时段（如 `BYHOUR=9,10,...,18`） |
| 4 | **清理测试数据** | 删除缓存和会议表中的测试记录 |
| 5 | **部署到 Mac mini** | 将 Automation 迁移到办公室 Mac mini 常驻运行 |

---

## 九、配置变更指南

### 9.1 修改调度频率

通过 WorkBuddy Automation 管理界面或 `automation_update` 工具修改 `rrule`：

```
# 每 2 小时运行一次
FREQ=HOURLY;INTERVAL=2

# 仅工作日每小时
FREQ=HOURLY;INTERVAL=1;BYDAY=MO,TU,WE,TH,FR

# 仅工作日 9-18 点每小时
FREQ=WEEKLY;BYDAY=MO,TU,WE,TH,FR;BYHOUR=9,10,11,12,13,14,15,16,17,18;BYMINUTE=0
```

### 9.2 新增触发事件

在 automation-3 的 prompt 中添加新的触发规则段落，格式参考现有的「报名提醒」和「评审汇总」。

### 9.3 更换 Webhook

替换 prompt 中的 Webhook URL（`key=...` 部分）即可。同时需要更新 `review-automation.py` 和 `review-bot.py` 中的硬编码 URL。

---

## 附录 A：数据库 Schema

### automations 表

```sql
CREATE TABLE automations (
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  prompt TEXT NOT NULL,
  status TEXT NOT NULL,              -- ACTIVE / PAUSED
  schedule_type TEXT NOT NULL DEFAULT 'recurring',  -- recurring / once
  next_run_at INTEGER,               -- Unix timestamp (ms)
  last_run_at INTEGER,               -- Unix timestamp (ms)
  cwds TEXT NOT NULL DEFAULT '[]',   -- JSON 数组，工作目录
  rrule TEXT NOT NULL DEFAULT '',    -- iCalendar RRULE
  scheduled_at TEXT,                 -- 一次性任务时间（ISO 8601）
  valid_from TEXT,                   -- 有效期开始
  valid_until TEXT,                  -- 有效期结束
  model_id TEXT,                     -- AI 模型 ID
  model_is_thinking INTEGER NOT NULL DEFAULT 0,
  created_at INTEGER NOT NULL,       -- Unix timestamp (ms)
  updated_at INTEGER NOT NULL,       -- Unix timestamp (ms)
  push_to_wechat INTEGER NOT NULL DEFAULT 0,  -- 是否推送到微信
  skills_json TEXT NOT NULL DEFAULT '[]'       -- 关联 skill 列表
);
```

### automation_runs 表

```sql
CREATE TABLE automation_runs (
  thread_id TEXT PRIMARY KEY,
  automation_id TEXT NOT NULL,
  status TEXT NOT NULL,              -- PENDING_REVIEW / COMPLETED / FAILED
  read_at INTEGER,
  thread_title TEXT,                 -- 执行摘要
  source_cwd TEXT,
  runs_json TEXT,
  result_success INTEGER,            -- 1=成功, 0=失败
  created_at INTEGER NOT NULL,
  updated_at INTEGER NOT NULL
);
```

---

## 附录 B：文件清单

| 文件路径 | 类型 | 用途 |
|----------|------|------|
| `.review_meetings.json` | 数据 | 会议主干表，调度驱动源 |
| `.review_records_cache.json` | 数据 | 报名记录缓存 |
| `.review_automation_state.json` | 状态 | 调度防重复状态 |
| `review-automation.py` | 脚本 | 旧版定时任务脚本（保留） |
| `review-archive.py` | 脚本 | 会后归档脚本 |
| `.codebuddy/automations/*/memory.md` | 日志 | 各 Automation 执行记录 |
| `~/Library/.../automations.db` | 数据库 | Automation 配置和运行记录 |
