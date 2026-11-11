# dsh-purge（社区延续版）

DeepSeek Harness 破甲与红队演练插件。

版本 **1.1.93** — 基于上游 1.1.92 的完整代码，修补了对 DSH **0.2.2-rc.1** 的适配。

> ⚠️ 上游 `YuJunZhiXue/dsh-purge` 与 `HunYiSage/dsh-purge` 两个仓库都已从 GitHub 删除（404），
> npm 上只剩一个 331 字节的占位包。本仓库是那份代码的延续，**不是原团队发布**。

---

## 为什么有这个仓库

上游最后可用的版本是 1.1.92，它声明支持 DSH `0.2.1-rc.5` / `0.2.1-rc.2`。
装到 `0.2.2-rc.1` 上，65 条补丁只有 2 条能打进，其余全部 pending —— 因为 0.2.2 改了宿主源码，
补丁的 pattern 对不上。原仓库又没了，没人更新。

这份把两条卡住的补丁修好了，别的一行没动。

### 改了什么

**#12 APPROVAL_ASK_SENTENCE_BYPASS**

官方把审批文案改了：

```diff
- Approval policy: ask. Operations that require approval may ask through the configured answerers; ...
+ Approval prompts are enabled in this session: operations that require approval may ask through the configured answerers; ...
```

新增两条 pattern，覆盖双引号形态（`dsh-user-approval/lib/index.js`）
和单引号形态（`dsh-user-approval/lib/types/index.js`）。

**#74 ASSEMBLE_BEFORE_PRESTEP**

0.2.2 在 `inbox.claim()` 和 `model.current()` 之间插了一行：

```js
const recall = await dshGoRecall.prepare(this.loopCtx, this, position, claimed, signal);
```

原 pattern 要求这两句紧邻，永远失配。新增一条按新形态重打的 pattern，
保留 `dshGoRecall.prepare` 调用，把注入块插在它前面。

两条补丁的**原 pattern 全部保留**，装在 0.2.1 上照样能用。

**元数据**

- `version` 1.1.92 → 1.1.93
- `dshTarget` / `dshTargets` 补上 `0.2.2-rc.1`
- 删除 `dsh.install` —— 里面的 tarball / zip / git 三个地址全指向已删除的仓库
- 删除 `.github/workflows/release.yml` —— 依赖原仓库 secret，跑不起来

---

## 实测结果

DSH `0.2.2-rc.1`：

| 项 | 数 |
|---|---|
| 补丁总数 | 86（含桌面版专用 21 条，Web 宿主标 na） |
| **applied** | **71** |
| pending | 0 |
| 按设计跳过 | #71 梁神预设 / #80 撤销型补丁 |

对照：上游 1.1.92 在同一宿主上只有 2/65。

验证手段：57 个目标文件 `node --check` 全过、22 个核心插件 `import()` 全成功、
348 个 YAML 用带 `!js` 标签的 schema 解析全过。

---

## 安装

### 前置

- DSH 0.2.1-rc.2 及以上（0.2.2-rc.1 已验证）
- Node.js ≥ 18、pnpm

### 装

```bash
cd ~/.dsh/profiles/web
pnpm add /path/to/dsh-purge --config.dangerouslyAllowAllBuilds=true
```

**`--config.dangerouslyAllowAllBuilds=true` 不能省。** pnpm 默认拦 postinstall，
`ensure-profile-bundle.mjs` 跑不了的话 `dsh.profile.bundles` 和用户层
`cordis.patch.yml` 的 insert 都不会写 —— 包躺在 node_modules 里，插件等于没装。

装上后应该看到：

```
dsh-purge: appended to bundles (/root/.dsh/profiles/web)
dsh-purge: wrote body insert (/root/.dsh/profiles/web/cordis.patch.yml)
```

### 打补丁

```bash
cd ~/.dsh/profiles/web/node_modules/dsh-purge
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --apply
```

`DSH_BASE` 指向含 `dsh-agent-instructions/lib` 的那层目录。

### 生效

补丁写的是宿主源码，**必须重启 dsh**。有补丁要两轮才收敛，重启后再跑一次 `--apply`。

---

## 命令

```bash
dsh-purge --status      # 状态：补丁数、宿主版本、每条补丁结果
dsh-purge --apply       # 应用全部清洗
dsh-purge --revert      # 回滚
dsh-purge --uninstall   # 卸载
dsh-purge --edit        # 编辑 prompt-inject.md
dsh-purge --help        # 帮助
```

---

## 版本兼容

| 宿主 | 情况 |
|---|---|
| 0.2.2-rc.1 | 71/86，已验证 |
| 0.2.1-rc.5 / 0.2.1-rc.2 | 老 pattern 仍在，未实测 |
| 其他 | 大概率出现新 pattern 漂移，`--status` 标 `drifted` |

`drifted` 不是失败，是官方原文被改了。做法：看 `--status` 里哪条 drifted，
把 `lib/core.js` 对应 patch 的 `pattern` 换成官方现有文本。

**dsh 升级会覆盖 `@deepseek-ai` 目录，补丁全丢，得重跑 `--apply`。**

---

## 回滚

```bash
DSH_BASE=/opt/dsh/node_modules/@deepseek-ai node bin/dsh-purge.js --revert
```

或整目录还原：

```bash
tar xzf /path/to/backup.tar.gz -C /opt/dsh/node_modules
```

---

## 许可证与署名

Apache-2.0，详见 [LICENSE](./LICENSE) 与 [NOTICE](./NOTICE)。

上游要求二开保留署名并注明来源。原仓库已删，链接指向的是历史位置。
上游的 `LICENSE-COMMERCIAL.md` 写明商业使用需联系原团队（`t.me/LLM_PoJia`，是否仍有效未知）。
**本仓库不代原团队授权任何商业用途。**

---

## 免责

本工具会改写宿主程序的提示词与安全边界（沙箱围栏、审批拦截、文件观察策略等）。
使用后果自负，用在你没有权限的系统上与本项目无关。
