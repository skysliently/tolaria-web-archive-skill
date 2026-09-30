# tolaria-web-archive-skill

把任意网页**原封不动、带图、无损**归档进 [Tolaria](https://github.com/refactoringhq/tolaria) 笔记库的 ZCode Skill。

给它一个 URL,它会:用真实浏览器渲染页面(人机验证由用户介入,绝不代破解)→ 本地化全部图片与原始 DOM → 生成离线自包含快照与逐字 Markdown 转写 → 通过 Tolaria MCP 写入一篇带 AI 摘要、元数据表与标签的笔记。产物按文档分子目录结构化存放,笔记在 Tolaria 应用内可完整查看(图文闭环)。

## 归档产物结构

```
<vault>/
├── 2026/2026-09-30-<slug>.md           # 笔记:AI 摘要 → 元数据表 → 关键信息 → 原文全文
└── attachments/2026-09-30-<slug>/      # 与笔记同名的附件子目录
    ├── raw.html        # 原始 DOM(一字不动)
    ├── snapshot.html   # 离线自包含快照(去脚本、图片本地化,可整体搬移)
    ├── imgNN.<ext>     # 图片,按魔数真实格式落盘(png/jpg/gif)
    ├── screenshot.png  # 全页截图
    ├── article.md      # 逐字 Markdown 转写(图片引用独立成行)
    └── meta.json       # 元数据 + image_originals(CDN 原图溯源)
```

三层忠实度:**无损层**(DOM 一字不动、图片全部本地化)→ **转写层**(逐字 Markdown,不改写不缩写)→ **语义层**(AI 摘要与标签,只增不改)。

## 创建工具与模型

- **创建环境**:ZCode 智能体(智谱 ZCode 桌面端),2026-09-30 在真实归档任务中迭代开发、逐项验证
- **开发模型**:GLM-5.3-Flash(Z.ai GLM 系列,ZCode 会话模型)
- **开发使用的工具链**:agent-browser CLI(浏览器渲染与抓取)、python3 + lxml(归档脚本)、Tolaria MCP(笔记读写)、macOS `sips`(WebP→PNG 转码)
- **来源与致谢**:`dsh-plugin-web-archive` 的 ZCode 派生(与 `web-page-archive` 同源);本 skill 的脚本为独立副本,在其框架上做了四项 Tolaria 适配修复(见下文"踩坑记录")

## 使用前置条件

| 组件 | 用途 | 获取 / 安装 |
|---|---|---|
| **ZCode** | 运行 skill 的智能体(需支持 Skill 机制与 MCP) | ZCode CLI / 桌面端 |
| **Tolaria 应用 + vault** | 笔记库本体 | 安装 Tolaria 并创建 vault;在 ZCode 中配置 Tolaria 的 MCP server,确认 `mcp__tolaria__*` 系列工具可用 |
| **python3 + lxml** | 归档脚本 | `python3 -m pip install lxml` |
| **agent-browser** | 浏览器渲染(主路径) | `npm i -g agent-browser && agent-browser install`;注意 npm 全局 bin 目录(如 `~/.npm-global/bin`)需在 PATH 中 |
| **macOS** | WebP→PNG 转码依赖系统 `sips` | 推荐;非 macOS 时 WebP 图片保留 `.webp` 原格式,其余功能不受影响 |
| browser-use 插件(可选) | 应用内浏览器替代通道 | 仅当 agent-browser 不可用时兜底 |

安装自检:

```bash
python3 -c "import lxml; print('lxml ok')"
agent-browser --version          # 命令不存在 → 检查 PATH
# ZCode 会话中调用 mcp__tolaria__list_vaults,应返回你的 vault 列表
```

## 安装

```bash
git clone git@github.com:skysliently/tolaria-web-archive-skill.git
mkdir -p ~/.agents/skills
cp -R skills/tolaria-web-archive ~/.agents/skills/
```

`~/.agents/skills/` 是 ZCode 的用户级 skill 目录(本 skill 实际安装位置);项目级安装则放入 `<project>/.zcode/skills/`。重启会话后在可用 skill 列表中可见 `tolaria-web-archive`。

## 使用

ZCode 会话中任一方式触发:

- 显式调用:`/tolaria-web-archive <url> 存储这个文档`
- 自然语言:"把这个网页存到 Tolaria" / "归档这篇文章,要带图无损"

人机验证协议:页面出现验证码 / 登录墙时,skill 会保持**用户可见的浏览器窗口**并请你本人完成验证;你确认后再继续抓取。skill 不自动绕过任何验证。非 HTML 目标(PDF/视频/音频)不在范围内。

## 相对上游的四项修复(踩坑记录)

这些缺陷的特点都是"归档时无任何报错,打开笔记才发现",已在脚本与流程中内置修复:

1. **agent-browser eval 输出是 JSON 编码字符串**(带引号与转义),直接落盘会让 lxml 解析失败(NO_CONTENT 误报)→ 脚本自动识别并解码;手工解码必须"先读入变量再写回"(`open(p,'w')` 会先清空文件,直接 `open(p,'w').write(json.loads(open(p).read()))` 会把抓取销毁成空文件)。
2. **CDN 图片按扩展名错标**:微信/Anthropic 等 CDN 返回 JPEG/WebP 字节,旧脚本按 URL 查询串猜扩展名一律存成 `.png`——Tolaria 预览按扩展名加载,错标图在应用内**全部不显示**(浏览器靠内容嗅探不受影响,故潜伏)。现按魔数真实格式落盘,WebP 一律转码为真 PNG(经实测验证的渲染安全格式)。
3. **图片与图注同行不渲染**:源页面 DOM 常把 `<img>` 和图注放进同一段落流,Tolaria 的块级编辑器不解析段落内内联图 → 转写层强制每个图片引用独立成行(前后空行),图注为相邻段落,文字一字不动。
4. **附件按文档子目录存放**:全部产物写入 `attachments/<笔记名>/`,与文档命名强关联;快照内图片引用为同目录相对路径,整个目录可整体搬移。

核心理念:校验的判定标准是**应用内可渲染**,不是"路径存在"——引用闭合 ≠ 图片可见。

## 适用性说明

- 笔记路径与 frontmatter 遵循本库的编年法约定(`<年>/<日期>-<slug>.md`),换库使用时按你自己的 vault `AGENTS.md` 约定调整即可,skill 正文对写库规范有完整描述
- 首次接入建议先用一个简单页面跑通全流程,再处理需要登录/验证的站点

## 已验证环境

- macOS 26(darwin 25.5.0)arm64 · ZCode · GLM-5.3-Flash
- Tolaria(MCP 连接本地 vault)· agent-browser(Chrome 154.0.8037.92)
- 实测通过:微信公众号文章、anthropic.com 研究报告(均无需人机验证);含验证墙站点的介入协议见 SKILL.md
