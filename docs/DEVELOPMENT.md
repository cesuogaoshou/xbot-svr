# Delta Agent — 三角洲行动代练工作室智能助手

基于 [easy-wx/xbot](https://github.com/easy-wx/xbot) 二次开发的企业微信机器人，为代练工作室提供智能客服、订单管理、打手调度、客户维护等能力。

---

## 目录

- [1. 项目背景](#1-项目背景)
- [2. 基底项目架构（xbot）](#2-基底项目架构xbot)
- [3. 新增模块设计](#3-新增模块设计)
- [4. 数据库设计](#4-数据库设计)
- [5. 命令设计](#5-命令设计)
- [6. 意图引擎（模糊对话）](#6-意图引擎模糊对话)
- [7. 数据看板](#7-数据看板)
- [8. 定时任务](#8-定时任务)
- [9. 开发步骤](#9-开发步骤)
- [10. 部署指南](#10-部署指南)
- [11. 使用示例](#11-使用示例)

---

## 1. 项目背景

### 业务场景

叶来电运营一个三角洲行动（Delta Force: Hawk Ops）代练工作室，当前状态：

| 指标 | 现状 |
|---|---|
| 身份 | 宣传工作室 / 中介派单 |
| 打手数量 | 多人（精确数量待确认） |
| 客户数量 | ~20 个稳定客户 |
| 周单量 | ~20 单 |
| 当前收入 | ~50 元/天（净利） |
| 工具 | 只有手机，在网吧办公 |
| 合作线 | ① 上家单源 ② 宣传合作方 ③ "刘"工作室技术交流 |
| 附加业务 | "撞车"（游戏内物品低买高卖） |

### 痛点

1. **客户服务占时间**：一个人对接客户、谈价格、记需求，效率极低
2. **没有管理系统**：打手、订单、收支全在脑子里，容易出错
3. **用手机办公**：打字慢、切换麻烦、无法同时处理多件事
4. **客户维护缺失**：没有自动回访，客户回头率靠运气

### 目标

开发一个企业微信机器人，让叶来电**在手机上通过聊天就能管理整个工作室**：

- 客户发消息 → 机器人自动回复价格、收集需求、创建订单
- 叶来电报指令 → 机器人执行下单/派单/查进度/看数据
- 超时单子 → 机器人自动催打手
- 老客户 7 天未下单 → 机器人自动回访
- 手机浏览器打开 → 数据看板，一眼看清今天情况

---

## 2. 基底项目架构（xbot）

### 2.1 技术栈

```
Python 3.9+
├── Flask                    ← HTTP 服务
├── wecom-bot-svr            ← 企微机器人回调框架（封装了加解密 + webhook）
│   └── sbzhu/weworkapi_python  ← 底层：企微官方加解密库
├── SQLite (local.db)        ← 权限数据库
└── APScheduler（待引入）     ← 定时任务
```

### 2.2 现有目录结构

```
xbotsvr/
├── wecom_app.py              ← 主入口：Flask 服务 + msg_handler + event_handler
├── cmd_process.py            ← 命令路由：[scene] [cmd] [args] → 动态导入对应模块函数
├── sync_async_proc.py        ← 异步任务：长耗时任务不阻塞企微 5 秒超时
├── auth.py                   ← 权限管理 (UserPermissionHelper → SQLite)
├── config.py                 ← 配置文件（企微参数 + scene_dirs 列表）
│
├── common/                   ← 公共模块
│   ├── __init__.py           ← 导出 logger, FileMsg, MarkdownMsg, XBotMsg
│   ├── logger.py
│   └── datastructs.py
│
├── services/                 ← [待开发] 代练业务模块目录（在 scene_dirs 第 1 位）
│   └── __init__.py
│
├── activities/               ← 活动相关模块（scene_dirs 第 2 位）
├── public/                   ← 公开命令模块，无需权限（scene_dirs 第 3 位）
│   └── pub_demo.py
│
└── spec_line_proc_funcs/     ← # 开头的特殊对话处理
    └── hash_proc.py
```

### 2.3 消息处理流程

```
┌──────────────────────────────────────────────────────────────────┐
│  企微服务器                                                        │
│    │  POST /wecom_bot                                             │
│    ▼                                                              │
│  wecom_app.py                                                     │
│    │  msg_handler(req_msg, server)                                │
│    │    ├── 文本消息 → cmd_process.handle_command(user_id, msg)   │
│    │    │     ├── # 开头 → spec_line_proc_funcs 特殊处理          │
│    │    │     ├── "help" → help_msg()                             │
│    │    │     └── [scene] [cmd] [args] → 动态导入模块+函数         │
│    │    │           ├── 有权限 → 执行 → 返回结果                   │
│    │    │           └── 无权限 → "Permission denied"               │
│    │    └── 其他消息类型 → "暂不支持"                              │
│    │                                                              │
│    └── sync_async_proc.SyncAsyncRspProcessor                      │
│          ├── 2 秒内完成 → 同步返回                                 │
│          └── 超过 2 秒 → 先回空白消息，异步完成后主动推送           │
└──────────────────────────────────────────────────────────────────┘
```

### 2.4 关键约定

- **命令格式**：`[scene] [cmd] [args...]`，如 `order create 老板A 黄金→钻石 200`
- **函数命名**：模块里 `cmd_` + 子命令名，如子命令是 `create`，函数就是 `cmd_create`
- **模块发现**：从 `config.py` 的 `scene_dirs` 列表按顺序搜索
- **返回值**：可以是 `str`、`MarkdownMsg`、`FileMsg`，框架自动选择发送方式
- **权限控制**：仅在模块不在 `public/` 目录时生效

---

## 3. 新增模块设计

### 3.1 新增文件清单

```
xbotsvr/
├── models.py                 ← [新] SQLAlchemy 模型 + 数据库初始化
├── business.db               ← [新] 业务数据库（独立于 local.db）
│
├── services/                 ← 代练业务模块（全是新增）
│   ├── __init__.py           ← [改] 添加包说明
│   ├── order.py              ← [新] 订单管理
│   ├── player.py             ← [新] 打手管理
│   ├── client.py             ← [新] 客户管理
│   ├── price.py              ← [新] 价格查询
│   └── dashboard.py          ← [新] 数据摘要
│
├── rules/
│   └── intents.yaml          ← [新] 关键词意图规则
│
├── templates/
│   └── dashboard.html        ← [新] 手机端数据看板（单独 Flask 路由）
│
├── reminder.py               ← [新] APScheduler 定时任务
├── intent_engine.py          ← [新] 意图匹配引擎
│
└── config.py                 ← [改] 填入真实企微配置
```

### 3.2 模块职责

| 模块 | 职责 | 关键函数 |
|---|---|---|
| `models.py` | 数据库表定义 + Session 管理 | `init_db()`, `get_session()` |
| `services/order.py` | 创建订单、查询、派单、完成、取消 | `cmd_create()`, `cmd_status()`, `cmd_assign()`, `cmd_done()`, `cmd_cancel()`, `cmd_list()` |
| `services/player.py` | 打手 CRUD、评分、状态管理 | `cmd_add()`, `cmd_list()`, `cmd_rate()`, `cmd_pause()`, `cmd_active()` |
| `services/client.py` | 客户管理、回访 | `cmd_list()`, `cmd_add()`, `cmd_detail()` |
| `services/price.py` | 价格表 | `cmd_price()` |
| `services/dashboard.py` | 今日摘要 | `cmd_dashboard()` |
| `intent_engine.py` | 模糊对话关键词匹配 | `match(text) -> dict` |
| `reminder.py` | 定时检查超时订单 + 客户回访 | `check_overdue_orders()`, `check_inactive_clients()` |

---

## 4. 数据库设计

### 4.1 表结构

```sql
-- 打手表
CREATE TABLE players (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT    NOT NULL,              -- 打手名/昵称
    contact         TEXT    NOT NULL,              -- 微信/QQ
    skill_level     TEXT    DEFAULT '',            -- 擅长段位，如 "钻石以下"
    max_daily       INTEGER DEFAULT 5,             -- 每日接单上限
    total_completed INTEGER DEFAULT 0,             -- 累计完成单数
    rating          REAL    DEFAULT 3.0,           -- 评分 1.0-5.0
    deposit_paid    REAL    DEFAULT 0,             -- 已交押金
    status          TEXT    DEFAULT 'active',      -- active | paused | banned
    created_at      TEXT    DEFAULT (datetime('now','localtime'))
);

-- 订单表
CREATE TABLE orders (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    client_name     TEXT    NOT NULL,              -- 客户名
    client_contact  TEXT    DEFAULT '',            -- 客户微信/QQ
    order_type      TEXT    NOT NULL,              -- rank_up | pass | season | items
    detail          TEXT    NOT NULL,              -- 需求详情 "黄金→钻石 3天"
    price           REAL    NOT NULL,              -- 价格
    player_name     TEXT    DEFAULT '',            -- 指派打手名
    player_share    REAL    DEFAULT 0,             -- 打手分成
    status          TEXT    DEFAULT 'pending',     -- pending | assigned | in_progress | done | cancelled
    deadline        TEXT    DEFAULT '',            -- 截止日期
    note            TEXT    DEFAULT '',            -- 备注
    completed_at    TEXT    DEFAULT '',
    created_at      TEXT    DEFAULT (datetime('now','localtime'))
);

-- 收支表
CREATE TABLE transactions (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    type            TEXT    NOT NULL,              -- income | expense
    amount          REAL    NOT NULL,
    category        TEXT    DEFAULT '',            -- order_income | player_share | net_cafe | clip_fee | other
    related_order_id INTEGER,
    note            TEXT    DEFAULT '',
    created_at      TEXT    DEFAULT (datetime('now','localtime'))
);

-- 客户表
CREATE TABLE clients (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    name            TEXT    NOT NULL,
    contact         TEXT    DEFAULT '',
    total_orders    INTEGER DEFAULT 0,
    total_spent     REAL    DEFAULT 0,
    first_order_at  TEXT    DEFAULT '',
    last_order_at   TEXT    DEFAULT '',
    tags            TEXT    DEFAULT '',            -- "老客户,高价单"
    source          TEXT    DEFAULT '',            -- 闲鱼 | 抖音 | 朋友介绍 | 上家
    created_at      TEXT    DEFAULT (datetime('now','localtime'))
);
```

### 4.2 索引

```sql
CREATE INDEX idx_orders_status   ON orders(status);
CREATE INDEX idx_orders_player   ON orders(player_name);
CREATE INDEX idx_orders_client   ON orders(client_name);
CREATE INDEX idx_orders_deadline ON orders(deadline);
CREATE INDEX idx_players_status  ON players(status);
CREATE INDEX idx_clients_name    ON clients(name);
```

---

## 5. 命令设计

所有命令格式：`[模块] [子命令] [参数...]`，在企微群里 @机器人 或私聊机器人发送。

### 5.1 订单模块 (`order`)

```
order create <客户名> <类型> <详情> <价格> [时限]
    创建新订单
    例: order create 老板A rank_up 黄金→钻石 200 3天

order status [订单ID]
    查询订单状态，无参数时返回今日摘要
    例: order status 5

order assign <订单ID> <打手名>
    分配订单给打手
    例: order assign 5 张三

order done <订单ID>
    标记完成，自动记收支
    例: order done 5

order cancel <订单ID> [原因]
    取消订单
    例: order cancel 5 客户不要了

order list [状态]
    按状态筛选订单
    例: order list pending / assigned / overdue
```

### 5.2 打手模块 (`player`)

```
player add <名字> <联系方式> [擅长段位] [日接单上限]
    添加打手
    例: player add 张三 wx_zhangsan 钻石以下 5

player list [状态]
    列出打手
    例: player list / player list all

player rate <打手名> <评分>
    评分 1.0-5.0
    例: player rate 张三 4.5

player pause <打手名>
    暂停派单

player active <打手名>
    恢复派单

player remove <打手名>
    清退打手
```

### 5.3 客户模块 (`client`)

```
client list
    客户列表及消费统计

client add <名字> <联系方式> [来源]
    添加客户
    例: client add 老板B wx_laobanb 闲鱼

client detail <客户名>
    客户详情（历史订单、总消费）
```

### 5.4 价格模块 (`price`)

```
price
    返回价格表（Markdown 格式）
```

### 5.5 看板模块 (`dashboard`)

```
dashboard
    今日数据：收入 / 进行中 / 完成 / 超时 / 在线打手 / 待分配
```

### 5.6 收支模块 (`finance`)

```
finance today
    今日收支汇总

finance week
    本周收支汇总

finance add <类型> <金额> <备注>
    手动记账
    例: finance add expense 28 网吧网费
```

---

## 6. 意图引擎（模糊对话）

### 6.1 设计

客户不会用命令格式说话。意图引擎把自然语言映射到业务动作。**命令格式优先匹配，意图引擎兜底**。

### 6.2 规则配置 (`rules/intents.yaml`)

```yaml
intents:
  - name: ask_price
    keywords: ["多少钱", "价格", "报价", "怎么收费", "便宜"]
    reply: |
      📋 **三角洲行动代练价格表**
      > 青铜 → 黄金：XX 元（1-2天）
      > 黄金 → 钻石：XX 元（2-3天）
      > 钻石 → 统帅：XX 元（3-5天）
      > 通行证全包：XX 元
      > 具体价格看账号情况，发我段位~

  - name: place_order
    keywords: ["下单", "代练", "上分", "帮我打", "想打到"]
    reply: "收到！告诉我：1️⃣当前段位 2️⃣目标段位 3️⃣时间要求，马上安排~"
    action: create_order_draft

  - name: ask_progress
    keywords: ["进度", "打完了吗", "好了没", "还要多久"]
    action: query_client_orders

  - name: ask_safety
    keywords: ["安全", "封号", "会不会被封", "靠谱吗"]
    reply: |
      ✅ 全部手打，不碰外挂
      ✅ 打完不顶号、不毁号
      ✅ 封号问题全额退款
      ✅ 已服务 20+ 老板，零封号记录

  - name: want_refund
    keywords: ["退款", "退钱", "不打了"]
    action: alert_owner   # 不自动处理，通知叶来电

  - name: greeting
    keywords: ["你好", "在吗", "hi", "hello"]
    reply: "老板好！三角洲专业代练，段位/通行证/任务/撞车都可接，有什么需要？"

  - name: ask_time
    keywords: ["多久", "几天", "什么时候能"]
    reply: "青铜到钻石一般 2-3 天，具体看账号情况~"
```

### 6.3 特性

- **可热更新**：改 YAML 调 `engine.reload()`，不重启
- **Fallback**：没匹配到 → 友好引导消息
- **V2 展望**：多轮对话上下文

---

## 7. 数据看板

### 7.1 页面

手机浏览器打开，一眼看清今日数据：

```
┌──────────────────────────────────┐
│  📊 XX工作室 · 今日数据            │
│  （每 60 秒自动刷新）              │
│──────────────────────────────────│
│  💰 今日收入      ¥ 450           │
│  📋 进行中        8 单            │
│  ✅ 今日完成      12 单            │
│  ⚠️ 超时未交      1 单（红色）     │
│  👥 在线打手      5 人            │
│  ⏳ 待分配        3 单            │
│──────────────────────────────────│
│  [查看所有订单] [打手状态] [客户]  │
│  本周收入趋势 ████░░░░  ¥2100     │
└──────────────────────────────────┘
```

### 7.2 技术方案

- 在 `wecom_app.py` 中增加 `/dashboard` 路由，返回纯 HTML
- 数据通过 `/api/dashboard` JSON 获取
- 无需前端框架，移动端原生兼容

---

## 8. 定时任务

### 8.1 依赖

```powershell
pip install apscheduler
```

### 8.2 任务

| 任务 | 频率 | 说明 |
|---|---|---|
| 超时检查 | 每 30 分钟 | `status != 'done' AND deadline < now` → 提醒叶来电 |
| 客户回访 | 每天 10:00 | `last_order_at` 超过 7 天的老客户 → 回访清单 |
| 每日汇总 | 每天 23:00 | 当日数据摘要推送 |

---

## 9. 开发步骤

### Phase 1：数据层（1 天）

| 步骤 | 文件 | 内容 |
|---|---|---|
| 1.1 | `models.py` | Player、Order、Transaction、Client 四个模型 + 建表 |
| 1.2 | `models.py` | 实现 `init_db()` + `get_session()` + CRUD 便捷方法 |

### Phase 2：服务模块（2 天）

| 步骤 | 文件 | 内容 |
|---|---|---|
| 2.1 | `services/price.py` | `cmd_price` → Markdown 价格表 |
| 2.2 | `services/player.py` | 打手 CRUD 全套命令 |
| 2.3 | `services/order.py` | 订单全套命令 |
| 2.4 | `services/client.py` | 客户管理命令 |
| 2.5 | `services/dashboard.py` | `cmd_dashboard` 摘要 |
| 2.6 | `services/finance.py` | 收支命令 |

### Phase 3：意图引擎（1 天）

| 步骤 | 文件 | 内容 |
|---|---|---|
| 3.1 | `rules/intents.yaml` | 关键词规则 |
| 3.2 | `intent_engine.py` | IntentEngine 类 |
| 3.3 | `wecom_app.py` | msg_handler 集成（命令优先，意图兜底） |

### Phase 4：看板 + 定时（1 天）

| 步骤 | 文件 | 内容 |
|---|---|---|
| 4.1 | `templates/dashboard.html` | 手机看板页面 |
| 4.2 | `wecom_app.py` | `/dashboard` + `/api/dashboard` 路由 |
| 4.3 | `reminder.py` | 三个定时任务 |
| 4.4 | `wecom_app.py` | 启动 APScheduler |

### Phase 5：联调（1 天）

| 步骤 | 内容 |
|---|---|
| 5.1 | 本地 curl 测试所有命令 |
| 5.2 | ngrok 穿透 → 企微后台配置回调 |
| 5.3 | 企微群聊实测 |
| 5.4 | 叶来电手机验收 |

---

## 10. 部署指南

### 10.1 本地开发

```powershell
cd D:\demo\xbotsvr
pip install wecom-bot-svr apscheduler pyyaml
python wecom_app.py
```

### 10.2 企微参数获取

1. 登录 [企业微信管理后台](https://work.weixin.qq.com)
2. 应用管理 → 自建应用
3. 获取参数填入 `config.py`：
   - `wecom_corp_id`：我的企业 → 企业 ID
   - `wecom_token`：接收消息 → 随机生成
   - `wecom_aes_key`：接收消息 → 随机生成（43 字符）
   - `wecom_bot_key`：机器人 webhook 地址 `key=` 后面的值
   - `wecom_bot_name`：机器人名称，必须严格一致

### 10.3 ngrok 穿透

```powershell
python wecom_app.py          # 终端 1
ngrok http 5001              # 终端 2
# 把 https 地址填入企微后台回调 URL
```

### 10.4 生产部署

阿里云/腾讯云轻量服务器 2C2G，~50 元/月

```ini
# /etc/systemd/system/delta-agent.service
[Unit]
Description=Delta Agent WeChat Work Bot
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/xbotsvr
ExecStart=/usr/bin/python3 wecom_app.py
Restart=always

[Install]
WantedBy=multi-user.target
```

---

## 11. 使用示例

### 客户询价

```
客户: 你好
机器人: 老板好！三角洲专业代练，段位/通行证/任务/撞车都可接~

客户: 多少钱
机器人: 📋 三角洲代练价格表
       > 青铜 → 黄金：XX 元（1-2天）
       > ...

客户: 下单 黄金打到钻石
机器人: 收到！告诉我当前段位、目标段位、时间要求~
```

### 叶来电操作

```
叶来电: order create 老板A rank_up 黄金→钻石 200 3天
机器人: ✅ 订单 #12 已创建 | 待分配

叶来电: order assign 12 张三
机器人: ✅ 已分配给张三 | 分成 ¥120 | wx_zhangsan

叶来电: dashboard
机器人: 📊 今日收入 ¥450 | 进行中 8单 | 完成 12单 | ⚠️超时 1单
```

### 自动催单

```
机器人（主动推送 → 叶来电）:
⚠️ 超时提醒：
- #8 老板C 钻石→统帅 张三 超时1天
- #15 老板D 通行证 李四 超时0.5天
```

---

## 附录 A：依赖

```
# requirements.txt
wecom-bot-svr         # 企微机器人框架
apscheduler           # 定时任务
pyyaml                # YAML 解析
```

## 附录 B：参考链接

- 基底项目：https://github.com/easy-wx/xbot
- 企微回调框架：https://github.com/easy-wx/wecom-bot-svr
- 企微加解密：https://github.com/sbzhu/weworkapi_python
- 企微机器人文档：https://developer.work.weixin.qq.com/document/path/99399
