# 实现方案与工程计划：Web 端本地优先笔记（常用核心能力）

- 对应 PRD：`docs/plans/2026-03-01-web-notes-prd-design.md`
- 目标：给 Codex/开发直接落地的工程化实现方案（模块、数据结构、接口、关键算法与任务拆分）

## 1. 技术栈（推荐且可替换）

### 1.1 前端工程
- Node.js：20+
- 包管理：pnpm（或 npm）
- 构建：Vite + React + TypeScript
- 测试：Vitest（单测）；可选 Playwright（端到端）

### 1.2 编辑与渲染
- 编辑器：CodeMirror 6
- Markdown 渲染：markdown-it（GFM 支持：表格、任务列表、删除线）
- 语法解析（索引用）：自研轻量解析（正则 + 行扫描）+ 必要时复用 markdown-it token

### 1.3 索引与搜索
- 全文检索：FlexSearch（Worker 内维护 index）
- 链接/标签/块引用：自研索引（内存 + 持久化快照）

### 1.4 图谱
- 实现路径 A（推荐）：D3 force simulation + Canvas/SVG（自渲染，依赖少）
- 实现路径 B（更快集成）：react-force-graph-2d（依赖多但省时间）

### 1.5 导入导出
- zip：fflate（轻量、性能好）

## 2. 架构概览

### 2.1 分层
- UI 层：页面与组件（文件树、编辑器、预览、面板、弹窗）
- 应用层：命令系统、快捷键、路由/布局、状态管理、任务调度
- 领域层：Vault（文件系统抽象）、文档模型（Note）、解析与索引（links/tags/search）
- 基础设施层：适配器（FsAccess / IndexedDB / Remote stub）、Worker 通信、持久化

### 2.2 关键原则
- 所有“会变大/会卡 UI”的操作进 Worker：首次扫描、全文索引构建、全库搜索。
- 所有文件读写走统一 `VaultAdapter`：UI 不关心浏览器差异。
- 所有索引更新走增量：保存一篇笔记只更新该笔记相关索引。

## 3. 目录结构（建议）

```
src/
  app/
    App.tsx
    router.ts
    commands/
      registry.ts
      palette.tsx
      keybindings.ts
    layout/
      model.ts
      persistence.ts
      Workspace.tsx
  vault/
    VaultAdapter.ts
    FsAccessAdapter.ts
    IndexedDbAdapter.ts
    RemoteAdapter.ts
    recentVaults.ts
  markdown/
    render.ts
    wikilinks.ts
    extract.ts
    anchors.ts
    embeds.ts
  index/
    indexer.ts
    linkIndex.ts
    tagIndex.ts
    search/
      protocol.ts
      client.ts
  graph/
    model.ts
    GraphView.tsx
  ui/
    components/
      FileTree.tsx
      SearchDialog.tsx
      BacklinksPanel.tsx
      TagsPanel.tsx
      SettingsView.tsx
  workers/
    search.worker.ts
    index.worker.ts
```

## 4. 核心数据结构（TypeScript 类型草案）

### 4.1 Vault 与文件
- `VaultId`: string（Chromium：基于目录句柄的稳定 id；FF/Safari：基于导入批次/用户命名）
- `VaultFilePath`: string（统一用 POSIX 风格：`folder/note.md`）
- `VaultEntry`:
  - `path: VaultFilePath`
  - `kind: "file" | "folder"`
  - `name: string`
  - `parentPath: VaultFilePath | null`
  - `mtimeMs?: number`（文件）
  - `size?: number`（文件）

### 4.2 文档解析产物（用于索引）
- `ParsedNote`:
  - `path`
  - `title`（首个 H1 或文件名）
  - `headings: { text, slug, level, line }[]`
  - `blocks: { id, line }[]`（来自 `^blockId`）
  - `outLinks: { targetPathOrName, targetHeadingSlug?, targetBlockId?, alias? }[]`
  - `embeds: { targetPathOrName, targetHeadingSlug?, targetBlockId? }[]`
  - `tags: string[]`（含 `a/b` 形式）
  - `plainText`（用于全文检索）

### 4.3 索引（内存态）
- `LinkIndex`:
  - `out: Map<path, Set<targetPath>>`
  - `in: Map<path, Set<sourcePath>>`
  - `unresolvedMentions: Map<missingTargetName, Set<sourcePath>>`
- `TagIndex`:
  - `tagToFiles: Map<tag, Set<path>>`
  - `fileToTags: Map<path, Set<tag>>`
- `SearchIndex`：Worker 内（FlexSearch）

## 5. VaultAdapter 抽象（必须稳定）

### 5.1 接口（示例）
- `listDir(path): Promise<VaultEntry[]>`
- `readTextFile(path): Promise<string>`
- `writeTextFile(path, content): Promise<void>`
- `mkdir(path): Promise<void>`
- `move(from, to): Promise<void>`
- `remove(path, { recursive }): Promise<void>`
- `stat(path): Promise<{ mtimeMs, size }>`
- `exists(path): Promise<boolean>`
- `getCapabilities(): { kind: "fsaccess" | "indexeddb" | "remote"; supportsNativeFolder: boolean }`

### 5.2 FsAccessAdapter（Chromium）
- 使用 `showDirectoryPicker()` 获取目录句柄；在 IndexedDB 持久化 handle 以支持“最近 Vault”
- 路径解析：以目录句柄为根，逐级 `getDirectoryHandle/getFileHandle`
- 变更检测：不做实时 watch；提供“手动刷新索引”命令；页面重新聚焦时可触发轻量 rescan（对比 mtime）

### 5.3 IndexedDbAdapter（Firefox/Safari）
- object store：`files`（key=path，value={content, mtimeMs, size}），`folders`（可选）
- 导入时写入；编辑保存时更新
- 导出时遍历所有 `files` 生成 zip

### 5.4 RemoteAdapter（预留）
- 仅实现接口签名与错误提示（例如抛出 `RemoteNotConfiguredError`）
- 不包含鉴权、同步、冲突解决逻辑

## 6. Markdown 能力实现细则

### 6.1 WikiLink 解析
支持：
- `[[name]]`
- `[[name|alias]]`
- `[[name#Heading]]`
- `[[name^blockId]]`
- `![[name]]`（embed）

解析策略（索引用）：
- 行扫描 + 正则提取 `!?\[\[...\]\]`
- 对 `...` 内部按 `|` 分离 alias；按 `#` 或 `^` 分离目标锚点
- `name` 到 `path` 的解析：优先匹配同名 `.md` 文件（允许多个同名时给出 disambiguation UI：最小实现可选“打开搜索结果让用户选”）

### 6.2 Heading 锚点（`#Heading`）
- 渲染侧：为标题生成 `id`（slug）
- 解析侧：复用同一套 slug 规则（建议使用 GitHub slugger 兼容方式；或自研 slugify 并保证一致）

### 6.3 Block 引用（`^blockId`）
- 解析侧：识别块 id（建议：当行尾出现 `^id` 且 `id` 为 `[A-Za-z0-9_-]+`）
- 渲染侧：为包含 `^id` 的块元素增加 `id="blockid"`（并在预览中隐藏 `^id` 文本或渲染为弱化）
- 跳转：点击 `[[A^block]]` 时打开 A 并滚动到对应块 id

### 6.4 Embed（`![[...]]`）
- 渲染时递归展开目标笔记内容
- 循环与深度保护：
  - 最大深度：例如 3
  - 使用 `visitedPaths` 检测环路；遇到环路显示占位提示

## 7. 索引与 Worker 通信

### 7.1 IndexWorker（可选但推荐）
职责：
- 首次扫描/导入后，批量解析所有笔记并产出 `ParsedNote` 列表
- 增量更新：接收单文件内容更新，返回该文件的解析产物

### 7.2 SearchWorker（必须）
职责：
- 维护 FlexSearch 索引：add/update/remove
- 提供查询：返回命中文档列表与片段（片段可在主线程做高亮也可在 Worker 返回）

### 7.3 通信协议（建议）
使用 `postMessage` + `MessagePort`（或 Comlink）：
- `INIT_VAULT({ vaultId })`
- `BULK_INDEX({ docs: {path, title, plainText}[] })`
- `UPDATE_DOC({ path, title, plainText })`
- `REMOVE_DOC({ path })`
- `QUERY({ q, limit }) -> { results: {path, score, highlights?}[] }`

## 8. UI 与交互实现要点

### 8.1 工作区布局（最小可用实现）
实现“2 列 + Tabs”模型即可满足 PRD 的最低要求：
- 左列：Tab 列表 + 当前 Tab 内容
- 右列：Tab 列表 + 当前 Tab 内容
- 操作：在命令/菜单中选择“在右侧打开/在左侧打开/在当前打开”
- 持久化：`layout = { splitRatio, leftTabs, rightTabs, activeLeft, activeRight }`

移动端降级：
- 单列 Tabs；左右列概念隐藏；“在右侧打开”映射为“新 Tab 打开”

### 8.2 命令面板
- 命令数据结构：`{ id, title, keywords?, run(), defaultShortcut? }`
- UI：输入框 + 列表（上下键选择，回车执行）

### 8.3 快捷键配置
- `keybindings: Record<commandId, string[]>`
- 冲突检测：同一作用域下同快捷键绑定多个命令时提示并要求用户确认

### 8.4 反链面板
- 当前文件 `path` → `LinkIndex.in.get(path)` 得到来源列表
- 展示上下文：来源文件中命中链接附近若干字符（可通过简单 substring，或在解析阶段保留 link 的位置范围）

### 8.5 标签面板
- 标签列表：按出现频次排序
- 点击标签：展示文件列表（可用搜索结果 UI 复用）

## 9. 导入/导出实现

### 9.1 导入 zip
- 解析 zip entries → 过滤目录/非文本（最小实现先支持 `.md` 与纯文本）
- 冲突策略（至少覆盖/跳过）：由导入弹窗选择
- 写入目标：
  - IndexedDbAdapter：直接写入 `files` store
  - FsAccessAdapter：在选定目录下创建/覆盖文件（需权限）

### 9.2 导出 zip
- 列出所有文件（递归）
- 以 path 为 zip entry 名称写入
- 触发浏览器下载

## 10. 测试策略（建议最低配）
- 单测（Vitest）：
  - WikiLink 解析（含 alias、heading、block、embed）
  - Tag 提取（含层级）
  - Heading slug 规则一致性（渲染与解析）
  - Embed 深度/循环保护
- 端到端（可选 Playwright）：
  - 导入 zip → 搜索 → 打开 → 编辑 → 导出 zip

## 11. 交付清单（工程产物）
- 可运行的 Web 应用（`pnpm dev`）
- 可构建的静态产物（`pnpm build`）
- PRD 范围 12 项功能均可演示
- README：环境要求、浏览器差异说明、导入导出使用说明

