# ECC Agent Memory 管理报告

本文基于 `everything-claude-code` 当前仓库实现，梳理 ECC 在 Claude hook、observer、metrics bridge、JS state store、ECC2 Rust runtime 中的 agent memory 管理机制。目标不是泛泛谈“有记忆能力”，而是把 memory 的采集、过滤、持久化、回注、检索、压缩和治理边界拆到 function level。

这份报告覆盖四种 memory：

1. 会话连续性 memory：让下一次会话能接上一次工作。
2. 行为学习 memory：让 observer 从工具使用中提炼 instinct 和 learned skill。
3. 运行态 operational memory：让 statusline、dashboard、monitor 快速读取当前 session 的最近状态。
4. 结构化长期 memory：把 session、decision、message、file activity 沉淀进 SQLite 和 context graph。

## 1. 执行摘要

ECC 不是单一的 memory 子系统，而是多层 memory surface 的组合：

- Markdown session memory：面向 Claude hook，落在 `~/.claude/session-data/*.tmp`。
- Observer project memory：面向 continuous learning，落在 homunculus project 目录。
- Metrics bridge memory：面向 statusline 和快速监控，落在 `/tmp/ecc-metrics-<session>.json` 和 `~/.claude/metrics/*.jsonl`。
- SQLite control-plane memory：面向 ECC2 调度、检索、冲突分析和 context graph，落在 `~/.claude/ecc/state.db`。

这四层的分工不同：

- Session tmp 文件解决“下次打开还能接上”。
- Observer 解决“系统能否从重复行为中学到模式”。
- Metrics bridge 解决“高频读取是否足够轻”。
- SQLite/context graph 解决“事实是否可查询、可压缩、可回忆”。

如果用一句话概括 ECC 当前 memory 架构，它是一个 event-first、log-first、hook-first 的系统，然后在 Rust `StateStore` 里逐步图结构化，而不是一个一开始就做 embedding/RAG 的系统。

## 2. 总体架构

### 2.1 Memory Surface 一览

| 层级 | 存储位置 | 主要消费者 | 目标 |
| --- | --- | --- | --- |
| Session continuity | `~/.claude/session-data/*.tmp` | `session-start.js` | 恢复上次工作上下文 |
| Legacy session compatibility | `~/.claude/sessions/*.tmp` | `session-start.js`, `session-manager.js` | 向后兼容旧安装 |
| Learned skills | `~/.claude/skills/learned/` | `session-start.js` | 把 learned skill 摘要注入新会话 |
| Observer project memory | `~/.local/share/ecc-homunculus/projects/<projectId>/...` | observer loop, `session-start.js` | project-scoped continuous learning |
| Tool activity log | `~/.claude/metrics/tool-usage.jsonl` | `ecc-metrics-bridge.js`, Rust `StateStore` | 记录每次工具调用与文件活动 |
| Cost tracker log | `~/.claude/metrics/costs.jsonl` | `ecc-metrics-bridge.js`, Rust `StateStore` | 汇总 token/cost |
| Session bridge | `/tmp/ecc-metrics-<session>.json` | statusline, context monitor | 提供低延迟会话聚合视图 |
| Durable state store | `~/.claude/ecc/state.db` | JS CLI, ECC2 runtime | 持久化结构化控制面 memory |

### 2.2 主数据流

#### 启动时

1. `SessionStart` hook 触发。
2. `session-start.js` 清理过期 session 文件，注册 observer lease。
3. 它从 canonical 和 legacy session 目录选择最匹配当前 worktree/project 的最近 session。
4. 它把 historical session summary、active instincts、learned skill 摘要合并后写回 `additionalContext`。

#### 运行中

1. 每次 `PostToolUse` 触发 `session-activity-tracker.js`，把工具调用写入 `tool-usage.jsonl`。
2. 同次 hook 触发 `ecc-metrics-bridge.js`，把当前 session 的聚合指标写入 `/tmp/ecc-metrics-<session>.json`。
3. observer 的 `observe-runner.js` 把 pre/post observation 送给 `skills/continuous-learning-v2/hooks/observe.sh`。
4. ECC2 Rust `StateStore` 定期把 metrics JSONL 同步进 SQLite `tool_log` 和 `sessions`，同时更新 context graph。

#### 停止或压缩时

1. `pre-compact.js` 记录 compaction 事件。
2. `session-end.js` 从 transcript JSONL 生成或更新 markdown session summary。
3. 下一次会话启动时，这份 summary 会经过匹配、截断、加 guard 之后重新注入。

## 3. 第一层：会话连续性 Memory

这层主要由 `scripts/hooks/session-start.js`、`scripts/hooks/session-end.js`、`scripts/hooks/pre-compact.js` 和 `scripts/lib/session-manager.js` 组成。

### 3.1 基础路径与 ID 生成

文件：`scripts/lib/utils.js`

关键函数：

- `getClaudeDir()`
  - 返回 `~/.claude`。
  - 所有 Claude hook 侧 memory 都从这里分叉。

- `getSessionsDir()`
  - 返回 canonical session 目录 `~/.claude/session-data`。

- `getLegacySessionsDir()`
  - 返回 legacy 目录 `~/.claude/sessions`。

- `getSessionSearchDirs()`
  - 以 canonical-first 顺序返回 session 搜索目录。
  - 这是兼容旧装机和新装机的统一入口。

- `sanitizeSessionId(raw)`
  - 把原始 session id 或项目名清洗为文件名安全片段。
  - 处理非法字符、Windows 保留名、纯非 ASCII 输入。

- `getSessionIdShort(fallback = 'default')`
  - 优先使用 `CLAUDE_SESSION_ID` 尾部 8 位。
  - 否则退化到当前项目名。
  - 这是 session summary 文件命名的核心逻辑。

### 3.2 SessionStart：历史摘要回注

文件：`scripts/hooks/session-start.js`

#### 3.2.1 目录与上下文控制函数

- `getSessionRetentionDays()`
  - 读取 `ECC_SESSION_RETENTION_DAYS`。
  - 决定 session 文件保留窗口。

- `isSessionStartContextDisabled()`
  - 读取 `ECC_SESSION_START_CONTEXT`。
  - 支持 `0/false/off/none/disabled` 关闭注入。

- `getSessionStartMaxContextChars()`
  - 读取 `ECC_SESSION_START_MAX_CHARS`。
  - 控制注入总长度。

- `limitSessionStartContext(additionalContext, maxChars)`
  - 超过上限时截断。
  - 会追加 truncation marker，防止无提示丢内容。

#### 3.2.2 Session 文件发现与筛选函数

- `normalizePath(p)`
  - 调用 `fs.realpathSync` 做规范化。
  - 用于 worktree 精确匹配，避免 symlink 或大小写差异导致误判。

- `dedupeRecentSessions(searchDirs)`
  - 在多个 search dir 中查找 7 天内的 `*-session.tmp`。
  - 以 basename 去重。
  - 同名文件取最新 mtime；mtime 相同则优先目录优先级更高的那个。

- `pruneExpiredSessions(searchDirs, retentionDays)`
  - 扫描 session 目录并删除超龄 `.tmp` 文件。
  - 不抛致命错误，失败只记录 warning。

- `getSessionStartMode(rawInput)`
  - 解析 hook stdin。
  - 识别 `startup/resume/clear/compact`。
  - 只有 `startup` 继续做旧 summary 注入，其他模式跳过。

- `selectMatchingSession(sessions, cwd, currentProject)`
  - 当前 continuity 逻辑的核心函数。
  - 读取候选 session 文件头部中的 `**Worktree:**` 和 `**Project:**`。
  - 匹配优先级：
    1. 同 worktree。
    2. 同 project 且 session 没有 worktree 字段，作为 legacy 兼容。
    3. 否则不注入。
  - 它把选中的 `content` 一并返回，避免重复读文件。

#### 3.2.3 Learned memory 注入函数

- `parseInstinctFile(content)`
  - 解析 instinct 文档 frontmatter 和正文。
  - 支持 `confidence` 数值提取。

- `readInstinctsFromDir(directory, scope)`
  - 遍历 instinct 目录并解析每个文件。
  - 为每条 instinct 增加 `_scopeLabel` 和 `_sourceFile`。

- `extractInstinctAction(content)`
  - 从 `## Action` 区块提取最关键的一行。
  - 用于 session start 的摘要注入。

- `summarizeActiveInstincts(observerContext)`
  - 读取 project/global 两级 instinct 目录。
  - 以 `INSTINCT_CONFIDENCE_THRESHOLD = 0.7` 过滤。
  - project scope 优先覆盖 global scope。
  - 最终按 confidence 排序并限制为 6 条。

- `stripMarkdownInline(value)`
  - 清理 markdown 行内标记，便于摘要注入。

- `collapseWhitespace(value)`
  - 把多空白压成单空格。

- `truncateSummary(value, maxLength)`
  - learned skill 和 instinct 摘要共用的截断逻辑。

- `extractMarkdownHeading(content)`
  - 抽 `# heading`。

- `extractSection(content, headingPattern)`
  - 抽取某个二级标题区块内容。

- `extractFirstParagraph(content)`
  - fallback 摘要来源。

- `summarizeLearnedSkillFile(filePath, learnedRoot)`
  - 从 learned skill 文件里提取标题、slug、summary、mtime。

- `collectLearnedSkillFiles(learnedDir)`
  - 收集 learnedDir 下平铺 markdown 与目录式 `SKILL.md`。

- `summarizeLearnedSkills(learnedDir, learnedSkillFiles)`
  - 生成“Available learned skills”启动摘要。
  - 最多注入 6 条。

#### 3.2.4 SessionStart 主流程

- `main()`
  - 做以下事情：
    - 初始化 session/learned skill 目录。
    - 清理过期 session。
    - 用 `writeSessionLease()` 注册 observer lease。
    - 按 mode 决定是否注入旧 summary。
    - 注入 instinct 和 learned skill 摘要。
    - 报告 aliases、package manager、project type。
    - 通过 `writeSessionStartPayload()` 把最终 `additionalContext` 写回 hook output。
  - 里面最关键的安全设计是 stale replay guard：
    - 把 prior session summary 包成 “HISTORICAL REFERENCE ONLY” 块。
    - 明确要求模型不要重放历史 skill invocation 或 `ARGUMENTS=`。

- `writeSessionStartPayload(additionalContext)`
  - 把结构化 JSON 输出到 stdout。
  - hook consumer 用这个字段把 memory 注入当前会话。

### 3.3 SessionEnd：摘要提炼与写回

文件：`scripts/hooks/session-end.js`

#### 3.3.1 Transcript 抽取函数

- `extractSessionSummary(transcriptPath)`
  - 读取 transcript JSONL。
  - 抽取：
    - 最近 10 条 user messages。
    - 最多 20 个 tool 名称。
    - 最多 30 个修改过的文件。
  - 兼容三类格式：
    - `entry.type === 'user'`
    - `entry.role === 'user'`
    - `entry.message?.role === 'user'`
  - 兼容 direct `tool_use` 和 assistant content block 里的 `tool_use`。
  - 过滤 ANSI 噪声，解析失败行只记数不阻塞。

#### 3.3.2 Header 管理函数

- `getSessionMetadata()`
  - 收集 `project`、`branch`、`worktree`。

- `extractHeaderField(header, label)`
  - 从已有 markdown header 中抽字段。

- `buildSessionHeader(today, currentTime, metadata, existingContent)`
  - 构造统一 header。
  - 保留已有 `heading/date/started`，刷新 `Last Updated`。

- `mergeSessionHeader(content, today, currentTime, metadata)`
  - 只更新 header，不破坏后续 body。

#### 3.3.3 Summary block 管理函数

- `buildSummarySection(summary)`
  - 以 markdown 形式渲染：
    - `### Tasks`
    - `### Files Modified`
    - `### Tools Used`
    - `### Stats`

- `buildSummaryBlock(summary)`
  - 用 `<!-- ECC:SUMMARY:START -->` / `<!-- ECC:SUMMARY:END -->` 包裹 generated section。
  - 这样 repeated Stop hook 可以只替换 generated block，不碰用户手写内容。

- `escapeRegExp(value)`
  - 给 marker-based replace 提供安全正则。

#### 3.3.4 SessionEnd 主流程

- `main()`
  - 解析 stdin JSON 的 `transcript_path`，也支持 env fallback。
  - 用 transcript UUID 派生 `shortId`，确保子进程 session 不会互相覆盖 summary 文件。
  - 生成 session 文件路径 `YYYY-MM-DD-<shortId>-session.tmp`。
  - 如果文件已存在：
    - merge header。
    - 只替换 generated summary block。
    - 老文件没有 marker 时走 migration 路径。
  - 如果文件不存在：
    - 创建新模板。
  - 这是 ECC 当前会话连续性 memory 的落盘点。

### 3.4 PreCompact：压缩边界记录

文件：`scripts/hooks/pre-compact.js`

关键函数：

- `main()`
  - 确保 sessionsDir 存在。
  - 追加 `compaction-log.txt`。
  - 找当前最近 session 文件并追加 compaction 说明。

这层的作用很窄，但重要：

- 它不生成摘要。
- 它只标记“这里发生过压缩”。
- 这给后续人工或系统判断 memory 丢失边界提供了审计点。

### 3.5 Session 文件读取与管理 API

文件：`scripts/lib/session-manager.js`

关键函数：

- `parseSessionFilename(filename)`
  - 解析 session 文件名。
  - 做日期合法性校验，不只看 regex。

- `getSessionPath(filename)`
  - canonical sessionsDir 拼路径。

- `getSessionCandidates(options)`
  - 从 session search dirs 枚举符合条件的文件。
  - 返回 metadata、mtime、size、content existence 等。

- `buildSessionRecord(sessionPath, metadata)`
  - 从 path + metadata 构造标准记录。

- `sessionMatchesId(metadata, normalizedSessionId)`
  - 支持 short id、完整文件名、legacy no-id 格式匹配。

- `getMatchingSessionCandidates(normalizedSessionId)`
  - 找某个 session id 的所有候选文件。

- `getSessionContent(sessionPath)`
  - 读原始 markdown。

- `parseSessionMetadata(content)`
  - 从 markdown 中解析结构化字段：
    - title/date/started/lastUpdated/project/branch/worktree
    - completed/inProgress/notes/context

- `getSessionStats(sessionPathOrContent)`
  - 给 session content 做轻量统计。

- `getAllSessions(options)`
  - 支持 limit/offset/date/search。

- `getSessionById(sessionId, includeContent)`
  - 获取单个 session 记录。

- `getSessionTitle(sessionPath)`
  - 取标题。

- `getSessionSize(sessionPath)`
  - 取人类可读大小。

- `writeSessionContent(sessionPath, content)`
  - 覆写 session 文件。

- `appendSessionContent(sessionPath, content)`
  - 追加内容。

- `deleteSession(sessionPath)`
  - 删除 session 文件。

- `sessionExists(sessionPath)`
  - 检查文件是否存在。

## 4. 第二层：Observer / Continuous Learning Memory

这层的核心不在“恢复 transcript”，而在“从行为中学模式”。

### 4.1 Project-scoped observer 基础设施

文件：`scripts/lib/observer-sessions.js`

关键函数：

- `getHomunculusDir()`
  - 决定 observer memory 根目录。
  - 支持 `CLV2_HOMUNCULUS_DIR` 和 `XDG_DATA_HOME` 覆盖。

- `getProjectsDir()`
  - 返回 `<homunculus>/projects`。

- `getProjectRegistryPath()`
  - 返回 `<homunculus>/projects.json`。

- `readProjectRegistry()`
  - 读取 project registry。

- `runGit(args, cwd)`
  - observer 侧 git helper。

- `stripRemoteCredentials(remoteUrl)`
  - 删除 remote URL 中的凭据。

- `normalizeRemoteUrl(remoteUrl)`
  - 把 remote URL 正规化。
  - 这是 project identity 稳定化的重要步骤。

- `resolveProjectRoot(cwd = process.cwd())`
  - 优先用 `CLAUDE_PROJECT_DIR`。
  - 否则回退到 git root。
  - 没有项目根时返回空串。

- `computeProjectId(projectRoot)`
  - 基于 remote URL 或 project root 计算稳定 project id。

- `resolveProjectContext(cwd = process.cwd())`
  - 返回：
    - `projectId`
    - `projectRoot`
    - `projectDir`
    - `isGlobal`
  - 这是 observer memory 多项目隔离的核心函数。

- `getObserverPidFile(context)`
  - `.observer.pid`

- `getObserverSignalCounterFile(context)`
  - `.observer-signal-counter`

- `getObserverActivityFile(context)`
  - `.observer-last-activity`

- `getSessionLeaseDir(context)`
  - `.observer-sessions`

- `resolveSessionId(rawSessionId)`
  - 用 `utils.sanitizeSessionId()` 生成 observer 安全 session id。

- `getSessionLeaseFile(context, rawSessionId)`
  - 返回某 session 的 observer lease 文件路径。

- `writeSessionLease(context, rawSessionId, extra)`
  - 写 lease JSON。
  - SessionStart 会调用它。
  - 这是 observer 判断“这个 project 还有活 session 吗”的依据。

- `removeSessionLease(context, rawSessionId)`
  - 清理 lease。

- `listSessionLeases(context)`
  - 枚举 lease 文件。

- `stopObserverForContext(context)`
  - 根据 `.observer.pid` 给 observer 发 `SIGTERM`。
  - 清理 pid file 和 signal counter。

### 4.2 Observe hook 包装器

文件：`scripts/hooks/observe-runner.js`

关键函数：

- `getPluginRoot(options)`
  - 定位 ECC 插件根路径。

- `resolveTarget(rootDir, relPath)`
  - 阻断路径穿越。

- `toShellPath(filePath)`
  - 给 Windows shell 做路径转换。

- `findShellBinary()`
  - 找可用 `bash/sh`。

- `getPhaseFromHookId(hookId)`
  - 只允许 `pre` 或 `post`。

- `getTimeoutMs()`
  - 控制 observe hook 最长执行时间。

- `combineStderr(stderr, message)`
  - 合并错误信息。

- `run(raw, options)`
  - 真正调用 `skills/continuous-learning-v2/hooks/observe.sh`。
  - 失败降级为非阻塞 warning。

- `emitHookResult(raw, output)`
  - 保证 hook 即便降级也把原 stdin/stdout 流传递下去。

### 4.3 Observer memory 的治理约束

从 `tests/hooks/observer-memory.test.js` 可以看出，observer memory 当前有明确的防爆治理：

- 节流
  - observation 不是每次都立刻 signal observer。
  - 使用 `.observer-signal-counter` 聚合触发。

- 采样
  - observer 分析只看最近 observation tail，而不是整个历史文件。

- 防重入
  - observer loop 有 `ANALYZING` 和 cooldown，防止 signal 风暴。

- session-aware idle 退出
  - observer 会看 `.observer-sessions` 下是否还有 active lease。

这一层的结论是：observer memory 是 project-scoped、自治式、慢变量记忆，不是 request-time 的上下文恢复机制。

## 5. 第三层：运行态 Operational Memory

这层服务于高频读取场景，不追求语义丰富，追求低成本读取。

### 5.1 Tool activity 采集

文件：`scripts/hooks/session-activity-tracker.js`

#### 5.1.1 脱敏与压缩

- `redactSecrets(value)`
  - 删除 token、Authorization、AWS key、GitHub token、password 等。

- `truncateSummary(value, maxLength = 220)`
  - 统一摘要长度。

- `sanitizeParamValue(value, depth = 0)`
  - 控制对象深度、数组长度、字段数。

- `sanitizeInputParams(toolInput)`
  - 把工具参数转成安全 JSON。

#### 5.1.2 文件路径与文件事件抽取

- `pushPathCandidate(paths, value)`
  - 推入 file path 候选，过滤 URL 和 MCP path。

- `pushFileEvent(events, value, action, diffPreview, patchPreview)`
  - 去重后追加结构化 file event。

- `inferDefaultFileAction(toolName)`
  - 从工具名推断 `read/create/modify/delete/move/touch`。

- `actionForFileKey(toolName, key)`
  - 根据 `source_path/destination_path/file_path` 精细决定 action。

- `collectFilePaths(value, paths)`
  - 递归扫描对象里的 file path 字段。

- `extractFilePaths(toolInput)`
  - 对外导出的 file path 抽取器。

- `collectFileEvents(toolName, value, events, key, parentValue)`
  - 递归抽结构化 file events。

- `extractFileEvents(toolName, toolInput)`
  - 对外导出的 file event 抽取器。

#### 5.1.3 Diff/Patch preview 生成

- `sanitizeDiffText()`
- `sanitizePatchLines()`
- `buildReplacementPreview()`
- `buildCreationPreview()`
- `buildPatchPreviewFromReplacement()`
- `buildPatchPreviewFromContent()`
- `buildDiffPreviewFromPatchPreview()`

这一组函数的作用是把具体编辑动作压缩成短 preview，供后续 file activity 和 context graph 使用。

#### 5.1.4 Working tree 补强

- `runGit(args, cwd)`
  - 运行 git 子命令。

- `gitRepoRoot(cwd)`
  - 找 repo root。

- `candidateGitPaths(repoRoot, filePath)`
  - 为一个 file path 生成多种 git 路径候选。

- `patchPreviewFromGitDiff(repoRoot, pathCandidates)`
  - 从 `git diff` 中抽取有限 patch 片段。

- `trackedInGit(repoRoot, pathCandidates)`
  - 判断文件是否已被 git 跟踪。

- `enrichFileEventFromWorkingTree(toolName, event)`
  - 如果是 `Write`，但文件已被跟踪且有 diff，则把 action 修正为 `modify`。
  - 尽可能补全 diff preview / patch preview。

#### 5.1.5 Activity row 生成

- `summarizeInput(toolName, toolInput, filePaths)`
  - 为输入构造轻量摘要。

- `summarizeOutput(toolOutput)`
  - 为输出构造轻量摘要。

- `buildActivityRow(input, env = process.env)`
  - 生成最终行：
    - `session_id`
    - `tool_name`
    - `input_summary`
    - `input_params_json`
    - `output_summary`
    - `file_paths`
    - `file_events`

- `run(rawInput)`
  - 把 row 追加到 `~/.claude/metrics/tool-usage.jsonl`。

- `main()`
  - stdin reader。

### 5.2 Session metrics bridge

文件：`scripts/hooks/ecc-metrics-bridge.js`

关键函数：

- `toNumber(value)`
  - 数值归一化。

- `stableStringify(value, depth = 0)`
  - 稳定序列化输入，用于 hash。

- `hashToolCall(toolName, toolInput)`
  - 把工具调用压成 8 字符 hash。
  - 用于 loop detection。

- `extractFilePaths(toolName, toolInput)`
  - 提取被修改的文件。

- `readSessionCost(sessionId)`
  - 从 `costs.jsonl` 尾部读取当前 session 累积 cost/token。

- `run(rawInput)`
  - 读取或初始化 bridge JSON。
  - 更新：
    - `tool_count`
    - `files_modified`
    - `recent_tools`
    - `total_cost_usd`
    - `total_input_tokens`
    - `total_output_tokens`
  - 最终落到 `/tmp/ecc-metrics-<session>.json`。

### 5.3 Bridge 公共库

文件：`scripts/lib/session-bridge.js`

关键函数：

- `sanitizeSessionId(raw)`
  - 专门用于 tmp bridge 文件命名的 session id 清洗。

- `getBridgePath(sessionId)`
  - 生成 bridge 文件路径。

- `readBridge(sessionId)`
  - 读 JSON，失败返回 null。

- `writeBridgeAtomic(sessionId, data)`
  - `.tmp + rename` 原子写。

- `resolveSessionId()`
  - 从 `ECC_SESSION_ID` 或 `CLAUDE_SESSION_ID` 读取当前 session id。

### 5.4 这一层的定位

这层 memory 的特点：

- 读多写多，但要求轻。
- 不追求完整 transcript。
- 不追求富语义。
- 服务 statusline、context monitor、冲突检测、loop 检测。

它更像操作系统里的 working set，而不是知识库。

## 6. 第四层：JS State Store

JS state store 是 Node 侧对 durable memory 的统一 API。

### 6.1 DB 启动与 driver 包装

文件：`scripts/lib/state-store/index.js`

关键函数：

- `resolveStateStorePath(options = {})`
  - 决定 DB 路径。
  - 默认是 `~/.claude/ecc/state.db`。

- `wrapSqlJsDatabase(rawDb, dbPath)`
  - 给 sql.js 包装出类 `better-sqlite3` 的 API。
  - 自动处理事务外落盘。

- `openDatabase(SQL, dbPath)`
  - 负责打开现有 DB 或创建新 DB。
  - 尝试开启 foreign keys 和 WAL。

- `createStateStore(options = {})`
  - 应用 migrations。
  - 初始化 query API。
  - 暴露 schema validator。

### 6.2 Schema 校验

文件：`scripts/lib/state-store/schema.js`

关键函数：

- `readSchema()`
  - 读 `schemas/state-store.schema.json`。

- `getAjv()`
  - 构造 AJV 实例。

- `getEntityValidator(entityName)`
  - 为实体生成 validator。

- `formatValidationErrors(errors)`
  - 格式化报错。

- `validateEntity(entityName, payload)`
  - 返回 `{valid, errors}`。

- `assertValidEntity(entityName, payload, label)`
  - 无效实体直接抛异常。

### 6.3 Migrations

文件：`scripts/lib/state-store/migrations.js`

关键函数：

- `ensureMigrationTable(db)`
  - 确保 `schema_migrations` 存在。

- `getAppliedMigrations(db)`
  - 返回已应用 migration 列表。

- `applyMigrations(db)`
  - 依序执行 `MIGRATIONS`。
  - 当前包含：
    - `001_initial_state_store`
    - `002_work_items`

### 6.4 Query API

文件：`scripts/lib/state-store/queries.js`

#### 6.4.1 输入归一化

- `normalizeSessionInput()`
- `normalizeSkillRunInput()`
- `normalizeSkillVersionInput()`
- `normalizeDecisionInput()`
- `normalizeInstallStateInput()`
- `normalizeGovernanceEventInput()`
- `normalizeWorkItemInput()`

这一组函数统一默认值和时间戳，是 Node 侧 durable memory 的写入门卫。

#### 6.4.2 结果映射

- `mapSessionRow()`
- `mapSkillRunRow()`
- `mapSkillVersionRow()`
- `mapDecisionRow()`
- `mapInstallStateRow()`
- `mapGovernanceEventRow()`
- `mapWorkItemRow()`

#### 6.4.3 汇总与 readiness 计算

- `classifyOutcome()`
- `classifyWorkItemStatus()`
- `toPercent()`
- `summarizeSkillRuns()`
- `summarizeInstallHealth()`
- `summarizeWorkItems()`
- `summarizeReadiness()`

#### 6.4.4 主 API 装配

- `createQueryApi(db)`
  - 预编译 statements。
  - 返回一组高层接口，包括：
    - `upsertSession`
    - `insertSkillRun`
    - `upsertSkillVersion`
    - `insertDecision`
    - `upsertInstallState`
    - `insertGovernanceEvent`
    - `upsertWorkItem`
    - `listRecentSessions`
    - `getSessionDetail`
    - `listWorkItems`
    - `getStatus`

JS state store 的角色不是 hook continuity，而是 Node CLI 和管理脚本的结构化事实层。

## 7. 第五层：ECC2 Rust StateStore

Rust `StateStore` 是 ECC 当前最完整的长期 memory 核心。

文件：`ecc2/src/session/store.rs`

### 7.1 DB 初始化与 schema 演化

- `StateStore::open(path)`
  - 打开 SQLite。
  - 开启 foreign keys。
  - 设置 busy timeout。
  - 调用 `init_schema()`。

- `StateStore::init_schema()`
  - 建表并兼容旧 schema。
  - 当前承载的 memory 维度包括：
    - `sessions`
    - `tool_log`
    - `session_profiles`
    - `messages`
    - `session_output`
    - `decision_log`
    - `context_graph_entities`
    - `context_graph_relations`
    - `context_graph_observations`
    - `context_graph_connector_checkpoints`
    - 冲突和 daemon activity 相关表

这一步决定了 ECC2 memory 的统一事实模型。

### 7.2 Session 基本事实

- `insert_session(&self, session)`
  - 插入 session 主记录。

- `upsert_session_profile(&self, ...)`
  - 记录 session profile。

- `get_session_profile(&self, session_id)`
  - 读取 profile。

- `update_state_and_pid(&self, session_id, state, pid)`
  - 原子刷新 session state 和 pid。

- `update_state(&self, session_id, state)`
  - 只刷新状态。

- `update_pid(&self, session_id, pid)`
  - 只刷新 pid。

- `update_metrics(&self, session_id, metrics)`
  - 写 token/tool/file/cost/duration。

- `refresh_session_durations(&self)`
  - 根据 state 和时间戳重算 duration。

- `touch_heartbeat(&self, session_id)`
  - 刷新 `last_heartbeat_at`。

- `list_sessions(&self)`
  - 列出所有 session。

- `get_latest_session(&self)`
  - 最近 session。

- `get_session(&self, id)`
  - 单 session 查询。

- `delete_session(&self, session_id)`
  - 删除 session 及其关联 output/tool_log/messages。

### 7.3 Metrics 同步

- `sync_cost_tracker_metrics(&self, metrics_path)`
  - 把 cost tracker JSONL 汇总回 `sessions`。

- `sync_tool_activity_metrics(&self, metrics_path)`
  - 把 `tool-usage.jsonl` 导入 `tool_log`。
  - 聚合每个 session 的 `tool_calls` 和 `files_changed`。
  - 在导入 file events 的同时调用 `sync_context_graph_file_event()`。

这是从 operational memory 进入 durable memory 的关键同步函数。

### 7.4 工具调用与文件活动

- `insert_tool_log(&self, ...)`
  - 直接插单条 tool log。

- `query_tool_logs(&self, session_id, page, page_size)`
  - 分页读取 session tool log。

- `list_tool_logs_for_session(&self, session_id)`
  - 全量读取某 session tool log。

- `list_file_activity(&self, session_id, limit)`
  - 从 `tool_log.file_events_json` 或 `file_paths_json` 还原 file activity。
  - 输出 `FileActivityEntry`。

- `list_file_overlaps(&self, session_id, limit)`
  - 检测当前 session 和其他活跃 session 是否触碰了同一路径。
  - 这是并发冲突预警的 memory 视图。

### 7.5 Session 输出缓冲

- `append_output_line(&self, session_id, stream, line)`
  - 写 `session_output`。
  - 同时裁剪成固定窗口。
  - 顺手刷新 session `updated_at` 和 `last_heartbeat_at`。

- `get_output_lines(&self, session_id, limit)`
  - 取最近输出窗口。

这一层和 `ecc2/src/session/output.rs` 里的内存 ring buffer 形成双缓冲：

- 进程内存里有 `SessionOutputStore`。
- durable store 里有 `session_output`。

### 7.6 消息与 handoff memory

- `send_message(&self, from, to, content, msg_type)`
  - 插入 `messages`。
  - 调用 `sync_context_graph_message()`。

- `list_messages_sent_by_session(&self, session_id, limit)`
  - 查询某 session 发出的消息。

- `list_messages_for_session(&self, session_id, limit)`
  - 查询与某 session 相关的所有消息。

- `unread_message_counts(&self)`
  - 每个目标 session 的未读计数。

- `unread_approval_counts(&self)`
  - 只看 `query/conflict` 类型的未读计数。

- `unread_approval_queue(&self, limit)`
  - 审批/冲突消息队列。

- `latest_unread_approval_message(&self)`
  - 最近一条未读审批消息。

- `unread_task_handoffs_for_session(&self, session_id, limit)`
  - 查询未读 handoff，并按 priority 排序。

- `unread_task_handoff_count(&self, session_id)`
  - handoff 未读数。

- `unread_task_handoff_targets(&self, limit)`
  - 哪些 session 正在积压 handoff。

- `mark_messages_read(&self, session_id)`
  - 批量标记已读。

- `mark_message_read(&self, message_id)`
  - 单条标记已读。

- `latest_task_handoff_source(&self, session_id)`
  - 获取最近把任务交给该 session 的来源。

- `latest_task_handoff_activity(&self, session_id)`
  - 从最近 handoff 反推“收到/委派/spawned”状态文本。

这层 memory 支撑多 agent 协作，不只是聊天记录。

### 7.7 决策 memory

- `insert_decision(&self, session_id, decision, alternatives, reasoning)`
  - 写 `decision_log`。
  - 自动调用 `sync_context_graph_decision()`。

- `list_decisions_for_session(&self, session_id, limit)`
  - 查询单 session 的决策历史。

- `list_decisions(&self, limit)`
  - 查询全局最近决策。

决策 memory 在 ECC 里是第一类事实，不是普通注释。

## 8. 第六层：Context Graph 作为长期 Agent Memory

如果只看“长期可检索 memory”，Rust context graph 是最核心的一层。

### 8.1 自动同步函数

- `sync_context_graph_session(&self, session_id)`
  - 确保 session 在图里有实体表示。
  - metadata 包含 task、project、task_group、agent_type、state、working_dir、pid、worktree 等。

- `sync_context_graph_decision(&self, session_id, decision, alternatives, reasoning)`
  - 把 decision 变成 `decision` entity。
  - 建立 `session --decided--> decision` relation。

- `sync_context_graph_file_event(&self, session_id, tool_name, event)`
  - 把 file path 变成 `file` entity。
  - relation type 用 action，如 `modify/delete/read/move`。
  - metadata 保存 `last_action/last_tool/diff_preview`。

- `sync_context_graph_message(&self, from, to, content, msg_type)`
  - 把 session 间消息写成图关系。

- `sync_context_graph_history(&self, session_id, per_session_limit)`
  - 对历史 decision/file activity/message 做回填。
  - 这是从旧事实重建 graph 的批处理入口。

### 8.2 实体与关系 API

- `upsert_context_entity(&self, session_id, entity_type, name, path, summary, metadata)`
  - 用 `entity_key` 做 UPSERT。
  - 支持 summary 覆盖与 metadata 刷新。

- `list_context_entities(&self, session_id, entity_type, limit)`
  - 列出实体。

- `upsert_context_relation(&self, session_id, from_entity_id, to_entity_id, relation_type, summary)`
  - 关系是 UPSERT，不会无限重复插入同一条边。

- `list_context_relations(&self, entity_id, limit)`
  - 枚举某实体相关关系。

- `get_context_entity_detail(&self, entity_id, relation_limit)`
  - 返回实体详情、入边、出边。

### 8.3 Observation API

- `add_context_observation(&self, session_id, entity_id, observation_type, priority, pinned, summary, details)`
  - 给实体追加 observation。
  - 自动触发 observation compaction。

- `add_session_observation(&self, session_id, observation_type, priority, pinned, summary, details)`
  - 先确保 session entity 存在，再写 observation。

- `set_context_observation_pinned(&self, observation_id, pinned)`
  - 把某 observation 提升为 pinned。

- `list_context_observations(&self, entity_id, limit)`
  - 按 `pinned DESC, created_at DESC` 查询。

### 8.4 检索与压缩

- `recall_context_entities(&self, session_id, query, limit)`
  - 这是 ECC 当前最接近“agent memory recall”的函数。
  - 过程是：
    - 把 query 切 terms。
    - 拉一批候选 entity。
    - 汇总 relation 数、observation 文本、observation 数、priority、是否 pinned。
    - 调用 `context_graph_matched_terms()` 和 `context_graph_recall_score()` 排序。
  - 它本质上是基于结构、term match、priority、recency 的 symbolic recall，不是 embedding recall。

- `compact_context_graph(&self, session_id, keep_observations_per_entity)`
  - 外部入口。

- `compact_context_graph_observations(&self, session_id, entity_id, keep_observations_per_entity)`
  - 真正执行 compaction：
    - 删除重复 observation。
    - 对未 pinned observation 做 retention。
    - pinned observation 永不被普通 compaction 删除。

这一组函数说明 ECC 的长期 memory 现在更像一个可压缩 knowledge graph，而不是原始 transcript 档案馆。

### 8.5 Connector checkpoint

- `connector_source_is_unchanged(&self, connector_name, source_path, source_signature)`
  - 判断外部源是否变化。

- `upsert_connector_source_checkpoint(&self, connector_name, source_path, source_signature)`
  - 写 connector source checkpoint。

- `connector_checkpoint_summary(&self, connector_name)`
  - 给 connector 同步状态做汇总。

这部分把外部知识源的增量同步也纳入了 memory 管理边界。

## 9. ECC2 运行时：内存缓冲与后台写入

### 9.1 进程内输出缓冲

文件：`ecc2/src/session/output.rs`

关键类型和函数：

- `OutputStream`
  - `Stdout/Stderr` 枚举。

- `OutputLine`
  - 单行输出及其时间戳。

- `SessionOutputStore::new(capacity)`
  - 创建环形缓冲。

- `SessionOutputStore::subscribe()`
  - 提供广播订阅。

- `SessionOutputStore::push_line(session_id, stream, text)`
  - 写入 ring buffer，并广播事件。

- `SessionOutputStore::replace_lines(session_id, lines)`
  - 批量替换缓冲内容。

- `SessionOutputStore::lines(session_id)`
  - 读取某 session 当前 buffer。

这是进程内 working memory，与 SQLite `session_output` 相互补充。

### 9.2 异步 DB writer

文件：`ecc2/src/session/runtime.rs`

关键函数：

- `DbWriter::start(db_path, session_id)`
  - 启动后台线程写 DB。

- `DbWriter::update_state()`
- `DbWriter::update_pid()`
- `DbWriter::append_output_line()`
- `DbWriter::touch_heartbeat()`
  - 把会话运行中的关键变更异步写入 DB。

- `run_db_writer(db_path, session_id, rx)`
  - 真正消费 `DbMessage` 队列。

- `capture_command_output(db_path, session_id, command, output_store, heartbeat_interval)`
  - 运行子进程。
  - 拉 stdout/stderr。
  - 同时更新：
    - 进程内 `SessionOutputStore`
    - durable `session_output`
    - heartbeat
    - state/pid

- `capture_stream(session_id, reader, stream, output_store, db_writer)`
  - 把单个流逐行持久化。

运行时这部分说明 ECC2 不只是“最终落盘”，而是在执行过程中持续刷新 memory。

### 9.3 Daemon 作为 memory 维护者

文件：`ecc2/src/session/daemon.rs`

关键函数：

- `run(db, cfg)`
  - 周期性执行 heartbeat 检查、scheduled task dispatch、remote dispatch、backlog coordination、auto-merge、auto-prune。

- `resume_crashed_sessions(db)`
  - 启动时把 stale running session 标为 failed。

- `check_sessions(db, cfg)`
  - 心跳检查。

daemon 虽然不是传统意义上的 memory 模块，但它维护了 session memory 的一致性和新鲜度。

## 10. 代码级设计判断

### 10.1 优点

- 分层明确
  - hook continuity、observer learning、operational metrics、durable graph 分开管理。

- 会话恢复逻辑比常见“取最近文件”更稳
  - `selectMatchingSession()` 用 worktree 精确匹配，避免跨 worktree 污染。

- 重放防护做得早
  - historical summary 明确标记为 stale reference。

- operational memory 与 durable memory 解耦
  - `/tmp` bridge 负责快读，SQLite 负责长期事实。

- context graph 已具备结构化 recall 能力
  - 即使还没有 embedding，也已经不是纯字符串检索。

### 10.2 当前边界

- session markdown summary 与 Rust context graph 是两套 memory surface
  - 一个面向 Claude hook 注入。
  - 一个面向 ECC2 控制面检索。
  - 目前没有完全统一。

- observer instinct 与 context graph observation 是两套长期记忆机制
  - 前者偏 shell workflow。
  - 后者偏 SQLite graph。

- JS state-store 与 Rust StateStore 概念有重叠
  - 长期看可以继续收敛成一个 canonical durable schema。

- `recall_context_entities()` 仍是 symbolic recall
  - 没有 embedding/vector 召回层。

## 11. Function Index

这一节按模块列出最重要的 memory 相关函数，方便后续 code tour。

### `scripts/lib/utils.js`

- `getClaudeDir`
- `getSessionsDir`
- `getLegacySessionsDir`
- `getSessionSearchDirs`
- `getLearnedSkillsDir`
- `sanitizeSessionId`
- `getSessionIdShort`
- `findFiles`

### `scripts/hooks/session-start.js`

- `normalizePath`
- `dedupeRecentSessions`
- `getSessionRetentionDays`
- `isSessionStartContextDisabled`
- `getSessionStartMaxContextChars`
- `getSessionStartMode`
- `limitSessionStartContext`
- `pruneExpiredSessions`
- `selectMatchingSession`
- `parseInstinctFile`
- `readInstinctsFromDir`
- `extractInstinctAction`
- `summarizeActiveInstincts`
- `summarizeLearnedSkillFile`
- `collectLearnedSkillFiles`
- `summarizeLearnedSkills`
- `main`
- `writeSessionStartPayload`

### `scripts/hooks/session-end.js`

- `extractSessionSummary`
- `getSessionMetadata`
- `extractHeaderField`
- `buildSessionHeader`
- `mergeSessionHeader`
- `buildSummarySection`
- `buildSummaryBlock`
- `escapeRegExp`
- `main`

### `scripts/hooks/pre-compact.js`

- `main`

### `scripts/lib/session-manager.js`

- `parseSessionFilename`
- `getSessionCandidates`
- `getMatchingSessionCandidates`
- `parseSessionMetadata`
- `getSessionStats`
- `getAllSessions`
- `getSessionById`
- `writeSessionContent`
- `appendSessionContent`
- `deleteSession`

### `scripts/lib/session-aliases.js`

- `loadAliases`
- `saveAliases`
- `resolveAlias`
- `setAlias`
- `listAliases`
- `deleteAlias`
- `renameAlias`

### `scripts/lib/observer-sessions.js`

- `getHomunculusDir`
- `normalizeRemoteUrl`
- `resolveProjectRoot`
- `computeProjectId`
- `resolveProjectContext`
- `getSessionLeaseDir`
- `writeSessionLease`
- `removeSessionLease`
- `listSessionLeases`
- `stopObserverForContext`

### `scripts/hooks/observe-runner.js`

- `getPluginRoot`
- `resolveTarget`
- `findShellBinary`
- `getPhaseFromHookId`
- `run`
- `emitHookResult`

### `scripts/hooks/session-activity-tracker.js`

- `redactSecrets`
- `sanitizeInputParams`
- `extractFilePaths`
- `extractFileEvents`
- `enrichFileEventFromWorkingTree`
- `summarizeInput`
- `summarizeOutput`
- `buildActivityRow`
- `run`

### `scripts/hooks/ecc-metrics-bridge.js`

- `hashToolCall`
- `extractFilePaths`
- `readSessionCost`
- `run`

### `scripts/lib/session-bridge.js`

- `sanitizeSessionId`
- `getBridgePath`
- `readBridge`
- `writeBridgeAtomic`
- `resolveSessionId`

### `scripts/lib/state-store/index.js`

- `resolveStateStorePath`
- `wrapSqlJsDatabase`
- `openDatabase`
- `createStateStore`

### `scripts/lib/state-store/schema.js`

- `readSchema`
- `getEntityValidator`
- `validateEntity`
- `assertValidEntity`

### `scripts/lib/state-store/migrations.js`

- `ensureMigrationTable`
- `getAppliedMigrations`
- `applyMigrations`

### `scripts/lib/state-store/queries.js`

- `createQueryApi`
- `normalizeSessionInput`
- `normalizeSkillRunInput`
- `normalizeDecisionInput`
- `normalizeInstallStateInput`
- `normalizeGovernanceEventInput`
- `normalizeWorkItemInput`
- `summarizeSkillRuns`
- `summarizeInstallHealth`
- `summarizeWorkItems`
- `summarizeReadiness`

### `ecc2/src/session/output.rs`

- `SessionOutputStore::new`
- `SessionOutputStore::subscribe`
- `SessionOutputStore::push_line`
- `SessionOutputStore::replace_lines`
- `SessionOutputStore::lines`

### `ecc2/src/session/runtime.rs`

- `DbWriter::start`
- `DbWriter::update_state`
- `DbWriter::update_pid`
- `DbWriter::append_output_line`
- `DbWriter::touch_heartbeat`
- `run_db_writer`
- `capture_command_output`
- `capture_stream`

### `ecc2/src/session/store.rs`

- `StateStore::open`
- `StateStore::init_schema`
- `insert_session`
- `update_state_and_pid`
- `update_state`
- `update_pid`
- `update_metrics`
- `refresh_session_durations`
- `touch_heartbeat`
- `sync_cost_tracker_metrics`
- `sync_tool_activity_metrics`
- `insert_tool_log`
- `query_tool_logs`
- `list_tool_logs_for_session`
- `list_file_activity`
- `list_file_overlaps`
- `append_output_line`
- `get_output_lines`
- `send_message`
- `list_messages_for_session`
- `unread_message_counts`
- `unread_task_handoffs_for_session`
- `mark_messages_read`
- `insert_decision`
- `list_decisions_for_session`
- `sync_context_graph_history`
- `sync_context_graph_session`
- `sync_context_graph_decision`
- `sync_context_graph_file_event`
- `sync_context_graph_message`
- `upsert_context_entity`
- `list_context_entities`
- `recall_context_entities`
- `get_context_entity_detail`
- `add_context_observation`
- `add_session_observation`
- `set_context_observation_pinned`
- `list_context_observations`
- `compact_context_graph`
- `compact_context_graph_observations`
- `upsert_context_relation`
- `list_context_relations`
- `connector_source_is_unchanged`
- `upsert_connector_source_checkpoint`
- `connector_checkpoint_summary`

## 12. 结论

ECC 的 agent memory 管理不是一个点功能，而是一套分层系统：

- Session tmp 文件负责短到中期的会话连续性。
- Observer homunculus 负责 project-scoped 的行为学习。
- Metrics JSONL 和 tmp bridge 负责运行态工作记忆。
- Rust `StateStore` 和 context graph 负责 durable、可检索、可压缩的长期事实记忆。

从代码成熟度看，最值得继续演进的方向不是“再加一个记忆入口”，而是收敛：

1. 把 session markdown surface 和 context graph surface 建立更清晰的双向映射。
2. 把 observer instinct 和 context graph observation 的角色分工进一步统一。
3. 决定 JS state-store 和 Rust `StateStore` 谁是长期 canonical schema。
4. 如果后续要做语义 recall，再在 `recall_context_entities()` 之上叠加 embedding/vector 层，而不是重写现有 symbolic recall。
