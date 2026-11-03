# Cowart for Claude

基于 tldraw 的 AI 无限画布：图片、视频、网页截图、AI 生成的 HTML / Slides 摆在一张画布上，用箭头标出要改哪里，交给 AI 迭代。

本仓库 fork 自 [zhongerxin/Cowart](https://github.com/zhongerxin/Cowart)（MIT）。原版是 Codex 插件，画布以 MCP Apps 小部件嵌在对话里，每个项目一张；这个 fork 改成**全机一个画布服务**，让 **Claude Code 桌面版、Codex、ZCode** 的所有会话共用同一张画布、按页分工。上游代码尽量不动，改动集中在 `adapters/`。

## 相比原版

| | 原版 | 本 fork |
|---|---|---|
| 宿主 | Codex | Claude Code 桌面版、Codex、ZCode |
| 画布在哪看 | Codex 对话里的小部件 | 本地网页 `http://cowart.localhost`：Claude Code 的 Browser 面板、Codex 的内置浏览器，或任意浏览器 |
| AI 怎么操作画布 | MCP 工具 | `cowart` 命令（Claude Code、Codex，不占 MCP）；ZCode 走 MCP 薄桥 |
| 画布数据 | 每个项目一张（`<项目>/canvas`） | 全机一张（`~/.cowart/canvas`），所有项目、会话按页分工 |
| 生图 | 经对话交给 Codex 生成 | 面板里选好模型和参数，点发送由画布服务直接生成图片 / 视频，不经对话 |

新增：

- **按页分工**：每页同一时间归一个会话负责，画布上发出的请求自动送到负责那一页的会话；几个页面同时开着只保存差异，不互相覆盖、不误删别人的页。
- **常驻画布服务**：全机一个，Windows 登录时自启，不随某个会话或宿主关掉而消失；新版本自动顶替旧版，已开的画布不断。
- **AI 视频、网页参考、视频截帧**：底部工具栏多了「AI 视频」（占位框 + 生成面板）和「网页」（网页截图卡片，可「照这个做 HTML」）；右键视频可截取当前帧。
- **AI 图片面板**：模型、画幅、参数集中在一个面板里，标明花不花钱；生成的图保持原比例，不被拉伸。
- **标注绑定到卡片**：标注箭头钉在指着的卡片上，卡片移动跟着走、删卡片一起删；新增蓝色「注释」（常驻说明）和「清理标注」。
- **页的文件夹**：页菜单换成文件夹树 + 搜索，可按建页先后 / 名称 / 最近改过排序；页归哪个文件夹由 AI 管。
- **AI 整理画布**：放文字、建分组框、移动、改尺寸、删除。
- **右键菜单**：「拷贝索引」（页名、卡片 id、本机文件路径，粘给 AI 就能用）、「在资源管理器中显示」。
- **反馈收件箱**：在任何项目的会话里说「反馈：…」，记到本机，回本仓库统一处理。

顺带修掉的原版问题：Windows 上写文件失败时残留大量 `.tmp`；删图后 Ctrl+Z 只回来标注、图片回不来；最多 40 页；右键菜单开着时拖动卡片后菜单再也打不开。

## 安装

要求 Node ≥ 22.12。Claude Code 只支持桌面版（命令行版、IDE 插件不行）。AI 图片 / AI 视频的直接生成走可选的 beast 生成后端（本机有 `beast` 命令行时）；没有时这类请求交给会话处理。

```bash
git clone -c core.autocrlf=false -c core.eol=lf https://github.com/15195999826/cowart-for-claude.git
cd cowart-for-claude
npm ci --prefix adapters
```

- **Claude Code 桌面版（+ Codex）**：`npm --prefix adapters run install:claude`。装 cowart skill 和命令入口 `~/.cowart/bin/cowart.mjs`；Windows 上加画布服务的开机自启；电脑上有 Codex（`~/.codex`）的，顺带装 Codex 的 skill 和提醒 hook。
- **ZCode**：`npm --prefix adapters run install:zcode`。

装完新开一个会话（Codex 新开任务，第一次会请你信任提醒 hook）。卸载：`node ~/.cowart/bin/cowart.mjs uninstall`、`npm --prefix adapters run uninstall:zcode`。

## 使用

在会话里说「打开 Cowart 画布」或「打开 Cowart 画布 <页名>」：

- **Claude Code**：画布在右侧 Browser 面板打开；画布发来的请求（按标注修改、AI HTML 等）由后台监听唤醒会话，你确认后执行。
- **Codex**：给出链接，在内置浏览器（Ctrl+Shift+B）里看；有请求等着时，你下次发话会被提醒一句，说「看画布」处理。
- **ZCode**：给出网址，在任意浏览器打开。

说「接管 <页名>」把一页换给当前会话。AI 图片、AI HTML、AI Slides、按标注修改的画布操作同原版。

## 开发

先读 [FORK.md](FORK.md)：开发准则（上游原样 + 适配层）、目录、宿主差异、补丁点清单。适配层结构见 [adapters/claude/README.md](adapters/claude/README.md)、[adapters/zcode/README.md](adapters/zcode/README.md)。

```bash
npm --prefix adapters run check:contract   # 上游接口和补丁点还在不在
npm --prefix adapters run test:claude      # 端到端测试（Claude Code、Codex 命令会话）
npm --prefix adapters run test:zcode       # ZCode 桥
npm --prefix adapters run build:artifacts  # 改了 adapters/ 后重新生成 adapters/generated/
```

改了 `src/`（画布页面）要在根目录 `npm ci && npm run build:artifacts` 重新生成 `mcp/generated/`。在仓库目录里开 Claude 会话时，根目录 `.mcp.json` 里上游的 `cowart_mcp` 用不上，可在 `.claude/settings.local.json` 写 `{"disabledMcpjsonServers": ["cowart_mcp"]}` 关掉。

## 许可与致谢

MIT。原版作者 ZHONG XIN（[zhongerxin/Cowart](https://github.com/zhongerxin/Cowart)），画布基于 [tldraw](https://github.com/tldraw/tldraw)。上游的英文说明原样保留在 [README.en.md](README.en.md)。
