# tolaria-web-archive-skill

**[中文](README.md)** | **[English](README.en.md)**

A ZCode Skill that archives any web page **verbatim, with images, losslessly** into a [Tolaria](https://github.com/refactoringhq/tolaria) vault.

Give it a URL and it will: render the page in a real browser (human verification is done by the user, never bypassed) → localize every image and the raw DOM → produce an offline self-contained snapshot and a verbatim Markdown transcription → write a note via the Tolaria MCP with an AI summary, a metadata table, and tags. Artifacts live in a per-document folder under the vault's `attachments/`; the note is fully viewable inside the Tolaria app (text and images in one closed loop).

## Archive layout

```
<vault>/
├── 2026/2026-09-30-<slug>.md           # note: AI summary → metadata table → key facts → full text
└── attachments/2026-09-30-<slug>/      # attachment folder named after the note
    ├── raw.html        # original DOM (untouched)
    ├── snapshot.html   # offline self-contained snapshot (scripts stripped, images local, portable)
    ├── imgNN.<ext>     # images stored under their true format (png/jpg/gif, by magic bytes)
    ├── screenshot.png  # full-page screenshot
    ├── article.md      # verbatim Markdown transcription (each image on its own line)
    └── meta.json       # metadata + image_originals (CDN provenance)
```

Three fidelity layers: **lossless layer** (DOM untouched, all images localized) → **transcription layer** (verbatim Markdown, no rewriting or abridging) → **semantic layer** (AI summary and tags, additive only).

## Creation tooling & model

- **Environment**: built with the ZCode agent (Zhipu ZCode desktop), iterated and verified against real archiving tasks on 2026-09-30
- **Model**: GLM-5.3-Flash (Z.ai GLM series, the ZCode session model)
- **Toolchain used during development**: agent-browser CLI (rendering & capture), python3 + lxml (archive script), Tolaria MCP (note I/O), macOS `sips` (WebP→PNG transcoding)
- **Provenance**: a ZCode derivative of `dsh-plugin-web-archive` (same lineage as `web-page-archive`); the script here is an independent copy carrying four Tolaria-specific fixes (see "Hard-won fixes" below)

## Prerequisites

| Component | Purpose | Install |
|---|---|---|
| **ZCode** (or another agent, see below) | the agent that runs the skill (needs Skill support + MCP) | ZCode CLI / desktop |
| **Tolaria app + vault** | the note vault | install Tolaria, create a vault; register Tolaria's MCP server in your agent and verify the `mcp__tolaria__*` tools are available |
| **python3 + lxml** | archive script | `python3 -m pip install lxml` |
| **agent-browser** | browser rendering (primary path) | `npm i -g agent-browser && agent-browser install`; make sure the npm global bin dir (e.g. `~/.npm-global/bin`) is on PATH |
| **macOS** | WebP→PNG transcoding relies on the system `sips` | recommended; on other OSes WebP images keep their `.webp` form, everything else works |
| browser-use plugin (optional) | in-app-browser fallback channel | only needed if agent-browser is unavailable |

Self-check:

```bash
python3 -c "import lxml; print('lxml ok')"
agent-browser --version          # command not found → fix PATH
# in a session, call mcp__tolaria__list_vaults; your vault list should come back
```

## Install (ZCode)

```bash
git clone git@github.com:skysliently/tolaria-web-archive-skill.git
mkdir -p ~/.agents/skills
cp -R skills/tolaria-web-archive ~/.agents/skills/
```

`~/.agents/skills/` is ZCode's user-level skill directory (where this skill is actually installed); for project-level install use `<project>/.zcode/skills/`. After a session restart, `tolaria-web-archive` appears in the skill list. Claude Code / OpenCode / Codex users: see [Using with other agents](#using-with-other-agents).

## Usage

Any of these in a ZCode session:

- Explicit: `/tolaria-web-archive <url> archive this page`
- Natural language: "save this page to Tolaria" / "archive this article losslessly with images"

Human-verification protocol: when a page shows a CAPTCHA or login wall, the skill keeps the **user-visible browser window** open and asks you to complete the check yourself; capture resumes after your confirmation. The skill never bypasses verification. Non-HTML targets (PDF / video / audio) are out of scope.

## Using with other agents

The skill is not ZCode-bound. Its **hard dependencies are just two**: an agent that can run shell commands, plus a configured Tolaria MCP. agent-browser is a standalone CLI, independent of the agent. Only the skill-discovery mechanism and MCP registration differ:

| Agent | Skill mechanism | Install location | Invocation |
|---|---|---|---|
| **ZCode** | native Skill | `~/.agents/skills/` | `/tolaria-web-archive <url>` or natural language |
| **Claude Code** | native Agent Skills (same SKILL.md format) | `~/.claude/skills/` | `/tolaria-web-archive <url>` or natural language |
| **OpenCode** | SKILL.md (Anthropic skills-format compatible) | `~/.config/opencode/skill/` or project `.opencode/skill/` | natural language (matched via the SKILL.md description) |
| **Codex CLI** | no skills mechanism; route via AGENTS.md | anywhere (e.g. `~/skills/`) | declared in AGENTS.md, then natural language |

> Directory conventions may evolve with tool versions — defer to each tool's official docs.

**Claude Code** (the integration verified while building this repo):

```bash
# 1. install the skill
cp -R skills/tolaria-web-archive ~/.claude/skills/

# 2. register the Tolaria MCP (user scope, available in all projects)
claude mcp add tolaria --scope user --env WS_UI_PORT=9711 -- \
  node /Applications/Tolaria.app/Contents/Resources/mcp-server/index.js

# 3. once the mcp__tolaria__* tools show up in a session, you're set
```

**OpenCode**: copy `skills/tolaria-web-archive/` into its skill directory; register the Tolaria MCP under the `mcp` key of `opencode.json` (same stdio command + `WS_UI_PORT` env var).

**Codex CLI**: no native skills — two equivalent options:
- Global routing (recommended): add a rule to `~/.codex/AGENTS.md`, e.g. "when the user asks to save/archive a web page into Tolaria, first read `~/skills/tolaria-web-archive/SKILL.md` in full and follow it exactly";
- One-off: just say "read `path/to/SKILL.md` and archive <url> accordingly".
Register the MCP in `~/.codex/config.toml`:

```toml
[mcp_servers.tolaria]
command = "node"
args = ["/Applications/Tolaria.app/Contents/Resources/mcp-server/index.js"]

[mcp_servers.tolaria.env]
WS_UI_PORT = "9711"
```

**Cross-agent note**: the tool names cited in SKILL.md (`mcp__tolaria__*`, `mcp__web_reader__webReader`) are ZCode's MCP naming form. Other agents may expose the same server's tools under a different prefix (in Claude Code it is likewise `mcp__tolaria__*`) — defer to the tool list your MCP client injects; the skill's workflow is unaffected.

## Hard-won fixes over the upstream

All of these failed silently at archive time and only surfaced when the note was opened; they are now built into the script and the workflow:

1. **agent-browser eval output is a JSON-encoded string** (quotes and escapes included), which breaks lxml parsing (spurious NO_CONTENT) → the script now detects and decodes it; manual decoding must read into a variable first (`open(p,'w')` truncates the file, so `open(p,'w').write(json.loads(open(p).read()))` destroys the capture).
2. **CDN images mislabeled by extension**: WeChat/Anthropic CDNs serve JPEG/WebP bytes while the old script guessed `.png` from the URL query — Tolaria's preview loads by extension, so mislabeled images are **invisible in the app** (browsers content-sniff and render fine, so the defect stays latent). Now images are stored under their magic-byte-true format, and WebP is transcoded to real PNG (a render-safe format verified in testing).
3. **Image and caption on one line don't render**: source DOMs often flow `<img>` and its caption through one inline container; Tolaria's block editor doesn't parse inline images → the transcription forces every image reference onto its own line (blank lines around it), captions become adjacent paragraphs, text untouched.
4. **Per-document attachment folders**: all artifacts go to `attachments/<note-name>/`; the snapshot references images by same-folder relative paths, so the folder is portable as a unit.

Core principle: the acceptance bar is **renders inside the app**, not "path exists" — a closed reference loop ≠ a visible image.

## Portability notes

- Note paths and frontmatter follow this vault's chronicle convention (`<year>/<date>-<slug>.md`); when adopting into another vault, adjust to your own `AGENTS.md` conventions — the skill body documents its write-side rules in full
- For a first run, pick a simple page to exercise the whole flow before tackling sites behind logins/CAPTCHAs

## Verified environment

- macOS 26 (darwin 25.5.0) arm64 · ZCode · GLM-5.3-Flash
- Tolaria (MCP to a local vault) · agent-browser (Chrome 154.0.8037.92)
- Verified live: a WeChat official-account article and an anthropic.com research report (both without human verification); for walled sites see the intervention protocol in SKILL.md
