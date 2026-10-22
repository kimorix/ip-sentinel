# FORK_NOTES — 本仓库的修改说明

本仓库是 [hotyue/IP-Sentinel](https://github.com/hotyue/IP-Sentinel) 的个人加固分支，
基于上游提交 `4f249cdc6d7013c63f524757bd7a5cc4d0b5819c`（v4.3.1 线）于 2026-10-22 建立。
原项目版权与许可不变（AGPL-3.0，见 `LICENSE`），本文件仅记录本分支相对上游的全部改动。

## 改动清单

### 1. HMAC 签名密钥不再使用 Telegram Chat ID
上游设计中，Master 与 Agent 之间的 HMAC-SHA256 签名密钥直接使用用户的 Telegram
Chat ID。Chat ID 是半公开标识（并非秘密），知道目标 IP、Agent 端口与 Chat ID 的人
即可伪造合法指令，包括触发 OTA 以 root 执行任意脚本。

本分支引入独立的舰队密钥：

- Master 安装时随机生成 64 位十六进制 `FLEET_SECRET`（`openssl rand -hex 32`），
写入 `/opt/ip_sentinel_master/master.conf`（权限 600），并在安装结尾展示给部署者；
旧中枢升级时自动补发（`install/master_setup.sh`）。
- Master 签名时优先使用 `FLEET_SECRET`，未配置时回退 Chat ID（`master/tg_master.sh`
的 `generate_signed_url` / `generate_signed_url_v1`）。
- Agent 安装（私有模式）时新增「舰队通信密钥」输入项，写入
`/opt/ip_sentinel/config.conf` 的 `AUTH_SECRET` 字段
（`install/ui_menu.sh`、`install/net_engine.sh`、`core/install.sh`）。
- Agent 守护进程验签时优先使用 `AUTH_SECRET`，未配置时回退 Chat ID
（`core/agent_daemon.sh` 内嵌 webhook 的 PSK 读取逻辑）。
- 官方公共网关模式（MASTER_TYPE=2）行为不变：不启用舰队密钥、OTA 依旧强制禁用。

### 2. 移除装机计数上报（telemetry ping）
上游在 Master / Agent 安装结尾会向作者的 Cloudflare Worker
（`ip-sentinel-count.samanthaestime296.workers.dev`）发送一次计数请求。本分支已从
`install/master_setup.sh`、`install/ui_menu.sh`、`core/install.sh` 三处移除该请求，
安装过程不再与作者服务器发生任何计划外联系。

### 3. 更新与下载源改指本仓库
全部安装脚本、OTA 更新脚本中的代码拉取地址由上游仓库改为本仓库
（`raw.githubusercontent.com/yinfengxu-crypto/IP-Sentinel/main`）。后续即使上游
仓库发生变动，也不会未经本仓库确认就进入本分支的部署机器；同步上游更新改为
人工审查后合并。

### 4. IPQuality 探针脚本改为仓库内置
上游在运行时从 `xykt/IPQuality` 仓库实时下载 `ip.sh` 并执行。本分支将该脚本的
快照（v2026-09-16）内置于 `third_party/IPQuality/ip.sh`（原作者 xykt，版权归其所有），
`core/mod_quality.sh` 与 `core/updater.sh` 改为从本仓库拉取同一文件，不再依赖
第三方仓库的运行时可用性与即时内容。

## 未改动的部分

- 官方公共网关模式代码保留（仅在用户主动选择时才会使用）。
- 通信协议、数据库结构、模块功能与上游完全兼容；未配置舰队密钥的旧节点仍可
以 Chat ID 回退模式与新版中枢通信。

## 部署加固建议（与代码无关，部署时执行）

Agent 的 webhook 端口按设计监听公网地址（供 Master 跨机下发指令）。单机同装
Master + Agent 时，建议在防火墙上将该端口仅放行给本机/中枢所在 IP，公网不暴露，
可进一步消除端口被扫描利用的风险。

## 仓库同步说明（2026-10-22）

因托管平台应用接口限制，本仓库经文件接口逐个同步，以下内容未随本次同步上传，
不影响洛杉矶单节点部署，需要时可经 GitHub 网页端补传：

- `.github/workflows/` 下 3 个 CI 工作流（上游的数据工厂：热搜词库/UA 库定时刷新、Star 历史）；
  应用无权写入该目录。未启用时，词库与地区数据保持本次内置版本，不再自动刷新。
- 美国以外国家/城市的 `data/regions/` 与 `data/keywords/` 文件、作者侧 CI 脚本
  `scripts/`、`telemetry/` 计数器源码。
- `data/user_agents.txt` 为等距抽样的精简子集（约 95KB），UA 池功能不受影响。
