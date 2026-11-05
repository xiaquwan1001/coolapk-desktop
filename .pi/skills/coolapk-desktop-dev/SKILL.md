---
name: coolapk-desktop-dev
description: 在 coolapk-desktop 仓库内开发/排查时的项目速查：构建与测试命令、版本发布六件套、docs 地图、本地 Agent 接口(agent-bridge/MCP)契约、Cookie 登录与接口反爬边界、.pi 工作区（记忆/会话/资料/技能）约定。触发词：酷安桌面版、coolapk-desktop、agent-bridge、MCP 发帖/点赞、cookie 登录、beta-release、Tauri 构建、版本发布、read_feed 工具。
---

# coolapk-desktop 项目速查

## 仓库定位

Tauri（Rust 后端 + Vue 前端）复刻官方酷安桌面客户端的独立实现。本仓库即唯一工作点，
会话/记忆/资料/技能都在 `.pi/` 下（目录地图与入库规则见 `.pi/README.md`；上游 `AGENTS.md` 只放上游通用规则）。
当前工作分支与上游差异状态先查 `.pi/memory/MEMORY.md` 顶部的「当前事实基线」，动手前 `git fetch` 看 main 是否已有同类改动。

## 远端与 PR 路径

仓库/远端/入库规则的唯一事实来源是 `.pi/README.md`，速记：

- `origin` = coolapk-desktop/coolapk-desktop（上游，无写权限，只 fetch）；`fork` = xiaquwan1001/coolapk-desktop（推送与开 PR 都走它，parent 就是当前上游）。
- 开 PR：`git fetch origin` → `git checkout -b <branch> origin/main` → `git add <具体文件>` → `git push fork <branch>` → `gh pr create --repo coolapk-desktop/coolapk-desktop --base main --head xiaquwan1001:<branch> --body-file <文件>`（正文走文件，反引号会被 shell 替换）。
- **`.pi/` 从不进上游 PR**：只动 `.pi/` 的改动直接提交 `local/workspace` 并 `git push fork local/workspace`。
- 历史仓库已作废：旧 `community` remote 删除（仓库改名成现在的 fork）、旧上游 daimiaopeng 归档、错 fork 归档为 `xiaquwan1001/coolapk-desktop-legacy-archived`。记忆里"必须走 community 仓库"的结论已过时，别再照做。
- 主工作区停在 `local/workspace`（= main + `.pi/` 提交）。切到特性分支后 `.pi/` 只变成未跟踪（`?? .pi/`），**文件不会丢**；在特性分支提交一律用具体路径，别 `git add -A`。

## 命令

```bash
npm run typecheck                                   # vue-tsc --noEmit
npm test                                            # vitest run（src-tauri 侧用 cargo test）
npm run build                                       # prebuild 会跑 set-version.mjs + clean-dist.mjs
cargo check --manifest-path ./src-tauri/Cargo.toml
npm run version:set -- <X.Y.Z>                      # 同步六个版本文件，勿手改
git diff --check
```

发布纪律（完整流程见 `AGENTS.md`）：六个版本文件一致 → 独立提交 `发布版本 vX.Y.Z` →
带注释 tag `vX.Y.Z` → 推送分支与 tag → `git ls-remote` 核验。检查失败必须停。

## docs 地图（改代码前读对应文档）

| 文档 | 用途 |
| --- | --- |
| `docs/coolapk-api.md`、`docs/COOLAPK_API_DOCUMENTATION.md` | 接口、签名、请求头契约 |
| `docs/cookie-guide.md` | 手动抓 Cookie 导入的登录兜底 |
| `docs/agent-bridge.md` | 本地 Agent 接口：地址、token、scope、MCP 工具、写闸门 |
| `docs/beta-release.md` | beta-release.yml 触发与 Android versionCode 规则 |
| `docs/performance-diagnostics.md`、`docs/mobile-pull-refresh-audit.md` | 性能诊断、移动端下拉刷新审计 |

## Agent Bridge / MCP 关键事实

- 默认关闭，仅本机：Unix socket（应用数据目录下 `agent-bridge.sock`）或回环 TCP `127.0.0.1:47653`；
  服务端校验 `Host`，拒绝任何带 `Origin` 的浏览器跨域请求。
- token 只存应用数据目录 `agent-bridge-token`（0600），仅在本地开关页显示，**不进仓库/截图/对话**。
- 读工具与写工具分 scope；写操作走 confirm/auto 闸门 + 按 scope 分桶限流 + 按工具/账号/requestId 幂等去重。
- 写能力现状以 `docs/agent-bridge.md` 顶部「写能力现状」为准（已接通：发帖+图片、编辑动态、评论/楼中楼、评论点赞、转发、收藏与收藏单管理、删除动态/评论、关注话题、主页置顶、点赞、关注、私信；读侧另有通知/收藏/关注流/楼中楼/私信列表/商品只读）。共 46 个 MCP 工具，全部走同一套 scope/闸门/限流/审计；评论类工具要传 `coolapk_feed_replies` 里的 `id`（`rid` 是楼中楼目标，顶层为 0）。应用收藏已接通 `coolapk_favorite_apk` / `coolapk_unfavorite_apk`（参数 `packageName`，内部走收藏单接口：收藏加默认收藏单、取消移出全部已选）；旧 `/v6/apk/favorite` 已废弃，别照 docs/coolapk-api.md 的 `id` 试。商品类只做只读，不做下单/购买/转卖等交易操作。
- 发帖走官方顺序：`client.upload_image(bytes,name,mime,"feed",None)` 取 `data.url` → 逗号拼 `pic` → `client.create_feed_with_options`；MCP 大图用 `imagePaths`（本机绝对路径，应用自己读盘），比 base64 省事，且 MCP body 上限已放宽到 32 MiB。
- 单机联调用例：`src-tauri/tests/agent_bridge`（审计写 `agent-bridge-audit.jsonl`）。

## 已知边界（别重复踩）

- 官方授权页无法向内置窗口回传登录态；直连 `account.coolapk.com` 会被拦
  （无 UA → 567 拦截页、正常 UA → 403 空体、部分 App-* 头 → 404 `{"status":1000,"message":"请求方式错误"}`）。
  可行路径只有 `docs/cookie-guide.md` 的手动 Cookie 导入，不要伪造请求头试探。
- 反编译产物统一放仓库已忽略的 `coolapk_all_decompiled/`；官方 deb 与上游源码快照放
  `.pi/artifacts/reference/`（清单见该目录 `README.md`），不要下载到 `~/下载/`。

## 工作区约定

- 目录地图与"哪些入库/哪些不入库"见 `.pi/README.md`；关键点：`.pi/` 只提交到 `local/workspace`，**不当上游 PR 内容**。
- 记忆：改 `.pi/memory/MEMORY.md`（提交到 `local/workspace`，`§` 分隔，行尾 `<!-- created=…, last=… -->`）；全局
  `~/.pi/agent/projects-memory/coolapk-desktop-dev` 是软链，`mem -t project` 可直接命中。
- 会话：`sessionDir: ".pi/sessions"`（不入库）；检索用 `sess`/`sessidx`。
- 技能：项目专属技能放 `.pi/skills/<name>/SKILL.md`，只写本项目才成立的知识，通用技能仍放全局。
- 运行时：`.pi/fabric/`、`.pi/trash/` 不入库；测试临时目录写在 `~/.cache/pi-tmp/`，可随时清。
