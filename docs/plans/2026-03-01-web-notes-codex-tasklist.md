# Codex 执行任务清单：生成 Web 端本地优先笔记项目

目标：让 Codex 可以按本清单从零生成一个可运行的 Web 项目，实现 PRD 的 12 项常用核心能力，并符合浏览器兼容与降级策略。

对应文档：
- PRD：`docs/plans/2026-03-01-web-notes-prd-design.md`
- 实现方案：`docs/plans/2026-03-01-web-notes-implementation-plan.md`

约束（不可违背）：
- 不做 PWA（不加 service worker、离线缓存策略、安装提示等）。
- 不引入“必须依赖后端才能使用”的能力；远端仅预留 `RemoteAdapter` 接口。
- 必须通过 `VaultAdapter` 抽象屏蔽浏览器差异：
  - Chromium：File System Access 真实落盘。
  - Firefox/Safari：导入/导出 + IndexedDB 持久化闭环。

## 0. Codex 一次性提示词（可直接复制给 Codex）

你是一个资深前端工程师。请用 Vite + React + TypeScript 生成一个 Web 笔记应用，满足以下要求：
- 实现 Vault（本地优先）与 12 项功能：文件树、Markdown 编辑/预览/分屏、WikiLink（含 embeds/heading/block）、反链、全文搜索（Worker）、标签、图谱、2 列+Tabs 工作区布局与持久化、命令面板与快捷键、基础设置、zip 导入/导出。
- 必须实现 `VaultAdapter` 抽象，并提供 `FsAccessAdapter`、`IndexedDbAdapter`、`RemoteAdapter(Stub)`。
- Chromium 下使用 File System Access API 直接读写用户选择的文件夹；Firefox/Safari 不支持文件夹读写时，要求提供 zip 导入/导出，并将内容存到 IndexedDB。
- 搜索必须在 Web Worker 中；首次扫描/全量索引不得卡 UI。
- 不做 PWA，不做 Sync/Publish，不做插件系统。

请先生成项目骨架与依赖，再按模块逐步实现，并附带 `README` 的运行说明与浏览器差异说明。

## 1. 工程初始化

- [ ] 创建前端项目（Vite + React + TS，TS strict）
- [ ] 增加基础依赖（建议：CodeMirror6、markdown-it、fflate、flexsearch、zustand）
- [ ] 增加基础脚本：`dev`、`build`、`test`
- [ ] 约定路径与别名（例如 `@/` 指向 `src/`）

验收：
- `pnpm dev` 可启动；无 TS 报错；初始页面可渲染。

## 2. 领域层：Vault 抽象与适配器

- [ ] 定义 `src/vault/VaultAdapter.ts`（读写、列目录、移动、删除、stat、capabilities）
- [ ] 实现 `FsAccessAdapter`（Chromium）
  - [ ] `showDirectoryPicker()` 打开目录
  - [ ] 目录句柄可持久化（IndexedDB 存 handle）以支持“最近 Vault”
  - [ ] 读写 `.md` 文件与目录操作
- [ ] 实现 `IndexedDbAdapter`（Firefox/Safari 降级）
  - [ ] `files` store：key=path，value=content/mtime/size
  - [ ] 目录列表可由路径前缀推导（或维护 folders store）
- [ ] 实现 `RemoteAdapter`（仅 stub，返回“未配置远端”错误）

验收：
- Chromium：可打开目录并列出文件；写入文件后刷新仍存在。
- FF/Safari（或在 Chromium 强制降级模式）：可创建/编辑文件并在刷新后仍存在（IndexedDB）。

## 3. 导入/导出（zip）

- [ ] 导入 zip：选择 zip → 解压 entries → 写入当前 Vault（冲突策略：覆盖/跳过）
- [ ] 导出 zip：遍历 Vault 全部文件 → 打包 zip → 下载
- [ ] 导入多文件（非 zip）作为补充（可选）

验收：
- 导出 zip 解压后目录结构与内容正确；再导入后可恢复。

## 4. Markdown：渲染、解析、锚点与嵌入

- [ ] Markdown 渲染：支持 GFM（表格、任务列表）
- [ ] WikiLink 渲染：
  - [ ] `[[name]]`、`[[name|alias]]` 渲染为可点击链接
  - [ ] `[[name#Heading]]` 生成锚点跳转
  - [ ] `[[name^blockId]]` 跳转到块 id
- [ ] Embed 渲染：`![[name]]` 递归展开（深度上限 + 环路检测）
- [ ] 解析（用于索引）：从 markdown 提取 outLinks、embeds、tags、headings、blocks、plainText
- [ ] heading slug 规则统一（渲染/解析一致）
- [ ] blockId 规则：识别行尾 `^blockId` 并在渲染中附加可跳转 id

验收：
- 点击 `[[New Note]]` 可创建并打开。
- `![[Other]]` 能嵌入且不会因循环嵌入卡死。
- `[[A#Heading]]`、`[[A^block]]` 能定位滚动。

## 5. 索引：链接、反链、标签

- [ ] LinkIndex：维护 out/in/unresolved
- [ ] TagIndex：维护 tag → files 与 file → tags
- [ ] 文件保存/重命名/移动/删除时增量更新索引

验收：
- 反链面板准确显示引用来源；删除链接后能更新。
- 标签面板按频次展示；点击标签可筛选文件列表并打开。

## 6. 搜索：Worker + UI

- [ ] `search.worker.ts`：FlexSearch index 初始化、更新、查询
- [ ] 主线程 client：批量索引 + 增量更新 + QUERY
- [ ] UI：全局搜索框/弹窗，结果高亮，回车打开并定位

验收：
- 输入搜索不阻塞 UI；结果可点击/回车打开；命中片段可见。

## 7. 编辑器：CodeMirror + 预览/分屏

- [ ] CodeMirror 编辑器组件（加载/保存、撤销重做、查找替换）
- [ ] 模式切换：编辑 / 预览 / 分屏
- [ ] 自动保存（防抖）+ 手动保存命令

验收：
- 编辑后内容会保存；切换预览显示正确；分屏可同时操作。

## 8. 文件树与工作区布局

- [ ] 文件树组件：新建/重命名/移动/删除/拖拽
- [ ] 工作区布局：最小实现为“左右 2 列 + Tabs”
- [ ] 布局持久化：每 Vault 保存 layout
- [ ] 移动端降级：单列 Tabs；侧栏抽屉；基础操作可完成

验收：
- 刷新后布局恢复；移动端可完成打开/编辑/搜索/反链查看。

## 9. 命令面板与快捷键

- [ ] 命令注册表（Open Search / Toggle Preview / Export Zip 等）
- [ ] 命令面板 UI（模糊搜索 + 执行）
- [ ] 快捷键系统（可配置、冲突检测）

验收：
- 纯键盘可打开搜索、切换预览、导出 zip。

## 10. 图谱（Graph）

- [ ] 基于 LinkIndex 生成 nodes/edges
- [ ] 交互：缩放/拖拽、点击打开、至少一种筛选（如按标签）
- [ ] 大图提示与降级（可只显示当前文件邻域）

验收：
- 500-2000 节点仍可交互；点击节点可打开笔记。

## 11. 设置与持久化

- [ ] 主题：浅/深色
- [ ] 编辑偏好：字体/字号/行高/自动换行/Tab 宽度（最小集合）
- [ ] 设置持久化（明确：全局 or 每 Vault；建议全局）

验收：
- 设置即时生效；刷新后保持一致。

## 12. 文档与收尾

- [ ] README：运行方式、浏览器差异（Chromium vs FF/Safari）、导入/导出说明
- [ ] 最低单测：WikiLink/Tag/Embed 深度保护/slug 规则

验收：
- 提供一份“手工验收脚本”（按 PRD 12 项逐条操作）。

