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

> 部署、接入与日常维护记录。凭据、Token、密码仅保存在服务器的 `.env` 或认证存储中，**不写入本笔记**。

## 部署环境

- 云服务：阿里云轻量应用服务器
- 配置：2 核 CPU、2 GB 内存、40 GB 磁盘、200 Mbps 带宽
- 系统：Linux（内核 `5.10.134-19.3.2.al8.x86_64`）
- Hermes 安装目录：`/home/admin/.hermes/hermes-agent`
- Python：3.11.15
- 当前 Hermes Agent：v0.20.0（2026.8.3）

## 安装方式

服务器镜像已包含 Hermes 环境。如需在新机器安装，可运行：

```bash
curl -fsSL https://hermes-agent.nousresearch.com/install.sh | bash
```

安装后建议依次执行：

```bash
hermes setup
hermes doctor
hermes --version
```

## 当前已完成配置

### 1. 模型与 Provider

当前会话使用：

- 模型：`gpt-5.6-terra`
- Provider：Custom endpoint

交互式调整模型/Provider：

```bash
hermes model
```

API Key 与其他凭据放在：

```text
/home/admin/.hermes/.env
```

配置项写入 `config.yaml`，不要把 Token 或 API Key 写进笔记或配置文件：

```bash
hermes config path
hermes config env-path
```

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

### 3. MCP：my-note 笔记库

已接入远程 MCP 服务 `my-note`：

- MCP 地址：`https://notalei.zeabur.app/api/mcp`
- Transport：HTTP
- 权限：已启用全部 26 个笔记工具
- Vault：`Obsidian Vault`（ID：1）

检查连接与工具：

```bash
hermes mcp list
hermes mcp test my-note
```

MCP URL、授权头等配置由 Hermes 管理；不要把 Authorization Token 写入笔记。配置变更后应重启 Gateway：

```bash
hermes gateway restart
```

### 4. 草稿自动整理任务

已创建每日定时任务：**每日草稿整理与索引待办**。

- 执行时间：每天 10:00（+08:00）
- 任务内容：读取 `草稿/`，按规范分类；更新相关索引和明确待办；将更新摘要、待办、待确认项发到微信。
- 当前任务 ID：`da1873c43eb3`

管理命令：

```bash
hermes cron list
hermes cron run da1873c43eb3       # 手动测试
hermes cron pause da1873c43eb3     # 暂停
hermes cron resume da1873c43eb3    # 恢复
hermes cron remove da1873c43eb3    # 删除
```

## 建议后续配置

### 必需/优先

- [x] 执行 `hermes doctor`，处理环境和依赖诊断项。
- [x] 通过 `hermes model` 复核主模型、Provider、备用模型以及限额策略。
- [x] 确认 Gateway 在服务器重启后可自动启动：`hermes gateway status`。
- [x] 用 `hermes mcp test my-note` 定期检查笔记库连接和授权是否有效。
- [x] 定期测试每日草稿任务是否能成功发送微信简报：`hermes cron run da1873c43eb3`。

### 可选扩展

- [x] 在 `hermes model` 或 `hermes auth` 配置一个备用 Provider，降低单一 Provider 不可用的风险。
- [ ] 视需要配置 TTS、浏览器、其他消息平台或更多 MCP 服务：`hermes setup`。
- [ ] 通过 `hermes config set approvals.mode smart` 保持智能命令审批；不要长期使用 `off`。
- [ ] 如需查看日志：`hermes logs` 或 `hermes logs errors`。
- [ ] 如需升级前检查状态：`hermes version`；确认兼容性后再执行 `hermes update`。

## 日常维护与排障

```bash
# 状态总览
hermes status --all

# 深度诊断
hermes doctor

# 查看错误日志
hermes logs errors

# 查看 MCP 配置与连通性
hermes mcp list
hermes mcp test my-note

# 查看定时任务
hermes cron list
```

## 安全约定

1. 密钥、Cookie、Authorization Token 和密码仅存放在 `.env`、`auth.json` 或受控的密钥管理工具中。
2. `config.yaml` 用于非敏感配置；优先使用 `hermes config set`，不要手工编辑 YAML。
3. 命令审批保持 `smart`；只有排障或明确需要时才临时使用 `--yolo` 或关闭审批，并在完成后恢复。
4. 对 MCP 或笔记库执行删除、移动、批量修改前，遵循 [[AI 笔记规范]] 的确认要求。
5. 修改配置后，必要时运行 `hermes gateway restart`，然后用 `hermes status --all` 验证。

## 参考

- [[AI 笔记规范]]
- [[笔记库使用说明]]
- Hermes 官方文档：https://hermes-agent.nousresearch.com/docs
