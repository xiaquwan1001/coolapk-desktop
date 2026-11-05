# coolapk-desktop 工作区（`.pi/`）

本目录是**本地工作区**，只提交到本地分支 `local/workspace`，**从不进上游 PR**。
上游 `coolapk-desktop/coolapk-desktop` 的提交树里没有任何 `.pi/` 文件
（`git ls-tree -r --name-only origin/main | grep '^.pi/'` 为空），可以随时核对。

这份文件是工作区与远端拓扑的唯一事实来源；`AGENTS.md` 管工作规则与版本发布，
`.pi/memory/MEMORY.md` 管事实与教训，本文件管"东西该放哪、代码该推到哪"，
三者冲突时以本文件 + 记忆顶部的「当前事实基线」为准。

## 工作点与资料存放

- 本仓库是唯一工作点：会话、资料、项目记忆都收在仓库内，不散落到 home。
- `.pi/settings.json` 设 `sessionDir: ".pi/sessions"`，在本目录启动的会话直接写进仓库。
- 新增资料优先放 `.pi/artifacts/` 并在该目录 `README.md` 登记；**不要**把 deb/zip 之类二进制放进
  `~/下载/`，也不要把它们提交入库。
- 不入库的那几个目录由 `.git/info/exclude` **本地忽略**；不要把它们写进 `.gitignore`——`.gitignore` 是上游文件，
  本地规则写进去就会随 PR 漂到上游。
- 这些约定只写在本文件与 `.pi/` 里。上游跟踪的 `AGENTS.md` 只放上游通用规则
  （更新日志、版本发布、提交规范），不再混入本地工作区说明——否则本地专属路径会随 PR 漂到上游。

## 目录职责

| 路径 | 内容 | 是否入库 |
| --- | --- | --- |
| `.pi/memory/` | 项目记忆（机制、契约、坑、issue/PR 结论） | 入库（`MEMORY.md`） |
| `.pi/skills/` | 项目专属技能，只放本项目才成立的知识 | 入库 |
| `.pi/settings.json` | 会话目录等 Pi 配置 | 入库 |
| `.pi/sessions/` | 会话原文与 fold evidence | 不入库 |
| `.pi/artifacts/reference/` | 官方安装包、上游源码快照等对照资料（清单见该目录 `README.md`） | 不入库 |
| `.pi/fabric/`、`.pi/trash/` | 运行时状态、可清理残留 | 不入库 |

"入库"= 提交到本地分支 `local/workspace`；**不是**提交到上游。

## 远端拓扑

| remote / 仓库 | 用途 |
| --- | --- |
| `origin` = `https://github.com/coolapk-desktop/coolapk-desktop.git` | 上游，唯一事实来源；**无写权限，只 fetch** |
| `fork` = `git@github.com:xiaquwan1001/coolapk-desktop.git` | 个人 fork，`parent` 就是当前上游；推送分支与开 PR 都走它 |

历史仓库（都不要再连）：

- `xiaquwan1001/coolapk-desktop-community`：曾经的 PR 出口，已改名为现在的 `fork`，
  本地 `community` remote 已删除。**记忆里"只有 community remote 能开 PR"的旧结论作废**，
  它对应的是 fork 改名前的拓扑。
- `daimiaopeng/coolapk-desktop`：旧上游，已归档（archived），`origin` 已指向新组织。
- `xiaquwan1001/coolapk-desktop-legacy-archived`：曾 parent 指向旧上游的错 fork，已归档只读，
  待网页 Settings → Danger zone 删除（`DELETE /repos` 需要 `delete_repo` scope）。

## 代码改动走 PR，工作区不走 PR

```bash
# 1) 从上游当前 main 起特性分支（不要基于 local/workspace）
git fetch origin && git checkout -b <branch> origin/main

# 2) 只提交代码/文档的具体路径，避免把 .pi/ 带进 PR
git add <具体文件...>

# 3) 推个人 fork，向上游开 PR（正文写文件，避免反引号被 shell 替换）
git push fork <branch>
gh pr create --repo coolapk-desktop/coolapk-desktop --base main \
  --head xiaquwan1001:<branch> --title "<标题>" --body-file /tmp/pr-<topic>.md
```

判断标准：改了 `src/`、`src-tauri/`、`docs/`、`README.md` 等上游存在的文件 → PR；
只动了 `.pi/` → 直接提交 `local/workspace` 并 `git push fork local/workspace`。

## 分支约定

| 分支 | 内容 | 说明 |
| --- | --- | --- |
| `local/workspace` | 上游 main + `.pi/` 工作区提交 | 主工作区，日常停在这里 |
| `main` | 跟随 `origin/main` | 只快进，不开发 |
| 特性分支 | 从 `origin/main` 起 | 一个特性一个分支、一个 PR |

切分支时 `.pi/` 会变成未跟踪状态（`git status` 显示 `?? .pi/`），**文件不会丢失**，
切回 `local/workspace` 即恢复跟踪。所以在特性分支上提交要用具体路径，
不要用 `git add -A` / `git commit -a`。

## 术语

- **上游**：`coolapk-desktop/coolapk-desktop`，即 `origin`。
- **本地工作区**：本仓库的 `local/workspace` 分支 + 磁盘上的 `.pi/` 目录。
- **入库**：提交到 `local/workspace`（不是上游）。
- **PR 分支**：从 `origin/main` 起、推到 `fork`、只含上游文件的特性分支。

## 相关文档

- `AGENTS.md`：更新日志、版本发布、版本六件套、提交规范。
- `docs/`：接口契约（`coolapk-api.md`）、Cookie 登录、beta 发布、性能诊断。
- `.pi/memory/MEMORY.md`：事实、契约与踩坑记录（顶部有当前事实基线）。
- `.pi/skills/coolapk-desktop-dev/SKILL.md`：开发/排查速查。
