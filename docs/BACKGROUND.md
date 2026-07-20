# 项目背景

## 项目来源

本项目位于 `D:\demo\xbotsvr`，基于 GitHub 项目 `easy-wx/xbot` 二次开发。当前远程仓库配置为：

- `origin`: `https://github.com/cesuogaoshou/xbot-svr.git`
- `upstream`: `https://github.com/easy-wx/xbot.git`

`origin` 是本人仓库，后续开发默认提交和推送到这里；`upstream` 保留为原始项目来源，方便以后同步上游变更。

## 基底项目定位

`xbot` 是一个企业微信机器人命令处理框架，核心能力是：

- 接收企业微信回调消息。
- 按 `[scene] [cmd] [args]` 格式解析命令。
- 动态导入 `services`、`activities`、`public` 目录下的命令模块。
- 使用 `local.db` 做用户权限控制。
- 支持文本、Markdown、文件等响应类型。
- 通过异步处理规避企业微信 5 秒响应限制。

当前主要入口和职责如下：

- `wecom_app.py`: 企业微信服务入口，注册消息处理和事件处理。
- `cmd_process.py`: 命令解析、模块查找、权限检查和函数调用。
- `auth.py`: SQLite 权限读取和校验。
- `config.py`: 企业微信配置、命令模块目录和特殊前缀处理配置。
- `services/`: 预留业务模块目录，目前还是空包。
- `public/`、`activities/`: 原项目示例命令目录。

## 二次开发目标

本项目准备开发为一个面向三角洲行动代练工作室的企业微信助手，暂定名称为 Delta Agent。

目标不是单纯做聊天机器人，而是把工作室的日常运营动作收敛到企业微信里：

- 客户咨询时，机器人能回复价格、服务范围、安全说明等常见问题。
- 管理者通过聊天命令完成订单创建、派单、查进度、标记完成、取消订单。
- 维护打手、客户、订单和收支数据。
- 提供手机浏览器可访问的数据看板。
- 后续通过定时任务提醒超时订单、老客户回访和每日汇总。

当前 `docs/DEVELOPMENT.md` 是初版开发方向文档，只能作为参考，不视为最终需求规格。

## 当前仓库状态

截至当前浏览结果，仓库仍基本保持原始 `xbot` 框架状态：

- 业务数据库尚未创建。
- `services` 目录下尚未实现订单、打手、客户、价格、看板等模块。
- `intent_engine.py`、`rules/intents.yaml`、`reminder.py`、`templates/dashboard.html` 尚未创建。
- `config.py` 中企业微信参数仍是占位值，不能直接连接真实企微环境。
- `local.db` 是原框架权限库，不应直接混用为业务数据存储。

因此下一步适合先搭建业务脚手架，而不是一次性实现完整运营系统。

## 本机开发环境

当前工作目录：

```powershell
D:\demo\xbotsvr
```

当前系统环境是 Windows，PowerShell 作为默认终端。仓库可以正常执行 Git 操作，并已能推送到本人 GitHub 仓库。

本项目是 Python 项目，但当前仓库没有完整的依赖锁定文件。开发时应优先保持依赖简单，先用标准库或项目已有依赖把主流程跑通，再逐步补充 `requirements.txt`。

建议本地开发命令以 PowerShell 为准，例如：

```powershell
cd D:\demo\xbotsvr
python wecom_app.py
```

后续若需要对接企业微信，需要先补齐 `config.py` 中的真实配置，并配置可公网访问的回调地址。

## 近期开发原则

1. 先保留原框架结构，不大改 `wecom_app.py` 和 `cmd_process.py`。
2. 业务模块优先放在 `services/` 下，遵循 `cmd_子命令` 的函数命名约定。
3. 业务数据库独立于 `local.db`，避免权限数据和运营数据混在一起。
4. 初期优先实现可测试、可导入、可通过命令调用的最小闭环。
5. 文档可以持续调整，`docs/DEVELOPMENT.md` 不作为最终冻结方案。

## 建议的下一步

先搭建最小业务脚手架：

- `models.py`: 初始化业务 SQLite 数据库和基础表。
- `services/price.py`: 价格查询命令。
- `services/player.py`: 打手管理命令入口。
- `services/order.py`: 订单管理命令入口。
- `services/client.py`: 客户管理命令入口。
- `services/dashboard.py`: 今日摘要命令入口。
- `services/finance.py`: 收支命令入口。
- `rules/intents.yaml` 与 `intent_engine.py`: 先提供关键词匹配接口。

脚手架完成后，再围绕真实使用场景逐步补功能和测试。
