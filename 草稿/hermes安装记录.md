---
类型: 知识
状态: 有效
tags:
  - hermes
  - ai
  - vps
来源: 个人纪录
创建日期: 2026-08-07
更新日期: 2026-08-07
---

# Hermes 安装与配置记录

> **用途**：记录本机 Hermes 的实际部署状态、可复用配置流程和日常运维入口。命令以当前安装的 Hermes CLI 为准；升级后先执行 `hermes --help` 或相应子命令的 `--help` 核对参数。

## 部署环境

| 项目 | 当前值 |
| --- | --- |
| 云服务 | 阿里云轻量应用服务器 |
| 规格 | 2 核 CPU、2 GB 内存、40 GB 磁盘、200 Mbps 带宽 |
| 系统 | Linux（内核 `5.10.134-19.3.2.al8.x86_64`） |
| Hermes 安装目录 | `/home/admin/.hermes/hermes-agent` |
| Python | 3.11.15 |
| Hermes Agent | v0.20.0（2026.8.3） |
| 配置目录 | `~/.hermes/`（以 `hermes config path` 输出为准） |
| 密钥文件 | `/home/admin/.hermes/.env`（以 `hermes config env-path` 输出为准） |

## 配置文件与安全边界

| 内容 | 建议位置 | 说明 |
| --- | --- | --- |
| 非敏感运行配置 | `config.yaml` | 通过 `hermes config set` 写入，避免手工改 YAML。 |
| API Key、Token、Cookie、密码 | `.env`、`auth.json` 或密钥管理工具 | 仅保存在服务器受控路径，不能提交到 Git 或复制到笔记。 |
| MCP 服务地址与授权 | Hermes 的 MCP 配置 | 地址可记录；授权头和 Token 不记录明文。 |
| 定时任务定义 | Hermes Cron 配置 | 修改后需实际手动运行一次验证。 |

先确认本机实际路径与当前有效配置：

```bash
hermes config path
hermes config env-path
hermes status --all
```

> `.env`、`auth.json` 等敏感文件应限制为所有者可读写，例如：`chmod 600 ~/.hermes/.env`。在编辑或备份前，先确认备份目标为加密或受控位置。

## 安装与首次初始化

### 新机器安装

服务器镜像已包含 Hermes 环境。如需在新机器部署：

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
hermes --version
hermes setup
hermes doctor
```

`hermes setup` 用于交互式完成基础初始化；每一步仅填入当前需要启用的 Provider、消息通道或扩展。安装完成后不要直接导入旧机器的整个 `~/.hermes/` 目录，应只迁移经过核验的**非敏感配置**，并在新机重新注入密钥。

### 推荐初始化顺序

1. 运行 `hermes doctor`，修复系统依赖、权限或服务诊断项。
2. 配置一个主 Provider 与模型，并进行一次最小对话测试。
3. 配置消息通道，确认收发正常。
4. 添加 MCP 服务，验证工具列表和最小读操作。
5. 创建或恢复 Cron 任务，先手动运行再启用定时执行。
6. 重启 Gateway，并用 `hermes status --all` 完整验收。

## 当前已完成配置

### 1. 模型与 Provider

当前会话使用：

- 模型：`gpt-5.6-terra`
- Provider：Custom endpoint

交互式查看或调整模型、Provider、备用模型和限额策略：

```bash
hermes model
hermes auth
```

配置原则：

- 凭据仅写入 `.env` 或 Hermes 认证存储；不要写进 `config.yaml`、Shell 历史、截图或笔记。
- 自定义 Endpoint 应使用 HTTPS，并记录服务用途，不记录包含密钥的完整 URL。
- 至少保留一个经过测试的备用 Provider；主 Provider 不可用时切换后应发送一次测试消息。
- 变更后用 `hermes status --all` 确认当前模型与认证状态；若 Gateway 未自动加载，再执行重启。

### 2. 微信（Weixin）消息通道

- Weixin 已配置并可用。
- Gateway 由 systemd 用户服务管理，当前处于运行状态。

常用运维命令：

```bash
hermes gateway status
hermes gateway restart
hermes gateway stop
hermes gateway start
```

配置或重新授权时，通过以下入口检查当前 CLI 支持的通道选项，而不是猜测配置字段：

```bash
hermes setup
hermes --help
hermes gateway --help
```

验收清单：

- [ ] 能从微信向 Agent 发送一条普通文本消息。
- [ ] Agent 的回复能正常回传微信。
- [ ] 重启 Gateway 后仍可收发。
- [ ] 服务器重启后，`hermes gateway status` 显示服务自动恢复。

### 3. MCP：my-note 笔记库

已接入远程 MCP 服务 `my-note`：

| 项目 | 当前值 |
| --- | --- |
| MCP 地址 | `https://notalei.zeabur.app/api/mcp` |
| Transport | HTTP |
| 权限 | 已启用全部 26 个笔记工具 |
| Vault | `Obsidian Vault`（ID：1） |

检查连接与工具：

```bash
hermes mcp list
hermes mcp test my-note
```

配置变更后重启 Gateway：

```bash
hermes gateway restart
```

权限控制建议：

- 新增 MCP 时先授予最小权限，确认用途后再扩大范围。
- 对笔记库的删除、移动、批量修改操作保留审批；执行前先明确作用范围与回滚方式。
- MCP URL、Authorization Header 和 Token 均由 Hermes 的受控配置保存，不在本笔记中留明文。

### 4. 草稿自动整理任务

已创建每日定时任务：**每日草稿整理与索引待办**。

| 项目 | 当前值 |
| --- | --- |
| 执行时间 | 每天 10:00（+08:00） |
| 任务 ID | `da1873c43eb3` |
| 任务行为 | 读取 `草稿/`，按规范分类；更新相关索引和明确待办；将更新摘要、待办、待确认项发到微信。 |

管理命令：

```bash
hermes cron list
hermes cron run da1873c43eb3       # 手动测试
hermes cron pause da1873c43eb3     # 暂停
hermes cron resume da1873c43eb3    # 恢复
hermes cron remove da1873c43eb3    # 删除
```

调整任务前的操作规范：

1. 先用 `hermes cron list` 记录原任务 ID、执行频率和任务内容。
2. 修改任务提示词时明确输入目录、允许的写入范围、禁止操作和通知渠道。
3. 先执行 `hermes cron run <任务ID>`，检查笔记变更与微信通知。
4. 成功后再恢复/保留定时执行；失败时暂停任务并查看日志。

## 配置变更标准流程

适用于模型、消息通道、MCP 和定时任务：

```bash
# 1. 变更前：检查状态
hermes status --all

# 2. 按对应入口修改（示例：模型）
hermes model

# 3. 重新加载服务
hermes gateway restart

# 4. 验收与诊断
hermes gateway status
hermes status --all
hermes logs errors
```

若变更涉及 MCP 或 Cron，还应额外执行：

```bash
hermes mcp test my-note
hermes cron run da1873c43eb3
```

## 日常维护与排障

```bash
# 状态总览与深度诊断
hermes status --all
hermes doctor

# 日志（先看错误；需要时再查看完整日志）
hermes logs errors
hermes logs

# MCP
hermes mcp list
hermes mcp test my-note

# 定时任务
hermes cron list

# 升级前后
hermes version
hermes update
hermes doctor
```

### 常见排查顺序

1. **消息无响应**：`hermes gateway status` → `hermes logs errors` → 重启 Gateway → 从微信发送测试消息。
2. **模型调用失败**：检查 Provider 状态与 `.env` 中密钥是否仍有效；用 `hermes model` 核对模型选择；避免在日志中粘贴密钥。
3. **MCP 不可用**：`hermes mcp list` → `hermes mcp test my-note` → 检查网络、远端服务和授权是否过期 → 重启 Gateway。
4. **定时任务未执行/未通知**：`hermes cron list` → 手动 `run` 测试 → 查看错误日志 → 核对时区、Gateway 状态及微信通道。
5. **升级后异常**：记录当前版本与报错，先运行 `hermes doctor`；必要时暂停 Cron，确认模型、通道和 MCP 三项核心能力后再恢复任务。

## 备份、升级与恢复

- 每次调整模型、通道、MCP、Cron 或升级 Hermes 后，更新本记录的版本、日期和变更摘要。
- 备份时仅导出非敏感配置、任务说明和恢复步骤；密钥单独存入加密密码库或受控密钥管理系统。
- 升级前执行 `hermes version`、`hermes status --all` 和一次 `hermes cron run da1873c43eb3`，确认基线正常。
- 升级后执行 `hermes doctor`，并重新验证：微信收发、`hermes mcp test my-note`、草稿任务手动运行。
- 出现严重故障时，优先暂停会写入笔记的定时任务，保留日志与配置快照，再进行恢复或回退。

## 安全约定

1. 密钥、Cookie、Authorization Token 和密码仅存放在 `.env`、`auth.json` 或受控密钥管理工具中。
2. `config.yaml` 仅保存非敏感配置；优先使用 `hermes config set`，不要手工编辑 YAML。
3. 命令审批保持 `smart`：`hermes config set approvals.mode smart`。仅在明确的临时排障中使用 `--yolo` 或关闭审批，完成后立即恢复。
4. 对 MCP 或笔记库执行删除、移动、批量修改前，遵循 [[AI 笔记规范]] 的确认要求。
5. 修改配置后，必要时运行 `hermes gateway restart`，然后用 `hermes status --all` 验证。
6. 不在聊天记录、终端历史、日志摘录或此笔记中粘贴密钥；如发生泄露，立即在 Provider/MCP 端轮换凭据并更新服务器配置。

## 维护待办

- [ ] 记录 Gateway 的实际 systemd 用户服务名称及查看日志的命令。
- [ ] 为主模型和备用 Provider 补充最后一次验证日期。
- [ ] 每月手动运行一次草稿任务并检查微信简报、笔记写入范围与 MCP 授权。
- [ ] 每次 Hermes 升级后更新本页顶部的版本信息和本节验收结果。

## 参考

- [[AI 笔记规范]]
- [[笔记库使用说明]]
- Hermes 官方文档：https://hermes-agent.nousresearch.com/docs
