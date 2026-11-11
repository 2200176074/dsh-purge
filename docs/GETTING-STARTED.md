# 上手文档

从零到能用的完整流程。假设你已经在用 DSH（DeepSeek Harness）。

---

## 你需要什么

| 项 | 要求 |
|---|---|
| DSH 版本 | 0.2.1-rc.2 及以上（0.2.2-rc.1 已验证 71/86） |
| Node.js | ≥ 18 |
| 包管理器 | pnpm |
| 磁盘 | 约 50 MB（插件 4 MB + 宿主备份 20 MB 起） |

---

## 一、装插件

### 1. 确认你的 profile 目录

DSH 把插件装在 profile 里，默认是 `~/.dsh/profiles/web`：

```bash
ls ~/.dsh/profiles/
```

看到 `web` 就对了。用桌面版的话可能是 `desktop`。

### 2. 安装

从本地目录装：

```bash
cd ~/.dsh/profiles/web
pnpm add /path/to/dsh-purge --config.dangerouslyAllowAllBuilds=true
```

或者从本仓库装：

```bash
cd ~/.dsh/profiles/web
pnpm add https://codeload.github.com/2200176074/dsh-purge/tar.gz/refs/heads/main \
  --config.dangerouslyAllowAllBuilds=true
```

### 3. 为什么那个 flag 不能省

pnpm 默认**拦截依赖的 postinstall 脚本**。而这个包的 postinstall
（`scripts/ensure-profile-bundle.mjs`）要做两件事：

1. 把 `dsh-purge` 写进 `package.json` 的 `dsh.profile.bundles`
2. 往 profile 根的 `cordis.patch.yml` 写 insert 块

跳过它的话，包文件在 `node_modules` 里躺着，但 DSH 根本不会加载它 ——
**看起来装好了，实际没生效**。这是最容易踩的坑。

### 4. 检查装对没有

```bash
cd ~/.dsh/profiles/web
grep dsh-purge package.json          # bundles 里要有 "dsh-purge"
grep -A3 "id: dsh-purge" cordis.patch.yml   # insert 块要存在
ls node_modules/dsh-purge/package.json      # 包要在
```

三样都齐了才对。

装好时 pnpm 应该输出过这两行：

```
dsh-purge: appended to bundles (/root/.dsh/profiles/web)
dsh-purge: wrote body insert (/root/.dsh/profiles/web/cordis.patch.yml)
```

如果没看到，说明 postinstall 被拦了，重装并确认带上了 flag。

---

## 二、找宿主目录

补丁要改的是 DSH 本体（`@deepseek-ai` 那一堆包），不是插件自己。
先确认宿主在哪：

```bash
ls /opt/dsh/node_modules/@deepseek-ai/dsh-agent-instructions/lib/index.js
```

能列出来就是这里。其他常见位置：

```bash
# npm 全局
npm root -g

# profile 内嵌
ls ~/.dsh/profiles/web/node_modules/@deepseek-ai

# 桌面版（要先解 app.asar）
```

判定标准只有一个：**那个目录下有 `dsh-agent-instructions/lib/index.js`**。

---

## 三、备份

打补丁会改 57 个宿主文件。**先备份**：

```bash
BK=~/dsh-bak-$(date +%Y%m%d-%H%M%S)
mkdir -p "$BK"
tar czf "$BK/host-full.tar.gz" -C /opt/dsh/node_modules @deepseek-ai
ls -lh "$BK/host-full.tar.gz"
```

约 20 MB。别省这一步。

---

## 四、打补丁

```bash
cd ~/.dsh/profiles/web/node_modules/dsh-purge
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --apply
```

`DSH_BASE` 指向 `@deepseek-ai` 那一层。

### 正常输出

```
✓ 已清洗 patch #1  WORKSPACE_CONTEXT_INTRO
✓ 已清洗 patch #4  SYSTEM_PROMPT_PERSONA
...
全部完成 / All done。重启 dsh 生效 / Restart dsh to take effect.
```

### 看到这些别慌

| 输出 | 意思 | 要处理吗 |
|---|---|---|
| `已是最新` | 之前打过了 | 不用 |
| `na` | 桌面版专用补丁，Web 宿主跳过 | 不用 |
| `skipped` | 撤销型补丁，没东西可撤 | 不用 |
| `文件缺失` | 该布局没这个文件（如梁神预设） | 不用 |
| **`drifted`** | **官方原文改了，pattern 对不上** | **要看** |

### drifted 怎么办

这是唯一需要动手的情况。`--status` 会指出是哪条：

```
! [#74] ASSEMBLE_BEFORE_PRESTEP  代码/Code — drifted
```

1. 打开官方对应文件，找到那段代码现在的样子
2. 打开 `lib/core.js`，搜 `id: 74`
3. 把那条 patch 的 `pattern` 字符串换成官方现有文本
4. 重新 `--apply`

改 pattern 时注意：`pattern` 必须和官方源码**逐字一致**，包括制表符和换行。

---

## 五、生效

**必须重启 dsh。** 补丁改的是磁盘上的源码，运行中的进程用的还是旧内存镜像。

有补丁要打两轮才收敛，重启后再跑一次 `--apply`：

```bash
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --apply
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --status
```

第二次 `--apply` 应该全是 `已是最新`。

---

## 六、确认状态

```bash
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --status | head -5
```

期望在 0.2.2-rc.1 上看到：

```
补丁 71/86 已清洗, 0 待清洗
宿主版本 / host  0.2.2-rc.1
```

`0 待清洗` 是关键。有 pending 就说明还有补丁没打上。

---

## 日常使用

### 查看状态

```bash
cd ~/.dsh/profiles/web/node_modules/dsh-purge
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --status
```

### 改提示词

```bash
node bin/dsh-purge.js --edit
```

会打开 `~/.dsh/prompt-inject.md`。改完重启 dsh。

提示词正文不进仓库（`.gitignore` 里排除了），每台机器自己维护。

### 回滚

```bash
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --revert
```

或者整目录还原：

```bash
tar xzf ~/dsh-bak-*/host-full.tar.gz -C /opt/dsh/node_modules
```

### 完全卸载

```bash
node bin/dsh-purge.js --uninstall
```

会先 revert 再移除插件。

---

## 演练台

插件带一个红队演练台，包含：

- **64 个工具** — 资产表、凭证库、攻击链、POC 库、证据账本、报告导出
- **23 个技能** — 侦察、扫描、隧道、横向、Web 指纹、Webshell 等
- **3 个预设** — 编排好的智能体角色

入口在 Web 界面侧边的 dock 面板。核心概念：

**靶标（target）按会话隔离** —— 每个根会话有自己的靶标，子智能体继承父会话的，
多会话并行不会串写。

**并发上限 3** —— 同一靶标最多 3 个执行智能体，超了直接拒绝，不靠提示词自觉。

**证据要带来源** —— 每次发现都记 `provenance`（passive / active）和 `tool`，
报告里能追溯是哪一步得出的。

数据存在 `$DSH_HOME/redteam/`（SQLite）。

---

## 常见问题

### 装完没反应

先查三样：

```bash
cd ~/.dsh/profiles/web
grep dsh-purge package.json
grep -A3 "id: dsh-purge" cordis.patch.yml
ls node_modules/dsh-purge/
```

缺哪样补哪样。多半是 postinstall 没跑。

### 报「插件根未找到」

`DSH_BASE` 没设或指错了。指到**含 `dsh-agent-instructions/lib` 的那层**：

```bash
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --status
```

### 打了补丁但没效果

1. 重启 dsh 了吗
2. `--status` 显示多少 applied
3. `~/.dsh/prompt-inject.md` 存在吗、内容对吗

### dsh 升级后补丁全没了

正常。升级会覆盖 `@deepseek-ai` 目录。重跑 `--apply`，并准备好处理新的 `drifted`。

### 和上游 dsh-purge 冲突吗

**不要同时装。** 两者改同一批宿主文件，备份后缀也相同（`.dshpurge.bak`），会互相覆盖。

从上游切过来：

```bash
# 先用上游回滚
cd <上游目录> && node bin/dsh-purge.js --revert
# 再装这个
```

### 提示词会不会被上传

不会。提示词在 `~/.dsh/prompt-inject.md`，`lib/default-prompt-inject.md` 在 `.gitignore` 里，
都不进仓库。默认提示词以加密槽形式内嵌在 `lib/asset-table.js`，本地解密使用。

---

## 目录结构

```
dsh-purge/
├── bin/dsh-purge.js        CLI 入口
├── lib/
│   ├── core.js             补丁表与执行引擎（86 条补丁定义）
│   ├── index.js            插件主入口
│   ├── identity.js         提示词装配与身份段
│   ├── anchor.js           尾部锚点（每轮重新渲染）
│   ├── profile-insert.js   profile 配置写入
│   ├── asset-table.js      加密提示词槽
│   ├── redteam/            演练台（工具 / 存储 / UI）
│   └── ...
├── client.js               Web 前端 bundle（dock 面板）
├── cordis.patch.yml        bundle 补丁声明
├── presets/redteam/        演练台预设
├── skills/redteam/         23 个技能
├── scripts/                postinstall 等
└── docs/                   截图与说明
```

---

## 关键概念

**补丁（patch）** — 一条对宿主源码的字符串替换规则。有 `pattern`（找什么）
和 `replace`（换成什么）。86 条，按 id 编号。

**清洗（apply）** — 把全部补丁打到宿主上。

**标记（marker）** — 补丁打完后写入的特殊字符串，用来判断"这条打过了没"。
`[dsh-purge]` 是通用前缀。

**备份（bak）** — 每个被改文件旁边的 `.dshpurge.bak`，存官方原件。
`--revert` 靠它还原。

**锚点（anchor）** — 注入提示词之外，每轮在 system prompt 末尾补一小段，
把身份和格式要求放回模型最后读到的位置。

**漂移（drifted）** — 官方源码变了，补丁的 `pattern` 找不到目标了。
不是错误，是需要更新的信号。
