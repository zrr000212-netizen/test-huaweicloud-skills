# GitCode→GitHub 代码增量同步 特性设计文档

| 项目 | 内容 |
|------|------|
| 特性名称 | GitCode→GitHub 跨仓库代码增量同步 |
| 文档版本 | v1.0 |
| 业务场景 | 多平台协作：GitCode 作为开发主仓，GitHub 作为协作镜像，实现单向增量同步 |

---

## 1. 概述

在多平台协作场景下，开发团队在 GitCode 平台进行日常开发，同时需要将代码同步至 GitHub 供外部协作、开源发布或合规审计。本特性实现 GitCode→GitHub 的单向增量同步，核心能力：

- **增量同步**：仅同步新增 commit，避免全量重复推送
- **身份隔离**：隐藏 GitCode 原始提交人信息，统一替换为指定账号
- **断点续传**：冲突或异常中断后，可从断点继续而非从头开始
- **跨机器恢复**：同步状态持久化到 Git 仓库，新机器可接续同步

---

## 2. 架构设计

### 2.1 模块划分

系统按职责划分为 4 个核心模块，边界清晰、单一职责：

```
┌─────────────────────────────────────────────────────┐
│                   sync-all.sh                        │
│              批量调度层（Orchestrator）                │
│  职责：读取配置 → 逐行解析 → 调度同步 → 汇总结果       │
└──────────────────────┬──────────────────────────────┘
                       │ 调用
┌──────────────────────▼──────────────────────────────┐
│            gitcode-to-github-sync.sh                 │
│              核心同步层（Sync Engine）                  │
│  职责：增量检测 → cherry-pick → 推送 → 状态更新        │
├─────────────────────────────────────────────────────┤
│  子模块：                                            │
│  ├── 状态管理（State）   state.json / mapping.log    │
│  ├── 锁机制（Lock）      sync.lock + PID 校验        │
│  └── 日志管理（Log）     sync.log + 滚动压缩          │
└──────────────────────┬──────────────────────────────┘
                       │ 读写
┌──────────────────────▼──────────────────────────────┐
│                  states/                             │
│              状态持久化层（Persistence）                │
│  职责：承载同步状态、commit 映射、运行日志              │
└─────────────────────────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│              sync-repos.conf                         │
│              配置层（Configuration）                   │
│  职责：声明同步任务、默认值                            │
└─────────────────────────────────────────────────────┘
```

**分层原则**：
- **调度层**不感知同步细节，仅做配置解析和结果汇总
- **同步层**不感知批量调度，单次同步自包含
- **持久化层**不感知业务语义，仅提供 key-value 和 append-only 存储
- **配置层**与逻辑完全解耦，格式变更不影响同步逻辑

### 2.2 可扩展性

| 扩展点 | 当前实现 | 扩展方式 |
|--------|---------|---------|
| 新增同步源/目标 | 配置文件添加一行 | `sync-repos.conf` 声明式配置，无需改代码 |
| 状态存储后端 | Git 仓库 | `init_state_repo()` 封装了状态初始化，替换存储后端仅需修改此函数 |
| 冲突处理策略 | 暂停等待人工 | cherry-pick 失败后脚本退出，可在此处插入自动合并策略 |
| 同步触发方式 | cron 定时 | `sync-all.sh` 为纯 CLI，可被任何触发器调用（Webhook、CI Pipeline） |
| 日志输出目标 | 文件 | `log()` 函数统一入口，可扩展为 syslog/HTTP 回调 |

### 2.3 技术选型

| 选型决策 | 选择 | 依据 |
|---------|------|------|
| 实现语言 | Bash + Python3（仅 json 操作） | Git 操作天然 shell 友好；Python3 仅用于 JSON 读写，避免 shell 下 JSON 处理的脆弱性；无额外依赖 |
| 同步策略 | cherry-pick 逐个同步 | 保留线性历史、不引入合并提交、可逐个处理冲突、断点续传 |
| 状态存储 | Git 仓库 | 跨机器可恢复、零运维成本、版本化可回溯、与项目基础设施一致 |
| 进程互斥 | 文件锁 + PID 校验 | 可移植、无外部依赖 |
| 配置格式 | 管道分隔文本 | 无需解析器依赖，字段数少足够清晰 |
| 提交人替换 | `--author` + `GIT_COMMITTER_*` | 同时替换 author 和 committer，确保 GitHub 端不泄露原始身份 |

---

## 3. 接口设计

### 3.1 核心同步脚本接口

```
gitcode-to-github-sync.sh \
  --gitcode-repo    <gitcode_ssh_url>      # 必填：源仓库 SSH 地址
  --gitcode-branch  <branch_name>          # 必填：源分支名
  --github-repo     <github_ssh_url>       # 必填：目标仓库 SSH 地址
  --github-branch   <branch_name>          # 必填：目标分支名
  --author-name     <name>                 # 必填：替换后的提交人姓名
  --author-email    <email>                # 必填：替换后的提交人邮箱
  [--state-dir      <local_path>]          # 可选：本地状态目录（默认自动计算）
  [--state-repo     <git_ssh_url>]         # 可选：状态持久化仓库（默认内置地址）
```

**接口契约**：
- **必填参数校验**：启动时检查 6 个必填参数，缺失即报错退出
- **幂等性**：相同参数重复运行，若无新 commit 则跳过（`up_to_date`），不产生副作用
- **退出码语义**：0=同步成功/已是最新，1=冲突/参数错误/锁冲突
- **无隐式依赖**：不依赖环境变量（除 Git SSH 配置），所有输入通过参数显式传入

### 3.2 批量调度脚本接口

```
sync-all.sh [--dry-run]
```

- `--dry-run`：仅打印待同步仓库列表，不执行同步

**配置文件接口**（`sync-repos.conf`）：

```
gitcode_repo | gitcode_branch | github_repo | github_branch | [author_name] | [author_email] | [state_repo]
```

- 管道 `|` 分隔，可选字段留空使用默认值
- `#` 开头为注释，空行跳过
- 默认值在 `sync-all.sh` 中集中定义（`DEFAULT_AUTHOR_NAME`、`DEFAULT_AUTHOR_EMAIL`、`DEFAULT_STATE_REPO`）

### 3.3 状态接口

**state.json**（结构化状态）：

```json
{
  "gitcode_repo": "",
  "gitcode_branch": "",
  "github_repo": "",
  "github_branch": "",
  "author_name": "",
  "author_email": "",
  "last_synced_commit": "",
  "last_synced_time": "",
  "total_commits_synced": 0,
  "sync_count": 0,
  "status": "initialized|up_to_date|synced|conflict"
}
```

- 读写通过 `get_state()` / `set_state()` 封装，外部不直接操作 JSON
- `status` 枚举值定义清晰，驱动后续处理逻辑

**mapping.log**（commit 映射）：

```
SRC_HEAD: <最新已同步的源commit SHA>
SYNC_TIME: <同步时间>
<原始SHA> -> <新SHA>  <同步时间>
```

- 首行 `SRC_HEAD` 为增量同步起始点，原子更新（先写临时文件再 mv）
- 映射记录去重（`append_mapping` 中 `grep -q` 校验）

---

## 4. 流程设计

### 4.1 单仓库同步流程（8 步）

```
┌──────────────────────────────────────────────────────────────┐
│ Step 1: 准备本地仓库                                         │
│   已有 → 复用 + 更新 remotes                                 │
│   无有 → clone GitHub → 添加 gitcode remote                  │
│                                                              │
│ Step 2: Fetch 源和目标                                       │
│   git fetch gitcode <branch>                                 │
│   git fetch github <branch>                                  │
│                                                              │
│ Step 3: 确保目标分支存在                                     │
│   存在 → checkout -B                                         │
│   不存在 → 创建孤儿分支（orphan branch）                      │
│                                                              │
│ Step 4: 确定增量起始点                                       │
│   mapping.log 有记录 → 校验有效性 → 增量同步                  │
│   无记录 / 记录过期 → 全量同步（从根 commit 开始）            │
│                                                              │
│ Step 5: 获取待同步 commit 列表                               │
│   git rev-list --reverse --ancestry-path                     │
│   保证只取源分支直系 commit，不取 merge 进来的旁支             │
│                                                              │
│ Step 6: 逐个 cherry-pick                                     │
│   普通 commit → cherry-pick --no-commit                      │
│   merge commit → cherry-pick --no-commit -m 1                │
│   冲突 → 持久化状态 → 退出（人工解决后续传）                  │
│   成功 → 替换 author/committer → 记录映射                     │
│                                                              │
│ Step 7: 推送到 GitHub                                        │
│   git push github <branch>:<branch>                          │
│                                                              │
│ Step 8: 更新状态                                             │
│   update_src_head → set_state → push_state_to_repo           │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 批量同步流程

```
读取 sync-repos.conf
  │
  ├── 跳过空行和注释
  ├── 解析管道分隔字段，空字段填充默认值
  │
  ▼
逐行调用 gitcode-to-github-sync.sh
  │
  ├── 成功 → SUCCESS++
  ├── 失败 → FAIL++（继续下一个，不中断批量）
  │
  ▼
输出汇总：总计 / 成功 / 失败
失败数 > 0 → exit 1
```

### 4.3 冲突恢复流程

```
脚本因冲突退出
  │
  ▼
人工介入：
  cd <work_dir>/<REPO_ID>/repo
  git add -A
  git commit --allow-empty
  │
  ▼
重新运行脚本（相同参数）
  │
  ▼
脚本从 mapping.log 读取 SRC_HEAD → 跳过已同步 commit → 从冲突 commit 继续cherry-pick
```

### 4.4 状态持久化流程

```
同步完成 / 冲突退出 / 已是最新
  │
  ▼
push_state_to_repo()
  │
  ├── 滚动压缩：mapping.log / state.json / sync.log 超 10MB → gzip 归档
  ├── sync.log 截断：保留最近 5000 行
  ├── git add -A → git commit → git push
  │
  ▼
状态仓库更新（跨机器可通过 git pull 获取最新状态）
```

---

## 5. 数据模型

### 5.1 仓库标识

```
REPO_ID = MD5(gitcode_repo + gitcode_branch + github_repo + github_branch)[0:12]
```

- 用途：区分不同同步任务的工作目录和状态目录
- 示例：`c3a0576c2780`、`b54f0fb5f705`

### 5.2 目录结构

```
sync-repository/                          # 状态仓库根目录
├── sync-repos.conf                       # 同步配置
├── sync-all.sh                           # 批量调度入口
├── gitcode-to-github-sync.sh             # 核心同步脚本
├── sync-all.log                          # 批量同步日志
├── states/                               # 状态持久化目录
│   └── <REPO_ID>/                        # 按仓库标识隔离
│       ├── state.json                    # 结构化状态
│       ├── mapping.log                   # commit 映射 + SRC_HEAD
│       ├── sync.log                      # 同步运行日志
│       ├── mapping-20260618.log.gz       # 滚动压缩归档
│       └── sync.lock                     # 进程锁文件
└── .gitignore                            # 忽略锁文件和 vim 临时文件

<work_dir>/                   # 工作目录（临时）
└── <REPO_ID>/
    └── repo/                             # 本地 Git 仓库（双 remote）
        ├── remote: gitcode → GitCode 源
        └── remote: github  → GitHub 目标
```

### 5.3 状态枚举

| status 值 | 含义 | 后续动作 |
|-----------|------|---------|
| `initialized` | 首次创建，未同步 | 下次运行执行全量同步 |
| `up_to_date` | 已是最新，无新 commit | 下次运行跳过 |
| `synced` | 同步成功 | 下次运行执行增量同步 |
| `conflict` | 冲突暂停 | 人工解决后重新运行，从冲突 commit 续传 |

---

## 6. 关键设计决策

| 编号 | 决策 | 备选方案 | 选择理由 |
|------|------|---------|---------|
| D1 | cherry-pick 逐个同步 | git merge / git rebase | 逐个处理可精确控制冲突粒度，支持断点续传；merge 引入合并提交污染目标历史；rebase 无法中断后续传 |
| D2 | 隐藏原始提交人 | 保留原始提交人 | 多平台协作场景下，GitCode 用户在 GitHub 无账号，保留原名无意义且泄露内部信息 |
| D3 | 状态持久化到 Git 仓库 | 本地文件 / 数据库 / 对象存储 | Git 仓库提供版本化、跨机器同步、零运维成本，且与项目基础设施一致 |
| D4 | 孤儿分支作为首次同步目标 | 从源分支 clone | 避免源分支根 commit 与目标分支根 commit 的 SHA 冲突；孤儿分支保证目标分支的独立历史 |
| D5 | `--ancestry-path` 限制 commit 范围 | 全范围 `rev-list` | 只取源分支直系 commit，避免同步 merge 引入的旁支 commit（旁支可能不属于目标仓库） |
| D6 | Python3 处理 JSON | jq / awk / 纯 shell | Python3 标准库 json 模块可靠性远超 shell 下 JSON 处理；jq 需额外安装；awk 处理 JSON 易出错 |

---

## 7. 安全设计

| 安全项 | 措施 |
|--------|------|
| 通信加密 | 全程 SSH 协议（`git@` 地址），无 HTTPS token 明文 |
| 身份隔离 | 原始提交人信息替换为统一账号（`--author-name` / `--author-email`），同时替换 author 和 committer |
| 状态安全 | 状态仓库为私有仓库，仅同步服务账号有写权限 |
| 锁安全 | 锁文件记录 PID，启动时校验进程存活，残留锁自动清理 |
| 无凭证硬编码 | SSH 密钥依赖系统 `~/.ssh/` 配置，脚本内无硬编码凭证 |

---

## 8. 运维设计

### 8.1 定时任务

```cron
# 每小时执行批量同步
0 * * * * <project_path>/sync-all.sh >> <project_path>/sync-all.log 2>&1
```

### 8.2 Dry-Run 模式

```bash
./sync-all.sh --dry-run   # 仅打印待同步仓库，不执行
```

### 8.3 日志与监控

| 日志文件 | 内容 | 轮转策略 |
|---------|------|---------|
| `sync-all.log` | 批量同步汇总 | 追加写入，依赖外部 logrotate |
| `states/<ID>/sync.log` | 单仓库详细日志 | 超 5000 行截断 + 超 10MB 滚动压缩 |
| `states/<ID>/mapping.log` | commit 映射记录 | 超 10MB 滚动压缩 |

### 8.4 故障恢复

| 故障场景 | 恢复方式 |
|---------|---------|
| 同步脚本异常退出 | 重新运行脚本，从 `SRC_HEAD` 断点续传 |
| 冲突导致暂停 | 手动解决冲突 → `git add -A && git commit --allow-empty` → 重新运行 |
| 目标分支被删除 | 脚本自动检测状态过期，重置为全量同步 |
| 源分支 force-push | 脚本检测 `merge-base --is-ancestor` 失败，从头全量同步 |
| 状态仓库损坏 | Git 仓库可回溯历史版本，`git revert` 恢复 |
| 新机器部署 | clone 状态仓库，配置 SSH 密钥，设置 cron 即可 |

---

## 附录 A：核心脚本函数清单

| 函数 | 所属脚本 | 职责 |
|------|---------|------|
| `init_state_repo()` | gitcode-to-github-sync.sh | 初始化状态仓库本地 clone，确定 STATE_DIR |
| `push_state_to_repo()` | gitcode-to-github-sync.sh | 滚动压缩 + 截断 + git commit/push 状态 |
| `rotate_and_compress()` | gitcode-to-github-sync.sh | 文件超阈值则 gzip 归档 |
| `log_info/warn/error()` | gitcode-to-github-sync.sh | 分级日志，同时写 stdout 和文件 |
| `init_state_json()` | gitcode-to-github-sync.sh | 初始化 state.json |
| `get_state()` / `set_state()` | gitcode-to-github-sync.sh | JSON 状态读写（Python3 封装） |
| `get_last_synced_from_mapping()` | gitcode-to-github-sync.sh | 从 mapping.log 读取 SRC_HEAD |
| `append_mapping()` | gitcode-to-github-sync.sh | 追加 commit 映射（去重） |
| `update_src_head()` | gitcode-to-github-sync.sh | 原子更新 SRC_HEAD |
| `acquire_lock()` / `release_lock()` | gitcode-to-github-sync.sh | 文件锁 + PID 校验 |
| `sync_repo()` | gitcode-to-github-sync.sh | 主同步逻辑（8 步） |
| `log()` | sync-all.sh | 批量日志 |

## 附录 B：配置文件示例

```conf
# GitCode→GitHub 增量同步配置
# 格式: gitcode_repo | gitcode_branch | github_repo | github_branch | [author_name] | [author_email] | [state_repo]

# ── 默认值 ──
# author_name:    <default_author_name>
# author_email:   <default_author_email>
# state_repo:     <state_repo_url>

# ── 同步任务 ──
<gitcode_repo_url> | <gitcode_branch> | <github_repo_url> | <github_branch>
```
