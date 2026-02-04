---
name: obsidian-tasks
description: 使用Tasks插件在Obsidian中管理和查询任务，支持任务状态跟踪、截止日期、计划日期、创建日期、重复规则、任务依赖等功能。提供强大的查询语法（Tasks Query Language）创建动态任务列表，支持过滤、排序、分组。适用于个人任务管理、项目跟踪、GTD实践、每日计划、习惯追踪等场景。当用户提到Tasks插件、任务管理、TODO列表、任务查询语法、重复任务、任务状态等概念时使用。
---
重要前提
Tasks插件是主动式任务管理系统：与Dataview不同，Tasks插件会直接修改源文件内容（例如完成任务、更新重复任务）。所有修改都遵循严格的规则，确保数据一致性。查询结果支持交互（如勾选框），勾选后会自动更新源文件。

## ⚠️ 数据安全关键警告

### 重要安全提示
- **直接文件修改**：Tasks插件直接修改源文件内容，包括自动完成、重复任务生成等操作
- **备份建议**：
  - 启用Obsidian的自动备份功能（设置 → 核心插件 → 备份）
  - 配置Git版本控制，定期提交更改
  - 启用重复规则和自动完成前，确保有可靠的备份机制
  - **重要**：定期备份 `.obsidian/plugins/obsidian-tasks/data.json` 文件，它包含你的自定义状态配置
- **数据恢复程序**：
  1. 关闭Obsidian
  2. 删除`.obsidian/plugins/obsidian-tasks/cache`目录（安全操作）
  3. 重启应用并等待索引完成（大型库可能需要5-10分钟）

### 🚨 最后的手段：重置插件配置
**警告**：此操作会导致所有自定义状态（Custom Statuses）配置丢失

1. 关闭Obsidian
2. **必须先备份** `.obsidian/plugins/obsidian-tasks/data.json` 文件
3. 删除库目录下`.obsidian/plugins/obsidian-tasks/data.json`
4. 重启应用并等待索引完成

**后果**：data.json存储了所有自定义状态的定义。如果删除了它且没有备份，所有的 [/]、[-] 等符号将失去其背后的"逻辑属性"，导致查询失效。
- **多用户环境警告**：在Git同步或团队共享库中，避免在不同设备上同时编辑同一任务。Tasks插件的自动修改可能导致合并冲突

### ⚠️ 全局过滤器陷阱
- **功能说明**：Tasks插件有一个核心设置——Global Filter（例如 #task）
- **行为**：如果启用了Global Filter，所有任务必须包含该标签
- **严重后果**：
  - 未包含标签的任务在查询中不会显示
  - `ignore global query` 无法跳过此限制（因为插件根本没索引这些行）
  - 这是索引层面的限制，而非查询层面的过滤

**全局过滤器副作用**：
- **自动标签**：开启全局过滤器后，任务模态框（Create or edit task）生成的任务会自动带上该标签
- **瞬间蒸发**：如果你手动删除标签，该任务将从所有Tasks查询结果中瞬间蒸发，且不报任何错误
- **迁移警告**：启用全局过滤器后，需要为所有现有任务添加标签，这是一个需要谨慎操作的迁移过程

### ⚠️ 数据安全附加警告

#### 同步冲突
- **风险**：如果你使用 Obsidian Sync 或 iCloud，Tasks 插件的频繁文件修改极易在同步过程中产生 `.sync-conflict` 副本
- **建议**：
  - 关闭“任务状态自动转换”功能，或
  - 确保单设备编辑，避免多设备同时操作
  - 定期检查并清理同步冲突文件

#### 版本控制冲突
- **Git注意事项**：在使用Git版本控制时，Tasks的自动修改可能导致频繁的合并冲突
- **最佳实践**：使用 `git pull --rebase` 减少冲突，定期提交更改

# Tasks插件使用指南

## 一、基础语法

### 基本任务结构

```markdown
- [ ] 基本任务
- [x] 已完成任务
- [/] 部分完成（进行中）
- [-] 已取消
```

### 任务元数据字段
任务通过内联字段（行尾元数据）存储丰富信息，严格遵循格式规范：

```markdown
- [ ] 任务文本 [元数据:: 值]
```

### 核心元数据字段

| 字段 | 格式 | 描述 | 示例 |
|------|------|------|------|
| 日期（Due） | 📅 YYYY-MM-DD 或 due:: YYYY-MM-DD | 任务截止日期 | 📅 2024-12-31 |
| 计划日期（Scheduled） | ⏳ YYYY-MM-DD 或 scheduled:: YYYY-MM-DD | 任务应开始处理的日期 | ⏳ 2024-12-25 |
| 创建日期（Created） | ➕ YYYY-MM-DD 或 created:: YYYY-MM-DD | 任务创建日期 | ➕ 2024-12-20 |
| 开始日期（Start） | 🛫 YYYY-MM-DD 或 start:: YYYY-MM-DD | 任务可开始的日期 | 🛫 2024-12-25 |
| 优先级 | 🔺 highest, ⏫ high, 🔼 medium, 🔽 low, ⏬ lowest 或 priority:: (highest|high|medium|low|lowest|none) | 任务优先级 | 🔼 medium |
| 重复规则 | 🔁 (every day|every week|every month|every year|every [n] [days|weeks|months|years]) [when done: (start|due|both)] | 任务重复规则 | 🔁 every day |
| 依赖任务 | ⛔ task123 或 dependsOn:: task123 | 任务依赖 | ⛔ task123 |
| 任务ID | 🆔 task123 | 任务标识符，用于依赖引用 | 🆔 task1 |
| 完成日期 | ✅ YYYY-MM-DD | 任务完成时间 | ✅ 2024-12-31 |

### 优先级标识（官方标准）

| 优先级 | Emoji标识 | 查询值 | 数值映射 |
|--------|-----------|--------|----------|
| Highest | 🔺 | highest | 1 |
| High | ⏫ | high | 2 |
| Medium | 🔼 | medium | 3 |
| Low | 🔽 | low | 4 |
| Lowest | ⏬ | lowest | 5 |
| None | (无符号) | none | (无数值) |

**重要**：
- 优先级Emoji只有箭头符号，没有🔴🟡🟢颜色圆点！
- **解析逻辑**：Tasks插件的解析器是从行尾向前解析元数据的
  - 优先级可以放在日期之前或之后
  - **绝对不能**放在描述文本中间
  - 必须与其他内容用空格分隔
- **Unicode字符差异警告**：这些是Unicode字符，在不同操作系统（iOS vs Windows）和字体下外观会有细微差异，手动输入极易出错。
- **强烈推荐**：通过模态框设置优先级：使用 `Tasks: Create or edit` 命令打开模态框，避免手动输入Emoji的错误。

### 优先级查询语法扩展

#### 比较查询
- **语法**：
  - `priority is above medium`（高于中等优先级）
  - `priority is below high`（低于高优先级）
- **优先级顺序**：Highest > High > Medium > Low > Lowest
- **示例**：
  - `priority is above medium` 匹配 Highest 和 High
  - `priority is below high` 匹配 Medium、Low 和 Lowest

#### 数值映射深度说明
- **内部数值**：
  - Highest (1) < High (2) < Medium (3) < Low (4) < Lowest (5)
- **比较逻辑**：
  - `priority is above medium` 实际上是查询数值 < 3 的任务
  - `priority is below high` 实际上是查询数值 > 2 的任务
- **性能优化**：数值比较在大型库中比字符串比较更高效

#### 无优先级查询
- **语法**：`priority is none`
- **功能**：匹配没有设置优先级的任务
- **使用场景**：用于查找未设置优先级的任务，以便进行优先级分类

### 日期类型映射（官方标准）

| 日期类型 | Emoji标识 | 内联字段 | 描述 | 自动行为 |
|---------|-----------|----------|------|----------|
| Due（截止） | 📅 | due:: | 任务截止日期 | 手动添加 |
| Scheduled（计划） | ⏳ | scheduled:: | 任务计划开始日期 | 手动添加 |
| Start（开始） | 🛫 | start:: | 任务可开始日期 | 手动添加 |
| Created（创建） | ➕ | created:: | 任务创建日期 | 手动添加 |
| Completion（完成） | ✅ | completion:: | 任务完成日期 | 完成任务时自动添加 |
| Cancelled（取消） | ❌ | cancelled:: | 任务取消日期 | 取消任务时自动添加（需在设置中开启 Record date of cancellation） |

### completion与done的区别

**重要区别**：
- **completion（完成日期）**：Tasks插件自动添加的完成日期，使用✅符号，格式为`✅ 2024-12-31`，表示任务实际完成的时间
  - **自动添加**：完成任务时会自动添加，除非在设置中禁用并重启Obsidian
- **done（完成状态）**：表示任务的完成状态，在Tasks查询中使用`done`或`not done`过滤，在Dataview中对应`task.completed`布尔值字段

**使用场景**：
- 使用`completion`字段：当你需要查询任务的具体完成时间，如`completed after last week`
- 使用`done`状态：当你只需要过滤任务的完成状态，不关心具体完成时间

**Dataview中的对应关系**：
- `task.completed`（布尔值）：对应Tasks的`done`状态
- `task.text`中可能包含`✅ 2024-12-31`：但需要从文本中提取，Dataview没有直接的`task.completion`字段

### 日期格式与时区注意事项

**日期格式**：
- 严格使用YYYY-MM-DD格式（无时间，无[[wiki links]]）
- 无效日期在查询中会被静默忽略

**时区处理**：
- 所有日期基于本地时区，无UTC支持
- 跨设备时区不匹配可能导致日期偏移
- 建议所有协作设备使用相同的系统时区设置

### 任务ID与块引用

#### 任务ID（官方标准）
- **格式**：使用🆔符号，如`- [ ] 任务描述 🆔 task1`
- **内联字段**：`id:: task1`
- **用途**：用于依赖引用和任务标识
- **依赖引用**：在依赖中使用任务ID值，如`dependsOn:: task1`或`⛔ task1`

#### 块引用（Obsidian标准功能）
- **格式**：`^block-id`（Obsidian标准块引用格式）
- **特点**：由字母、数字、连字符、下划线组成
- **用途**：用于在Obsidian中引用特定段落或任务
- **注意**：这是Obsidian的标准功能，与Tasks插件的任务ID是不同的概念

### 重要区别
- **任务ID**：Tasks插件特有，用于依赖关系，格式为`🆔 task123`
- **块ID**：Obsidian标准功能，用于块引用，格式为`^block-id`
- **依赖引用**：使用任务ID值（如`task123`），不是块引用（如`^block-id`）

### 任务依赖详细机制
- **依赖语法**：`dependsOn:: task123` 或使用 ⛔ 符号（如 `⛔ task123`）
- **依赖目标**：依赖的是任务ID，不是文件块引用（^block-id）
- **示例**：
  - `dependsOn:: abc123`（内联字段格式）
  - `⛔ abc123`（Emoji格式）
- **依赖状态**：
  - ⛔ 依赖任务未完成时，被阻塞任务显示为'blocked'状态
  - ✅ 依赖任务完成后，被阻塞任务自动变为可用状态，并在查询中显示为'unblocked'
- **依赖传递**：支持多级依赖链（任务A依赖任务B，任务B依赖任务C）
- **循环检测**：系统会检测并警告依赖循环，避免死锁
  - **显示**：在任务旁边显示⚠️图标，悬停显示"循环依赖"
  - **处理**：自动将状态设为"cannot-complete"，阻止完成操作
  - **解决**：需要手动打破依赖链，移除或修改导致循环的依赖关系
- **重要注意**：依赖字段会从下一个重复任务中移除

### 依赖查询语法

**官方查询语法**：
- `depends on <id>` - 查询依赖特定ID的任务
- `is blocked` - 查询被阻塞的任务
- `is blocking` - 查询正在阻塞其他任务的任务

**示例**：
```tasks
is blocked
not done
sort by due
```
### 严格格式要求
日期格式：必须使用ISO格式（YYYY-MM-DD），其他格式将被忽略
Emoji日期标识：系统仅识别预定义Emoji（📅⏳➕🛫），自定义Emoji无效
优先级标识：使用箭头Emoji（�⏫��⏬️），不使用颜色圆点

### YAML frontmatter优先级
**重要**：YAML frontmatter中的字段优先级高于行内元数据。

**示例**：
```yaml
---
due: 2024-12-31
---
- [ ] 任务文本 📅 2025-01-15
```

**实际效果**：任务的due日期为2024-12-31（YAML值覆盖行内值）

### ⚠️ YAML Frontmatter 的“只读性”陷阱

**关键限制**：Tasks 插件无法自动修改 YAML 里的数据

**严重后果**：
- 如果你在 YAML 定义了 due，当你在 Tasks 界面勾选完成一个重复任务时，插件会因为无法修改 YAML 而导致日期更新失败或生成错误任务
- 这会破坏重复任务的正常工作流

**严格禁忌**：不要在需要“交互”（如勾选、更新日期）的任务上使用 YAML 元数据

**推荐做法**：对于需要与 Tasks 插件交互的任务，使用行内元数据（如 `📅 2024-12-31`）而非 YAML

### 空格要求
**严格规范**：元数据必须与任务文本或标签之间存在至少一个空格的分隔边界，推荐使用两个空格以确保解析稳定性

**关键提醒**：在带有标签的任务中，如果元数据紧跟在标签后面（如 #tag📅2023），解析器 100% 会失败

```markdown
# 推荐示例
- [ ] 任务文本  📅 2024-12-31
- [ ] 任务文本 #tag  📅 2024-12-31

# 也可工作但不推荐
- [ ] 任务文本 📅 2024-12-31

# 错误示例
- [ ] 任务文本📅 2024-12-31
- [ ] 任务文本 #tag📅2024-12-31
```

## 重复任务规则引擎

### 重复语法（官方标准）

**重要规则**：所有重复规则必须以 `every` 开头，这是官方要求的必需语法。

```markdown
- [ ] 每日任务 🔁 every day
- [ ] 每周任务（完成时更新起始日期） 🔁 every week when done: start
- [ ] 每月15号 🔁 every month on the 15th
- [ ] 每季度首日 🔁 every 3 months on the 1st
- [ ] 每年生日 🔁 every year on December 31
```

### 完整重复规则语法表（官方支持的所有模式）

| 规则类型 | 语法示例 | 描述 |
|---------|----------|------|
| 每日重复 | 🔁 every day | 每天重复 |
| 每周重复 | 🔁 every week | 每周重复 |
| 每月重复 | 🔁 every month | 每月重复 |
| 每年重复 | 🔁 every year | 每年重复 |
| 工作日重复 | 🔁 every weekday | 每个工作日重复（周一至周五） |
| 间隔重复 | 🔁 every 3 days | 每3天重复 |
| 间隔重复 | 🔁 every 2 weeks | 每2周重复 |
| 间隔重复 | 🔁 every 6 months | 每6个月重复 |
| 周特定日 | 🔁 every week on Sunday | 每周日重复 |
| 周特定日 | 🔁 every week on Tuesday, Friday | 每周二和周五重复 |
| 间隔周特定日 | 🔁 every 2 weeks on Sunday | 每2周的周日重复 |
| 月特定日 | 🔁 every month on the 15th | 每月15日重复 |
| 月最后日 | 🔁 every month on the last | 每月最后一天重复 |
| 月最后特定日 | 🔁 every month on the last Friday | 每月最后一个周五重复 |
| 月倒数特定日 | 🔁 every month on the 2nd last Friday | 每月倒数第二个周五重复 |
| 间隔月特定日 | 🔁 every 3 months on the 1st | 每3个月的1日重复 |
| 特定月份特定日 | 🔁 every January on the 15th | 每年1月15日重复 |
| 复杂组合 | 🔁 every April and December on the 1st and 24th | 每年4月和12月的1日和24日重复 |
| 完成时更新 | 🔁 every day when done: start | 完成时更新start日期 |
| 完成时更新 | 🔁 every week when done: due | 完成时更新due日期 |
| 完成时更新 | 🔁 every month when done: both | 完成时同时更新start和due日期 |

### 重复任务生命周期
1. **创建**：初始任务创建
2. **完成**：用户勾选任务
   - **重要触发点**：必须在Tasks查询结果界面或通过插件命令触发完成操作
   - **注意**：如果在纯编辑器模式下手动将 [ ] 改为 [x]，插件可能不会触发新任务的生成（取决于索引状态和设置）
3. **生成新任务**：原任务被标记为完成，系统创建一个全新任务
4. **更新**：根据规则更新日期字段
   - `when done: start`：只更新start日期
   - `when done: due`：只更新due日期
   - `when done: both`：同时更新start和due日期
   - **默认行为**：如果未指定 `when done`，则使用 "Relative to the original date" 逻辑
     - **深度细节**：基于原定日期计算，而非当前日期
     - **重要警告**：如果原始任务过期了5天，新生成的任务也会基于原定日期，而非今天。这对习惯追踪至关重要
     - **关键补充**：如果任务没有截止日期（Due Date），重复规则将默认基于"完成日期"（Completion Date）计算

**锚点日期 (Anchor Date) 逻辑**：
Tasks 的默认逻辑是根据存在的日期字段优先级来确定的：

1. 如果有 due (截止)，基于原 due 偏移
2. 如果没有 due 但有 scheduled (计划)，基于原 scheduled 偏移
3. 如果没有 due 和 scheduled 但有 start (开始)，基于原 start 偏移
4. 如果没有任何日期字段，强制要求 when done 或默认失败

**智能修正特性**：
- 使用 `every weekday` 时，即使原始任务在周六，新任务也会自动修正到周一
- 系统会智能避开周末，确保任务落在工作日内
5. **保留**：原任务状态变更为[x]，添加completion日期

### 重要警告和限制
- **必需条件**：重复任务必须至少有一个日期字段（due/scheduled/start）才能正常工作
- **无效日期**：无效日期（如2月30日）会导致重复规则消失，不会生成新任务
- **依赖处理**：依赖字段（dependsOn）会从下一个重复任务中移除
- **每日笔记注意**：不推荐在每日笔记中使用重复任务，可能会创建重复条目
- **功能限制**：不支持 "for x times" 或 "until date" 语法
- **块ID保留**：重复任务生成时，插件会根据设置决定是否保留块ID（^block-id）
  - **Tasks 5.x 默认行为**：保留块ID以维持引用完整性
  - **用途**：确保与其他笔记的链接和引用保持有效
- **无限循环防御**：当任务日期超过100年时，自动停止重复
- **历史记录**：原任务不会被删除，而是标记为完成，保留完整历史
- **时区处理**：所有日期基于本地时区，无UTC转换

### 时区边界案例
- **跨时区协作**：所有设备应设置相同系统时区，避免日期计算差异
- **重复任务**：在时区变更日（如夏令时切换）可能跳过或重复
- **移动设备**：自动时区切换可能导致日期偏移
- **查询行为**：`today` 基于本地时区的当前日期，不同时区的设备看到的结果可能不同

## 二、标准查询

### 查询基础结构

```tasks
not done
due after 2024-12-20
sort by due
limit 10
```

必须在代码块中标记为tasks，每行一个查询指令。

### 基本查询命令

#### 过滤命令

| 命令 | 描述 | 示例 |
|------|------|------|
| done/not done | 过滤完成/未完成任务 | not done |
| due [before|after] YYYY-MM-DD | 过滤截止日期 | due after 2024-12-25 |
| scheduled [before|after] YYYY-MM-DD | 过滤计划日期 | scheduled before today |
| start [before|after] YYYY-MM-DD | 过滤开始日期 | start after tomorrow |
| created [before|after] YYYY-MM-DD | 过滤创建日期 | created after 2024-01-01 |
| happens [before|after] YYYY-MM-DD | 复合日期查询（包含due、scheduled、start三者之一） | happens before today |
| priority is (highest|high|medium|low|lowest) | 按优先级过滤 | priority is high |
| priority is not (highest|high|medium|low|lowest) | 按优先级过滤（否定） | priority is not high |
| priority equals (highest|high|medium|low|lowest) | 按优先级过滤（可选写法） | priority equals high |
| path [includes|does not include] "text" | 按文件路径过滤 | path includes "Projects" |
| root | 只查询根目录下的任务 | root |
| folder/path | 查询特定文件夹 | "Tasks/Work" |
| description [includes|does not include] "text" | 按任务描述过滤 | description includes "meeting" |
| has [due|scheduled|start|created|recurrence|priority] | 检查字段是否存在 | has due |
| no [due|scheduled|start|created|recurrence|priority] | 检查字段不存在 | no due |
| is recurring/is not recurring | 过滤重复/非重复任务 | is recurring |

### 路径查询指令

#### root 指令
- **功能**：只查询根目录下的任务，不包括子文件夹中的任务
- **使用场景**：当你只关心根目录中的任务，不需要查看子文件夹内容时
- **示例**：

```tasks
root
not done
sort by due
```

#### folder 指令
- **功能**：查询特定文件夹中的任务
- **语法**：使用文件夹路径，如`"Tasks/Work"`
- **使用场景**：当你需要关注特定文件夹中的任务，如工作任务、个人任务等
- **示例**：

```tasks
"Tasks/Work"
not done
sort by due
```

### 日期关键词
- **基础日期**：today（今天）、tomorrow（明天）、yesterday（昨天）
- **相对日期**：
  - in N days（N天后）、N days ago（N天前）
  - in 2 weeks（2周后）、3 weeks ago（3周前）
  - in 1 month（1个月后）、2 months ago（2个月前）
- **周相关**：this week（本周）、next week（下周）、last week（上周）
- **月相关**：this month（本月）、next month（下月）、last month（上月）
- **季度相关**：this quarter（本季度）、next quarter（下季度）、last quarter（上季度）
- **年相关**：this year（今年）、next year（明年）、last year（去年）

## 三、高级功能

### Status Type查询系统 (Tasks 4.0+)

Status Type与自定义状态系统配合，提供五类状态：

| 状态类型 | 匹配`not done` | 匹配`done` | 典型符号 |
|----------|----------------|------------|----------|
| TODO | ✅ | ❌ | [ ] |
| IN_PROGRESS | ✅ | ❌ | [/] |
| DONE | ❌ | ✅ | [x] |
| CANCELLED | ❌ | ✅ | [-] |
| NON_TASK | ❌ | ✅ | [~] |

**查询语法**：

```tasks
status.type is IN_PROGRESS
status.type is not CANCELLED
```

**重要区别**：
- `status.type is TODO` 只匹配未开始的任务
- `not done` 匹配TODO和IN_PROGRESS两种状态
- `status.type is DONE` 只匹配已完成的任务
- `done` 匹配DONE、CANCELLED和NON_TASK三种状态

### 自定义状态说明

**配置要求**：
- 自定义状态需要在Obsidian设置 → 插件 → Tasks → 状态中配置
- 可能需要自定义CSS来美化显示效果

**默认行为**：
- 未知的状态符号会默认为TODO类型
- 查询中的状态类型匹配不区分大小写（Tasks 5.0+）

### ⚠️ 状态转换的“强制符号”陷阱

**nextStatus 配置风险**：
- 如果 nextStatus 设置为空格 " "，会导致点击后状态循环，无法正常工作
- 建议为每个状态设置明确的下一个状态

**CSS 依赖警告**：
- 在使用自定义状态（如 [/] IN_PROGRESS）时，如果对应的 CSS 未加载，Obsidian 原生渲染器会将 [/] 渲染为一个带斜杠的复选框，但无法点击交互
- **必须提醒**：用户需要安装支持自定义状态的 CSS 主题（如 Minimal）或添加自定义 CSS
- **兼容性**：移动端对自定义状态的支持仍然较差，图标可能显示异常

### 依赖关系管理

#### 任务依赖语法
- **依赖语法**：`dependsOn:: task123` 或使用 ⛔ 符号（如 `⛔ task123`）
- **依赖目标**：依赖的是任务ID，不是文件块引用
- **示例**：
  - `dependsOn:: abc123`（内联字段格式）
  - `⛔ abc123`（Emoji格式）

#### 依赖状态
- **被阻塞**：依赖任务未完成时，被阻塞任务显示为'blocked'状态
- **可用**：依赖任务完成后，被阻塞任务自动变为可用状态，并在查询中显示为'unblocked'

#### 依赖查询
- **查询被阻塞任务**：`is blocked`
- **查询未被阻塞任务**：`is not blocked`
- **查询有依赖的任务**：`depends on task123`

#### 循环依赖检测
- **显示**：在任务旁边显示⚠️图标，悬停显示"循环依赖"
- **处理**：自动将状态设为"cannot-complete"，阻止完成操作
- **解决**：需要手动打破依赖链，移除或修改导致循环的依赖关系

### 自定义排序与分组

#### 自定义排序
- **基本排序**：`sort by due`、`sort by priority`等
- **自定义排序函数**：

```tasks
not done
sort by function(task) {
  return task.description.length;
}
```

#### 嵌套分组
- **多级分组**：支持最多3级嵌套分组
- **示例**：

```tasks
not done
happens in the next 7 days
group by folder
group by priority
sort by due
```

#### 分组函数（Group by Function）

**功能**：使用JavaScript函数进行自定义分组，实现复杂的分组逻辑

**语法**：`group by function <expression>`

**示例**：

```tasks
# 按任务描述长度分组
group by function task.description.length

# 按文件名后缀分组
group by function task.file.filename.split('.').pop()

# 按文件名大写形式分组
group by function task.file.filename.toUpperCase()

# 按文件所在文件夹分组
group by function task.file.folder.split('/').pop()
```

### 嵌套任务显示 (show tree)

**功能**：以树形结构显示嵌套任务，展示任务的层级关系

**语法**：`show tree`

**使用场景**：当你需要查看任务的层级结构，特别是在处理具有子任务的复杂项目时

**示例**：

```tasks
not done
path includes "Projects"
show tree
sort by due
```

### 高级查询技巧

#### 复杂条件组合

```tasks
not done
(due before tomorrow) OR (scheduled before tomorrow)
priority is high
path includes "Work"
sort by due
limit 5
```

#### 过滤函数（Filter by Function）

**功能**：使用JavaScript逻辑进行极度复杂的过滤，比普通过滤指令更强大

**语法**：`filter by function <expression>`

**示例**：
- `filter by function task.description.length > 20`
- `filter by function task.due && task.due.moment.isBefore(moment(), 'day')`

**示例**：

```tasks
not done
# 只显示描述中包含数字的任务
filter by function task.description.match(/\d+/) !== null

# 只显示描述长度超过20个字符的任务
filter by function task.description.length > 20

# 只显示截止日期在周末的任务 (Moment.js 语法)
filter by function task.due && [0, 6].includes(task.due.moment.day())
```

### 脚本过滤高级指南

#### 日期处理必须使用 Moment.js
- **核心语法**：`task.due.moment` 返回 Moment 对象
- **常用方法**：
  - `task.due.moment.day()` - 获取星期几（0-6，周日为0）
  - `task.due.moment.isBefore(moment(), 'day')` - 检查是否过期
  - `task.due.moment.valueOf()` - 获取时间戳（用于排序）

**示例**：

```tasks
# 查询过期的任务
filter by function task.due && task.due.moment.isBefore(moment(), 'day')

# 查询本周内到期的任务
filter by function task.due && task.due.moment.isBetween(
  moment().startOf('week'), 
  moment().endOf('week')
)
```

#### 重要注意事项
- **task.description**：不包含元数据Emoji，仅包含纯文本描述
- **空值处理**：必须处理 `task.due`、`task.priority` 等可能为 null 的情况
- **性能考虑**：复杂的过滤函数会影响查询速度，建议在大型库中谨慎使用

#### 分组与隐藏选项说明

**group by recurrence**：按重复规则分组，部分版本可能存在渲染问题

**hide recurrence rule**：在查询结果中隐藏重复规则，Tasks 5.x 后此选项仍然有效

**limit groups**：限制每个分组显示的任务数量，避免分组过大

**示例**：

```tasks
not done
group by folder
group by priority
limit groups 3  # 每个分组最多显示3个任务
sort by due
```

#### 移动端注意事项
- 移动端对自定义状态的支持仍然较差，图标可能显示异常
- 建议在移动端使用标准状态符号以获得最佳体验

### 边缘案例警告

**多行任务**：
- Tasks插件对多行列表项的支持有限
- 元数据必须在第一行末尾，后续行的内容不会被解析为元数据

**列表缩进**：
- 任务必须是标准的Markdown列表（使用 -, *, +）
- 缩进必须严格遵循Obsidian的层级结构
- 错误的缩进可能导致查询漏掉子任务

**示例**：
```markdown
# 正确格式
- [ ] 任务描述  📅 2024-12-31
  - [ ] 子任务描述

# 错误格式（元数据不在第一行）
- [ ] 任务描述
  额外说明  📅 2024-12-31
```

#### 查询布尔运算规则

**重要规则**：
1. **默认关系**：所有查询条件默认使用AND关系，即所有条件都必须满足
2. **OR操作**：使用大写的OR进行或运算，如`(due before tomorrow) OR (scheduled before tomorrow)`
3. **括号分组**：复杂条件需要使用括号分组，确保运算优先级正确
4. **否定操作**：不支持NOT前缀，必须使用is not形式，如`priority is not high`
5. **无显式AND**：不支持显式书写AND（默认就是AND关系）

**查询执行顺序**：
1. 全局过滤 (Settings) - 基于Global Filter标签
2. 指令过滤 (Query Block) - 基础过滤条件
3. 布尔逻辑 (OR 运算) - 复杂条件组合
4. 排序 (sort by) - 结果排序
5. 分组 (group by) - 结果分组
6. 限制 (limit) - 结果数量限制

### 调试与优化

#### explain 指令
- **功能**：显示查询的解析过程，是排查查询问题的重要工具
- **使用场景**：当查询结果不符合预期时，添加此指令查看查询是如何被解析的
- **示例**：

```tasks
not done
due after today
explain
```

### ignore global query 指令
- **功能**：忽略插件设置中配置的全局查询条件
- **使用场景**：在复杂查询中，当你需要临时绕过全局查询设置，使用完全自定义的条件时
- **示例**：
  - 全局设置了 `path includes "Tasks/"`，但你需要查询其他路径的任务
  - 全局设置了 `not done`，但你需要查询已完成的任务

### 查询默认行为

**默认排序**：
- 未指定排序时的默认顺序：status > due > path
- 优先级：状态 > 截止日期 > 文件路径

**全局查询行为**：
- 全局查询条件会自动附加到所有查询中，除非使用 `ignore global query` 指令
- 建议只在全局设置中配置通用过滤条件（如排除归档文件夹）

### 显示控制指令

**展示优化**：
- **short mode**：隐藏任务行尾的所有元数据符号（日期、重复、优先级等），仅显示文本，保持列表整洁
- **hide task count**：隐藏查询结果顶部的任务数量统计
- **hide backlink**：隐藏指向源文件的反向链接
- **hide edit button**：隐藏任务末尾的编辑图标（铅笔）
- **hide priority**：隐藏优先级符号
- **hide created date**：隐藏创建日期
- **hide start date**：隐藏开始日期
- **hide scheduled date**：隐藏计划日期
- **hide due date**：隐藏截止日期
- **hide recurrence rule**：隐藏重复规则

**示例**：

```tasks
not done
due in the next 7 days
short mode
hide task count
hide edit button
hide priority
sort by due
```

### 占位符 (Placeholders)（Tasks 5.0+ 支持）

**功能**：在查询中使用动态占位符，实现基于当前上下文的智能查询

**常用占位符**：
- **{{query.file.path}}**：当前文件的路径
- **{{query.file.title}}**：当前文件的标题
- **{{query.file.folder}}**：当前文件所在的文件夹

**示例**：

```tasks
# 仅查询当前文件内的任务
path includes {{query.file.path}}
not done
sort by due
```

```tasks
# 查询描述中包含当前文件名的任务
description includes {{query.file.title}}
not done
sort by due
```

## 高级查询场景

### 复杂条件组合

```tasks
not done
(due before tomorrow) OR (scheduled before tomorrow)
priority equals high
path includes "Work"
sort by due
limit 5
```

### 嵌套分组

```tasks
not done
happens in the next 7 days
group by folder
group by priority
```

### 动态日期计算

```tasks
not done
due after today
due before in 14 days
sort by due ascending
```

## 任务操作与交互

### 交互行为
- **勾选框**：点击任务前的勾选框将：
  1. 更新任务状态（[ ] → [x]）
  2. 自动添加completion日期（✅ YYYY-MM-DD）
  3. 如有重复规则，生成新任务
  4. 如有依赖任务，更新依赖状态
- **编辑**：点击任务文本进入源文件编辑
- **上下文菜单**：右键任务可：
  - 复制任务
  - 删除任务
  - 编辑原始内容
  - 转到源文件

### 任务状态转换规则
| 初始状态 | 操作 | 结果状态 | 附加行为 |
|----------|------|----------|----------|
| [ ] 未开始 | 勾选 | [x] 完成 | 添加completion日期，处理重复 |
| [/] 进行中 | 勾选 | [x] 完成 | 同上 |
| [-] 已取消 | 勾选 | [x] 完成 | 保留取消状态历史 |
| [x] 已完成 | 取消勾选 | [ ] 未开始 | 移除completion日期 |

## 与Dataview集成

### Tasks作为Dataview数据源

#### 方式1：使用TASK查询（推荐）
```dataviewjs
const result = await dv.query(`
TASK FROM #tasks
WHERE !completed
SORT due ASC
`);
dv.taskList(result.value);
```

#### 推荐的DataviewJS访问方式
```dataviewjs
// 推荐的DataviewJS访问方式
const tasks = dv.pages().file.tasks.filter(t => t.text.includes("📅") && !t.completed);
dv.taskList(tasks);

// 示例：只显示描述中包含数字的任务
const tasksWithNumbers = dv.pages().file.tasks.filter(t => 
  t.text.match(/\d+/) !== null && !t.completed
);
dv.taskList(tasksWithNumbers);
```

#### 方式2：遍历file.tasks（适用于自定义逻辑）
```dataviewjs
// 遍历所有包含tasks标签的文件中的任务
for (let page of dv.pages('#tasks')) {
  for (let task of page.file.tasks) {
    // 访问任务属性
    // task.completed - 任务是否完成
    // task.due - 截止日期
    // task.text - 任务文本（包含标签）
    // task.section - 任务所在章节
    // task.position - 任务在文件中的位置
  }
}
```

### 混合查询（Tasks + Dataview）

```dataview
TABLE WITHOUT ID
  file.link AS "项目",
  choice(length(filter(file.tasks, (t) => !t.completed)) > 0, "✅", "⏳") AS "状态",
  dateformat(date(today) + dur("7 days"), "yyyy-MM-dd") AS "下周截止"
FROM "Projects"
WHERE contains(file.tasks.due, this.due)
```

### 元数据映射规则
| Tasks字段 | Dataview等效字段 | 注意事项 |
|-----------|-----------------|----------|
| 📅 due | task.due | 必须格式正确才被解析为日期 |
| ⏳ scheduled | task.scheduled | 同上 |
| ➕ created | task.created | 同上 |
| 🛫 start | task.start | 同上 |
| � priority | task.priority | 高/中/低映射为数值 |
| 🔁 recurrence | task.recurring | 布尔值 |

## 配置选项详解

### 全局设置
- **任务完成时间**：设置完成任务时自动添加的completion日期格式
- **重复规则设置**：配置重复任务的行为（更新哪个日期、保留历史等）
- **默认优先级**：新任务的默认优先级
- **日期格式**：显示日期的格式（不影响存储格式）
- **任务状态**：自定义任务状态符号和含义
- **查询默认排序**：未指定排序时的默认顺序

### 高级配置
- **忽略文件夹**：指定不扫描的文件夹
- **忽略标签**：指定不索引的标签
- **状态符号映射**：自定义状态符号与含义映射

```json
[
{"symbol": " ", "name": "todo"},
{"symbol": "/", "name": "in-progress"},
{"symbol": "x", "name": "done"},
{"symbol": "-", "name": "cancelled"}
]
```

- **优先级映射**：自定义优先级符号与数值映射
- **日期推断规则**：配置如何从任务文本推断日期
- **自定义任务状态系统**：Tasks 5.0+引入的扩展状态系统
  - 配置路径：Obsidian设置 → 插件 → Tasks → 状态
  - 状态符号映射示例：
  
  ```json
  [
    {"symbol": " ", "name": "todo", "nextStatus": "in-progress"},
    {"symbol": "/", "name": "in-progress", "nextStatus": "done"},
    {"symbol": "x", "name": "done", "nextStatus": "todo"},
    {"symbol": "-", "name": "cancelled", "nextStatus": "todo"},
    {"symbol": "p", "name": "pending", "nextStatus": "in-progress"}
  ]
  ```
  
  - 工作流转换规则：可配置每个状态的下一个默认状态，支持自定义工作流
  - 查询语法：`status is todo` 或 `status is not done`



## 性能与限制

### 严格性能准则
1. **索引延迟**：文件修改后，Tasks索引更新需要1-3秒
2. **大库优化**：根据官方基准测试，在标准硬件上，超过2000个活跃任务时建议使用路径过滤
   - **性能基准（Intel i5/16GB RAM/SSD）**：
     - 500任务：索引时间<0.5秒
     - 2000任务：索引时间≈1.8秒
     - 5000+任务：建议使用exclude path参数
3. **排除路径参数**：使用`exclude path`减少索引范围，基于字符串包含匹配
   - `exclude path "Archived/"` 排除包含Archived的文件夹及子文件夹
   - `exclude path "Templates/"` 排除包含Templates的文件夹
   - `exclude path "/Archive/"` 更严格的匹配，避免误匹配My Archive等
   - `exclude path "Templates/" exclude path "Archive/"` 同时排除多个路径
   
   **性能误区**：
   - Tasks的路径过滤是基于字符串包含的，而非精确匹配
   - 如果你的文件夹叫 My Archive，它也会被 `exclude path "Archive"` 排除
   - 为了严格匹配，建议使用 `exclude path "/Archive/"` 格式
   - 支持简单的前缀匹配，复杂glob模式的支持程度有限

4. **大型库性能优化**（超过10,000个任务）
   - **按需查询**：避免在常驻侧边栏的笔记中放置过于复杂的查询
   - **查询策略**：将复杂查询放在需要时才打开的专用笔记中
   - **组合优化**：结合`exclude path`和`limit`指令，减少单次查询的任务数量
5. **避免全局查询**：永远不要在大型库中使用无过滤条件的查询
6. **查询缓存**：复杂查询结果会被缓存，但源文件修改后会失效

### 硬性限制
- **最大查询结果**：单个查询返回结果限制为10,000条
- **嵌套分组深度**：最多支持3级嵌套分组
- **重复规则深度**：默认最多生成50个重复实例
  - **限制调整**：可通过修改插件设置中的"最大重复实例数"调整此限制
  - **性能影响**：超过100可能导致性能下降，特别是在移动设备上
  - **最佳实践**：根据实际需要设置合理值，一般项目建议50-100个实例
- **查询语法限制**：不支持正则表达式（与Dataview不同）
- **时区限制**：所有日期基于系统本地时区，不支持时区转换

### 移动端功能限制
- **iOS/Android交互限制**：
  - 点击任务文本可能直接跳转到源文件，而非编辑模式
  - 长按任务显示上下文菜单，包含完成、编辑等选项
- **自动完成行为**：
  - 移动端网络延迟可能导致自动完成操作响应较慢
  - 重复任务生成可能在网络恢复后批量执行
- **性能注意事项**：
  - 移动设备上建议限制单个查询的任务数量（500条以内）
  - 减少复杂分组和排序操作以提升响应速度
  - 考虑使用`exclude path`排除不常用文件夹
- **移动端特有限制**：
  - 不支持自定义排序/分组函数
  - 长按任务而非右键显示上下文菜单
  - 重复任务生成可能延迟至网络恢复
  - 建议单个查询限制在500条以内

## 最佳实践与陷阱

### 推荐实践
- **文件组织**：将任务按项目/上下文存放在单独文件夹
- **元数据一致性**：统一使用Emoji或内联字段语法，避免混合
- **查询优化**：先使用`path`缩小范围，再应用其他过滤条件
- **块ID管理**：为关键任务添加块ID以便建立依赖关系
- **历史保留**：定期归档已完成任务，减少活跃任务数量

### 常见陷阱
1. **日期格式错误**：非ISO格式日期（2024/12/31）不会被识别
2. **空格缺失**：任务文本与元数据间缺少两个空格导致解析失败
3. **重复规则冲突**：多个重复规则在同一任务上导致不可预测行为
4. **依赖循环**：任务A依赖任务B，任务B又依赖任务A，形成死锁
5. **移动任务**：将任务移动到新文件会破坏块ID依赖关系
6. **YAML冲突**：YAML frontmatter中的同名字段会覆盖内联字段
7. **时区误解**：跨时区协同时，本地日期可能导致不一致

## 真实场景示例

### 1. 每日待办清单

```tasks
not done
(due before today) OR (scheduled before today) OR (start before today)
priority is high
hide recurrence rule
sort by priority, due
```

### 2. 项目进度跟踪

```tasks
path includes "Projects/Website"
not done
group by heading
group by priority
limit 20
```

### 3. 习惯追踪器

```tasks
is recurring
priority is low
scheduled after 7 days ago
group by recurrence
sort by scheduled
```

### 4. 依赖任务管理

```tasks
depends on project-start
not done
sort by due
```

### 5. 周回顾模板

#### 本周完成

```tasks
done
completed after last week
sort by completed
```

#### 未完成任务

```tasks
not done
due before next week
sort by due
```

#### 新任务

```tasks
created after last week
not done
sort by created
```

## 故障排除指南

### 任务不显示在查询结果中？
- [ ] 检查日期格式是否为严格ISO标准（YYYY-MM-DD）
- [ ] 验证任务文本与元数据之间是否有两个空格
- [ ] 确认文件是否在"忽略文件夹"配置中
- [ ] 检查任务是否被YAML frontmatter中的同名字段覆盖

### 重复任务不生成？
- [ ] 验证原始任务是否已完成
- [ ] 检查重复规则语法是否精确匹配规范
- [ ] 确认时区设置是否影响日期计算

### 数据恢复程序
- 当查询结果异常时：
  1. 关闭Obsidian
  2. 删除 `.obsidian/plugins/obsidian-tasks/cache` 目录
  3. 重启应用触发完整索引重建

## 版本迁移指南

### 从3.x到4.x的变更
- **破坏性变更**：
  - 查询语法更新，部分操作符变更
  - 重复规则引擎重构
  - 性能优化导致索引机制变化
- **迁移建议**：
  - 备份现有库
  - 逐步更新查询语句
  - 验证重复任务行为

### 从4.x到5.x的变更
- **扩展状态系统**：引入自定义任务状态
- **依赖机制增强**：添加自动状态转换
- **性能参数调整**：新增exclude path参数

## 高级自定义功能

### 自定义排序函数（Tasks 5.0+ 支持）

**基本示例**：
```javascript
sort by function(task) {
  return task.description.length;
}
```

**高性能建议**：处理空值情况以提高排序性能
```javascript
// 按截止日期排序，处理空值
return task.due ? task.due.moment.valueOf() : 0;

// 按优先级排序，处理空值
return task.priority ? task.priority : 999;
```

### 自定义渲染器API
- 用于修改任务显示格式
- 支持自定义状态图标
- 可扩展任务属性显示

#### 自定义渲染器示例

```javascript
// 自定义渲染器配置示例
const customRenderer = (task, element) => {
  // 为高优先级任务添加红色边框和火焰图标
  if (task.priority === 'high') {
    element.style.borderColor = '#ff0000';
    element.style.borderWidth = '2px';
    element.style.borderStyle = 'solid';
    
    // 查找或创建优先级元素
    const priorityElement = element.querySelector('.task-priority');
    if (priorityElement) {
      priorityElement.innerHTML = '🔥';
      priorityElement.title = '高优先级';
    }
  }
  
  // 为接近截止日期的任务添加黄色背景
  if (task.due && task.due.diffNow().days <= 3) {
    element.style.backgroundColor = '#fff3cd';
  }
  
  // 为已完成任务添加绿色对勾
  if (task.status === 'done') {
    const statusElement = element.querySelector('.task-status');
    if (statusElement) {
      statusElement.innerHTML = '✅';
    }
  }
};

// 在插件设置中应用
// settings.customRenderer = customRenderer;
```

## 参考资源
- 官方文档: https://github.com/obsidian-tasks-group/obsidian-tasks
- 查询语法详解: https://publish.obsidian.md/tasks/Queries
- 重复规则指南: https://publish.obsidian.md/tasks/Recurring+Tasks
- 社区论坛: https://forum.obsidian.md/c/plugins/tasks/66
- GitHub Issues: https://github.com/obsidian-tasks-group/obsidian-tasks/issues

## 版本兼容性
- 最低Obsidian版本: 1.0.0
- 推荐Obsidian版本: 1.4.0+
- Tasks插件版本要求: 4.0.0+（推荐5.0.0+以获得完整功能）
- 与Dataview兼容版本: Dataview 0.5.53+

**版本特性对应**：
- Tasks 4.0+：引入Status Type查询系统，重复规则引擎重构
- Tasks 5.0+：扩展状态系统，自定义排序函数语法更新，show tree功能

**重要提示**：Tasks插件持续更新，新版本可能引入向后不兼容的变更。升级前请备份库，并查阅更新日志。某些高级功能（如自定义状态符号、show tree）需要最新版本支持。