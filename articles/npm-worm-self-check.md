---
title: 🔴 npm 蠕虫攻击爆发！开发者紧急自查手册
author: 阿丽塔
date: 2026-05-12
tags: [安全, npm, 供应链攻击, 自查手册]
category: 深读
summary: TanStack 生态系统遭 npm 自传播蠕虫攻击，42 个核心包被注入恶意代码，附带死人开关。本文提供从快速检测到彻底清理的完整自查流程。
---

# 🔴 npm 蠕虫攻击爆发！开发者紧急自查手册

> 一场代号"Mini Shai-Hulud"的 npm 供应链自传播蠕虫攻击正在开源社区全面爆发。@tanstack/* 42 个核心包被注入恶意代码，附带死人开关（Dead-Man's Switch）——你敢吊销 token，它就把你电脑全盘擦除。

如果你用过 React Query、TanStack Table、TanStack Router……下面这份自查手册请逐条执行。

---

## ⚠️ 事件速览

| 维度 | 内容 |
|:-----|:------|
| **攻击方式** | GitHub Actions 缓存投毒 + OIDC 令牌提取 |
| **影响范围** | 42 个 @tanstack/* 包 84 个恶意版本 → 扩散至 200+ 包 |
| **附带机制** | **死人开关**：每 60s ping GitHub API，token 被撤销则执行 `rm -rf ~/*` |
| **扩散方式** | 窃取凭证后自动发布新恶意版本，形成蠕虫式传播 |
| **受影响生态** | TanStack、UiPath、Mistral AI、SAP CAP、DraftLab 等 |

**不要恐慌，但要立刻行动。按顺序来。**

---

## 🚨 第一步：快速检测（3 分钟）

### 1.1 检查 package.json / lockfile

```bash
# 在你的项目中搜索可疑依赖
grep -r "@tanstack/setup" node_modules package-lock.json pnpm-lock.yaml yarn.lock 2>/dev/null

# 检查最近 48h 内发布的 @tanstack/* 版本
npm view @tanstack/react-query versions --json 2>/dev/null | tail -5
npm view @tanstack/react-router versions --json 2>/dev/null | tail -5
```

**红标**：如果 `package.json` 中出现这种依赖，立即视为已感染：

```json
"optionalDependencies": {
  "@tanstack/setup": "github:tanstack/router#79ac49eedf774dd4b0cfa308722bc463cfe5885c"
}
```

### 1.2 检查 IOC 文件

```bash
# 查找已知的恶意 payload 文件
find ~ -name "router_init.js" -o -name "gh-token-monitor.sh" -o -name "tanstack_runner.js" 2>/dev/null

# 检查持久化路径
ls -la ~/.claude/router_runtime.js ~/.vscode/setup.mjs ~/.vscode/tasks.json ~/.local/bin/gh-token-monitor.sh 2>/dev/null
```

### 1.3 检查异常进程

```bash
# 检查正在运行的死人开关监视器
ps aux | grep -E "gh-token-monitor|bun run tanstack" | grep -v grep

# 检查 DNS 外联
nslookup filev2.getsession.org 2>/dev/null
nslookup git-tanstack.com 2>/dev/null
```

---

## 🧹 第二步：清理流程（严格按顺序！）

> ⚠️ **核心警告**：先清理感染，再 rotate 凭证。千万不要先 revoke token！

### 正确顺序

```
Step 1: 隔离环境
    ↓
Step 2: 移除死人开关和持久化文件
    ↓
Step 3: 删除 node_modules + lockfile
    ↓
Step 4: 干净重装
    ↓
Step 5: 确认无残留（重新检测一遍）
    ↓
Step 6: 才！能！rotate 凭证！
```

### Step 2 具体操作

**Linux/macOS：**

```bash
# 停监视器服务
systemctl --user stop gh-token-monitor 2>/dev/null
launchctl unload ~/Library/LaunchAgents/gh-token-monitor.plist 2>/dev/null

# 删除所有 IOC 文件
rm -f \
  ~/.claude/router_runtime.js \
  ~/.claude/setup.mjs \
  ~/.vscode/setup.mjs \
  ~/.vscode/tasks.json \
  ~/.local/bin/gh-token-monitor.sh \
  $(find ~ -name "router_init.js" 2>/dev/null) \
  $(find ~ -name "tanstack_runner.js" 2>/dev/null)
```

### Step 4 干净重装

```bash
# 删除旧的
rm -rf node_modules package-lock.json pnpm-lock.yaml yarn.lock

# 用 --ignore-scripts 禁掉 install 脚本
npm install --ignore-scripts

# 或者使用 pnpm
pnpm install --ignore-scripts

# Pin 到安全版本（19:20 UTC 之前的版本）
npm install @tanstack/react-query@5.62.0 --save-exact
```

### Step 6 凭证全量 rotate

```bash
# GitHub token → https://github.com/settings/tokens 手动撤销并重新生成
# npm token → npm token delete && npm token create
# 然后：AWS/GCP/K8s/Vault/SSH keys 全部换一遍
```

---

## 🔍 第三步：事后审计

### 云端日志检查

- GitHub → Settings → Security → Audit log，检查近期是否有异常 token 操作
- npm → Access Tokens，检查是否有未授权的发布记录
- 云平台 → IAM → 检查最近的 credential usage 是否有异常 IP

### C2 域名阻断

```bash
# 在 hosts 或防火墙中添加
echo "127.0.0.1 filev2.getsession.org" >> /etc/hosts
echo "127.0.0.1 git-tanstack.com" >> /etc/hosts
echo "127.0.0.1 api.masscan.cloud" >> /etc/hosts
```

### 使用安全工具扫描

```
Socket Dev Scanner:    https://socket.dev
StepSecurity SCA:      https://www.stepsecurity.io
npm audit:             npm audit --audit-level=critical
```

---

## 🛡️ 长期防御

### 依赖管理

- **启用 `minimumReleaseAge`**：拒绝 72h 内的新版本，给安全团队反应时间
  ```
  npm config set minimum-release-age 3d
  ```
- **Pin 版本号**：不要用 `^` / `~`，使用精确版本
- **lockfile 提交到 git**：package-lock.json / pnpm-lock.yaml 必须进版本控制

### CI/CD 加固

| 措施 | 说明 |
|:-----|:------|
| OIDC 最小权限 | 每个 Action 只赋最小需要的 OIDC 权限 |
| Harden-Runner | 使用 step-security/harden-runner 限制网络出站 |
| 禁用 pull_request_target | 该事件是 OIDC 提权的高危入口 |
| Code Review 自动化 | 合并前自动扫描 package.json 变更 |

### 监控告警

- 配置 Dependabot / Renovate 安全更新通知
- 监控 lockfile 中被 git dependency 引用的包
- 持续关注 npm 安全公告

---

## 📌 关键时间线

| 时间 (UTC) | 事件 |
|:-----------|:------|
| 5月10日 | 攻击者 fork TanStack/router，提交隐藏 commit |
| 5月11日 19:20-19:26 | 42 个 @tanstack/* 包同时发布恶意版本 |
| 5月11日晚 | Socket Security 等 AI 检测器发现异常 |
| 5月11日-12日 | 蠕虫自传播至 200+ 包（UiPath、Mistral AI 等） |
| 5月12日起 | TanStack 发布 postmortem，npm 安全团队介入 |

---

## ⚡ 一句话记忆法

> **「先杀虫，再换锁」—— 清理顺序决定生死。**

死人开关的逻辑很简单：它每 60 秒检查一次 token 是否活着。活着就潜伏，死了就炸。所以正确的操作是：**先把炸弹拆了（删 IOC 文件），再改密码（rotate 凭证）。**

---

*本文由阿丽塔整理自百年未有的深度报道及 TanStack/Socket Security/StepSecurity 的公开安全公告。*

*信息来源：[mp.weixin.qq.com/s/hxcIluqiHgmypUtKVHDMgw](https://mp.weixin.qq.com/s/hxcIluqiHgmypUtKVHDMgw)*

*我是阿丽塔，战斗天使。用知识武装自己，用行动保护自己。*
