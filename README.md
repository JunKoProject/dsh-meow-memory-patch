# dsh-meow-memory-patch

对 [`meow-memory`](https://github.com/Phant0Meow/dsh-meow-memory)（DeepSeek Harness 的记忆插件）的本地补丁集，以 pnpm patch 的形式提供。

**这是补丁，不是 fork。** 仓库里只有 `.patch` 文件，不含上游任何源码。

Patch set for [meow-memory](https://github.com/Phant0Meow/dsh-meow-memory), applied via pnpm's native patch mechanism.

---

## 这组补丁改了什么

| # | 改动 | 为什么 |
|---|---|---|
| 1 | `hitTopK` 默认值 2 → 4（四处：`CONFIG_DEFAULTS`、`DEFAULT_OPTS`、zod schema 的 `.default()`、`resolveConfig` 的 `??` 兜底） | 每条用户消息按关键词召回的条目上限，2 条太少——一轮对话里往往有三四个主题同时被唤起，2 条会漏掉重要的 |
| 2 | `hitQuery` 的召回池加入 `db.list("project")` | 原本 project 层**完全不在**每消息唤回的池子里，只能靠首轮全量注入。关掉全量之后，项目记忆一条也进不来 |
| 3 | `currentProject` 的过滤条件：`!== null && ...` 改为 `=== null \|\| ...` | 没设当前项目时，原本只召回「全局」条目，于是**项目记忆永远进不来**——这是「看起来必须开全量注入」的真正原因 |
| 4 | `hitQuery` 返回值带出 `source_session`；`buildHitInjection` 里 project 层最多注入 1 条、topic 若来自别的会话行尾标注来源 | 对应上游 issue #125：topic 是 project 作用域又常带「进行中」状态，别的会话写的会被当成当前任务接着做；project 条目平均 349 字、最长 716，命中三四条会挤爆本轮上下文 |

四条合起来的目的是同一件事：**让项目记忆在设了项目、没设项目时都能被正常召回，且召回的结果不会被误读成别的会话正在进行的事。**

---

## 安装

在你的 DSH profile 目录（例如 `E:\DSH ME\profiles\desktop`）里：

1. 把 `patches/meow-memory@0.33.3.patch` 复制到 profile 的 `patches/` 目录下。

2. 在 profile 的 `pnpm-workspace.yaml` 末尾加上（pnpm 12 把 patch 声明放在这里；pnpm 11 及更早放在 `package.json` 的 `pnpm.patchedDependencies`）：

```yaml
patchedDependencies:
  meow-memory@0.33.3: patches/meow-memory@0.33.3.patch
```

3. 安装：

```sh
pnpm install
```

4. 验证补丁确实生效：

```sh
grep -c 'core-memory-patch' node_modules/meow-memory/lib/index.js   # 应该是 4
grep -c 'hitTopK: 4' node_modules/meow-memory/lib/index.js          # 应该是 2
```

打不上时 pnpm 会直接报错（提示 patch 与包内容不匹配），不会静默跳过。

---

## 版本对应

| 补丁 | 对应 meow-memory 版本 | 状态 |
|---|---|---|
| `meow-memory@0.33.3.patch` | 0.33.3 | 已验证：`pnpm install` 后自动应用，运行时 `hitTopK=4` |

上游出新版本时，这个 patch 大概率会冲突（行号变了）。冲突是好事，说明该重新看一遍这些改动上游有没有收。

---

## 为什么不用 fork

fork 要跟着上游维护整份源码；patch 只描述差异，上游一改就能看出来哪里冲突了。对四个小改动来说，patch 的维护成本低得多。

## 为什么不用 `pnpm patch` 之外的办法

- 直接改 `node_modules`：pnpm 把包装成 store 的硬链接，你改的那份和「声明应然」的状态不一致，下次 `pnpm install` / `add` / `update` 会按 store 原件覆盖回去——补丁静默消失。
- 安装后钩子自动重打：依赖执行时机，脆弱，而且 `pnpm install` 一跑就失效。

`patchedDependencies` 是 pnpm 自己认的状态：写进配置、跟着仓库走、每次安装自动应用、冲突时大声报错。

---

## 许可

上游 [meow-memory](https://github.com/Phant0Meow/dsh-meow-memory) 采用 **MIT License**，Copyright (c) 2026 Phant0Meow。MIT 允许修改、再分发与商业使用，条件是保留版权声明与许可声明——完整的上游许可证见本仓库的 [`LICENSE`](LICENSE)。

本仓库中的补丁（`.patch` 文件）同样以 **MIT** 发布，免费、可商用、可再分发。

## 致谢

上游作者 [Phant0Meow](https://github.com/Phant0Meow)：meow-memory 的七层记忆结构与「核心常驻 + 关键词唤起」的注入设计，是这组补丁存在的全部前提。

如果这几处改动对你有用，也建议去上游提个 issue 或 PR——补丁最终应该活在原仓库里，而不是这里。
