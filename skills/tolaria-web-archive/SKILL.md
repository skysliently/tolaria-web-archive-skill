---
name: tolaria-web-archive
description: 保存网页到 Tolaria 笔记库（原封不动、一模一样、带图、无损存档）- Save any web page into the Tolaria vault losslessly with images. Browser-first rendering (agent-browser headed mode), user-assisted CAPTCHA/login intervention, raw HTML + offline self-contained snapshot + all images localized into a per-document folder under vault attachments/, verbatim Markdown transcription, then a Tolaria MCP note with frontmatter metadata, web-archive + content tags and an AI summary on top. Derived from web-page-archive; prefer this one when the destination is the Tolaria vault.
when_to_use: Use when the user asks to 保存/存档/收藏网页或文章 到 Tolaria/笔记库, pastes a URL saying 存下来/归档/存到库里, wants 原封不动/无损/带图保存, or the target page needs 登录/验证码. Not for non-HTML targets (PDF/视频/音频).
---

# Save Web Page → Tolaria（浏览器优先，无损带图）

把网页**原封不动、带图**保存进 Tolaria，并在笔记开头写入 AI 摘要、打上元数据与标签。本 skill 是 `dsh-plugin-web-archive` 的 ZCode 派生（脚本同源于 `web-page-archive`），相对父 skill 三点不同：**浏览器渲染优先**而非直接 HTTP；**无损层是硬交付物**；**人机校验由用户介入**，agent 不代破解。

用户显式调用本 skill 即视为同意本次 vault 写入；除此之外不主动写 vault。

## 产物与忠实度层级

每个文档的附件统一存放在 `attachments/<base>/` 子目录（`<base>` 与笔记文件名同名，即 `<日期>-<slug>`），自成一个结构化目录；笔记图片引用形如 `attachments/<base>/imgNN.png`。

| 层 | 产物（均在 `attachments/<base>/` 内） | 要求 |
|---|---|---|
| 无损层（必须） | `raw.html`（原始 DOM）、`snapshot.html`（离线自包含快照）、`imgNN.<ext>`（本地图片）、`screenshot.png`（全页截图，阶段 4 复制入内） | DOM 一字不动；图片全部本地化 |
| 转写层 | `article.md`（逐字 Markdown，图片指向 `attachments/<base>/` 且每张图独立成行） | 不改写、不缩写；图片语法按 Markdown 块级生成，不照抄源 DOM 的内联排版 |
| 语义层 | 笔记开头的 Summary + tags + 关键信息 | 你（LLM）分析生成，只增不改 |

`<base>` 由脚本从文章标题派生，与 `meta.json` 的 `suggested_note` 同源——附件目录与笔记一一对应，按文档检索附件时无需解析长文件名。快照内图片引用为同目录相对路径，整个目录可整体移动。

原文永不改写；摘要与元数据只是附加层。部分可访问时如实标注，不假装完整。

## Inputs

- 目标 URL（必须）
- vault：先 `mcp__tolaria__list_vaults` 确认（本机当前唯一 vault 为 `/Users/yunke/tolaria`）
- 用户指定标签（可选，原词保留）

依赖：`python3` + `lxml`（`python3 -m pip install lxml`）；agent-browser（缺失则 `npm i -g agent-browser && agent-browser install`）。

## 阶段 1 — 浏览器渲染（主路径，禁止跳过）

首选 `agent-browser` CLI 的 **headed 模式**（窗口用户可见可操作）；agent-browser 不可用时改用 browser-use 插件（IAB，见第 5 步）。

1. `agent-browser --headed open <url> && agent-browser wait --load networkidle`
2. **人机校验检查**：`agent-browser get title` + `agent-browser snapshot`（必要时截图）。命中任一 → 停下请用户介入：
   - 验证挑战特征："Just a moment"、"验证码"、"人机验证"、滑块、复选框挑战
   - 登录墙、公众号"环境异常"提示页
   - 正文明显为空或只剩页面骨架
   - 站点内容需登录才可见
   **介入协议**：告知用户"请在已打开的浏览器窗口完成验证/登录"，**等待用户回复确认后**再 `wait --load networkidle` + 重新 snapshot 复核。**绝不自动破解或绕过验证**——正确姿势是让用户本人过验证。用户拒绝介入且页面无法直抓 → 如实报告并停止。
3. **触发懒加载**：`agent-browser eval 'window.scrollTo(0, document.body.scrollHeight)'`，重复执行直到页高不再增长，再回顶部并 `wait --load networkidle`。
4. **抓取**：
   ```bash
   agent-browser get url > /tmp/web2tolaria-url.txt        # 重定向后的最终 URL，之后一律用它
   agent-browser eval 'document.documentElement.outerHTML' > /tmp/web2tolaria.html
   agent-browser screenshot /tmp/web2tolaria-shot.png --full
   ```
   **已知坑：`agent-browser eval` 的输出是 JSON 编码字符串**（带引号与 `\"`/`\n` 转义），不是裸 HTML。脚本现已自动识别并解码这种输入；若用了旧版脚本或需手工检查，落盘后先解码再进阶段 2，否则 lxml 解析失败、脚本以 exit 3（NO_CONTENT）退出。**注意必须先读入变量再写回**——`open(p,'w')` 会先清空文件，直接 `open(p,'w').write(json.loads(open(p).read()))` 会把抓取销毁成空文件：
   ```bash
   python3 -c "import json; p='/tmp/web2tolaria.html'; d=json.loads(open(p, encoding='utf-8').read()); open(p, 'w', encoding='utf-8').write(d)"
   ```
5. **IAB 替代通道**（仅当 agent-browser 不可用）：按 `control-browser` skill 打开页面，完成同样的校验检查与懒加载滚动；DOM 用 `tab.playwright.evaluate()` 取 HTML 落盘，截图经 `emitImage` 后存盘。拿不到完整 DOM 时接受降级，元数据标 `access: degraded`。

## 阶段 2 — 脚本归档

```bash
python3 scripts/archive_page.py --from-file /tmp/web2tolaria.html "<vault>" "<最终URL>"
```

产出全部落在 `<vault>/attachments/<base>/` 子目录（`<base>` 取自 meta.json 的 `base` 字段，与笔记文件名同名）。幂等（同 URL 重跑按目录复用已有文件，不产生冗余；meta.json 已存在时可直接复用进入阶段 3）；若同名目录已被**不同 URL** 的归档占用，脚本自动追加来源 token 去重为 `<base>-<token>`。meta.json 含 `image_originals`（每张本地图片对应的原图 CDN 地址，与 `files.images` 顺序对齐）和 `needs_enrichment: true`（确定性存档完成、语义层待补齐的握手信号）。

- **图片按真实格式落盘（硬要求）**：脚本下载后按魔数嗅探真实格式，以 `imgNN.<真实扩展名>` 落盘——PNG→`.png`、JPEG→`.jpg`、GIF→`.gif`，WebP 经 `sips` 转码为真 PNG（Tolaria 对 webp 的支持未验证，PNG 是已验证可渲染的安全格式；sips 失败则保留 `.webp`）。旧版脚本按 URL 查询串猜扩展名，会产出「`.png` 扩展名 + JPEG/WebP 字节」的错标文件——**Tolaria 预览按扩展名加载，错标图在应用内全部不显示**（浏览器靠内容嗅探不受影响，属潜伏缺陷，归档时无任何报错）。存量错标文件：`file` 核对后用 `sips -s format png` 转码为真 PNG（文件名与笔记引用均不变），原件备份到 /tmp。
- 图片多的页面脚本可能跑 1-4 分钟，Bash 调用给足超时（≥240s）。
- meta 中 `failed_images` 非空 → 从浏览器会话取 cookie（`agent-browser eval 'document.cookie'`）重跑并加 `--cookie "<cookie>"`；仍失败 → 失败图片保留远程 URL 引用，笔记元数据注明，不阻塞。
- 两个浏览器通道都不可用才降级直抓：`python3 scripts/archive_page.py "<url>" "<vault>"`（退出码 2 = 必须回到浏览器路径）。
- 全部通道失败 → `mcp__web_reader__webReader` 只能得 Markdown，标 `access: degraded`，向用户说明无损层缺失。

## 阶段 3 — 语义提取（LLM）

从 article.md 生成：

- `> Summary`：1-3 行中文摘要（专有名词保留原文），讲清"这篇是什么、核心结论/要点是什么"。
- tags：3-6 个 kebab-case 内容标签 + 站点标签（如 `weixin`、`zhihu`）；**必含 `web-archive`**（与 web-page-archive 及 vault 视图约定一致）+ 用户指定标签原词。
- `## 关键信息`：人物 / 产品 / 数据 / 结论，机器可读 bullet。

## 阶段 4 — 写入 Tolaria（必须走 MCP）

1. `cp /tmp/web2tolaria-shot.png "<vault>/attachments/<base>/screenshot.png"`（`base` 取自 meta.json，即附件子目录名——截图与该文档的其他产物同目录）
2. `mcp__tolaria__create_note`，路径用 meta.json 的 `suggested_note`（`<年>/<日期>-<slug>.md`），frontmatter：

```yaml
---
type: Note
status: Active
captured: <文章发布日期，缺失用今天>
url: <最终 URL>
source: <站点名>（<域名>）
archive: web-archive
via: browser        # browser | direct-http | degraded
tags: [web-archive, <站点标签>, <内容标签…>, <用户指定标签>]
---
```

3. 正文顺序：`> Summary` → 元数据表（标题 / 来源 / 发布时间 / URL / 快照与截图文件 / 字数 / 图片数 / 抓取方式与日期 / 若经用户过验证须注明）→ `## 关键信息` → `## 原文`（article.md 全文逐字粘贴，图片引用已是 `attachments/<base>/<fname>` 相对路径，**勿改**）。article.md 中每个图片引用已由脚本保证**独立成行**（前后空行），图注等相邻文字拆为相邻段落——这是硬性结构，粘贴勿合并，手工转写同样遵守：Tolaria 的块级编辑器不渲染段落内内联图，「图片+图注同行」会导致该图不显示。
4. **笔记图片策略——本地闭环（硬要求）**：原图必须以文件形式存进 vault `attachments/<base>/`。Tolaria MCP 的 `create_note` 不收二进制附件，直接写盘 `attachments/` 是 vault AGENTS.md 明示的唯一例外（执行时向用户说明这一理由）。笔记正文图片**一律本地路径引用**（article.md 已是 `attachments/<base>/<fname>` 形式，粘贴即用）：
   - 这是 Tolaria 的原生形态——官方文档写明 Tolaria 粘贴远程图时会后台导入 `attachments/` 并把引用改写为本地路径；且应用内 HTML 预览剥离远程资源、只放行 vault 内路径，本地引用才能让笔记与快照在应用内闭环查看。
   - `image_originals`（原图 CDN 地址）**只作溯源与校验用途，禁止写进正文当图源**——否则破坏本地闭环。
   - 本地引用渲染异常时，先修路径形态再验证（试 vault 根相对 `attachments/x.png` 与笔记相对 `../attachments/x.png` 两种），**不允许退回 CDN 图源**。
   - 仅当图片本身下载失败（`failed_images`）才保留远程引用，这是诚实标注的闭环破损点，元数据表必须注明。
4. `mcp__tolaria__refresh_vault` + `mcp__tolaria__open_note`。

## 阶段 5 — 校验

- 附件子目录 `attachments/<base>/` 中 `raw.html`、`snapshot.html`、全部 `imgNN.*`、`article.md`、`meta.json` 存在，且 `screenshot.png` 已复制入内
- **图片格式**：`file` 核对每个图片文件，魔数与扩展名一致（png/jpg/gif 皆可），不允许「扩展名 `.png` + 其他格式字节」的错标文件（Tolaria 会静默不显示）
- **图片块结构**：笔记与 article.md 中每个图片引用独占一行（前后空行），无「图片与图注同行」的内联图
- 笔记正文与 article.md 抽查比对无缺失：文字段落逐字在位；段落/chunk 比对时，「图片块与相邻文字拆行」属允许的结构性差异，只要求文字内容逐字在位
- **本地闭环**：笔记正文图片引用全部指向 `attachments/<base>/` 内的本地文件，无远程 http 图源（`failed_images` 对应的远程占位引用是唯一例外，须在元数据表注明）。判定标准是**应用内可渲染**，不只是路径存在——引用闭合 ≠ 图片可见
- tags 含 `web-archive`；`captured` / `url` / `source` 齐全
- 经用户介入过验证 → 元数据表注明"用户完成人机验证后抓取"
- 向用户报告**双路径**：附件子目录 `attachments/<base>/` 的产物清单 + vault 笔记路径

## 边界

- 非 HTML 目标（PDF / 视频 / 音频）不在范围
- 不自动绕过任何人机验证；这是合规边界，不是能力缺口
- 本 skill 是 `dsh-plugin-web-archive`（`~/workspace/dsh-dev-research/dsh-plugin-web-archive/`）的 ZCode 派生：脚本为独立副本，保留父插件的解析/转写/快照框架与 `image_originals` 契约，并额外包含四项 Tolaria 适配修复（`--from-file` 自动解码 agent-browser 的 JSON 输出、图片按魔数真实格式落盘、图片 token 行归一化、按文档分子目录存放），父 skill 没有这些修复；产物直写 vault `attachments/`、浏览器优先、带用户介入协议
- 附件布局与父插件不同：本 skill 为 `attachments/<日期>-<slug>/`（与笔记同名子目录 + 短文件名），`web-page-archive`/父插件仍为扁平 `attachments/<date>-<site>-<token>-imgNN.<ext>`。同一 URL 两个 skill 归档互不复制。本 skill 旧版扁平产物可按需迁移（文件移入 `attachments/<笔记文件名>/` 子目录 + 改笔记引用与 meta 路径），不迁移也不影响已有笔记渲染
