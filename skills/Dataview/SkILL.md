---
name: obsidian-dataview
description: 使用Dataview插件查询Obsidian笔记中的元数据和内容，支持Dataview Query Language (DQL)、DataviewJS、内联查询等多种形式。创建动态列表（List）、表格（Table）、任务（Task）、日历（Calendar）视图，进行过滤、排序、分组、展平（Flatten）、聚合计算等操作。适用于任务管理、书籍/文章阅读列表、知识图谱、日志分析、项目跟踪等场景。当用户提到Dataview、DQL、查询、表格视图、任务列表、日历视图、元数据字段、inline fields等时使用。
---
# Obsidian Dataview 技能
此技能使AI代理能够创建、编辑和优化有效的Dataview查询代码块，包括DQL查询、DataviewJS脚本、内联表达式、元数据添加等。所有语法、函数、字段均严格基于官方文档（https://blacksmithgu.github.io/obsidian-dataview/）
及社区高可信实践，确保准确性、性能和兼容性。避免非标准写法，优先使用FROM限制查询范围以防大仓库卡顿。

## 概述
Dataview是Obsidian最强大的动态查询插件，提供实时索引引擎，可索引笔记的YAML frontmatter、内联字段（inline fields）、文件系统元数据、标签、任务、链接等。使用DQL（Dataview Query Language）进行声明式查询，支持聚合函数、日期运算、数组操作等。查询结果实时更新，支持交互（如勾选任务）。

**重要：Dataview 是只读的** - Dataview 仅动态渲染视图和计算结果，**不会修改或写入任何 Markdown 源码**。所有查询结果都是实时计算的临时视图。
查询主要嵌入在Markdown代码块中：

DQL块：```dataview
DataviewJS块：dataviewjs ... 
内联查询：在文本中直接使用 =  或 $=  前缀

**重要**：DQL不是SQL，有关键差异（如命令顺序、FROM限制、表达式语法）。始终使用FROM缩小范围，避免全库扫描。

## 元数据添加方式

Dataview索引两种主要元数据源：

1. **YAML Frontmatter**（前置元数据）

```yaml
---
status: todo
priority: 5
due: 2024-12-31
tags: [task, urgent]
rating: 8.5
---
```
2. Inline Fields（内联字段）
行级：key:: value 或 [key:: value]（推荐用于句子中）
隐藏键：(long-key:: value)
任务/列表项：- [ ] Buy milk [due:: 2024-12-31]


键名自动规范化：小写、空格转短横线（My Key → my-key）。支持UTF-8、Emoji，但Emoji排序/渲染可能跨平台不一致。

### 元数据覆盖与冲突逻辑

当YAML Frontmatter和内联字段（Inline Fields）使用相同的键名时：
- **合并为列表**：该字段会变成一个包含两者的列表（Array/List），而不是简单的覆盖
- **使用陷阱**：在DQL中使用时需注意这种"意外的数组"可能导致比较运算失效（例如 `status = "todo"` 会因为变成了 `["todo", "todo"]` 而失败）
- **独立字段**：但 `file.tasks` 和 `file.lists` 中的内联字段是独立的，不会被YAML覆盖（tasks/lists的inline fields独立且不被覆盖）

## 元数据类型（Metadata Types）

| 类型     | 示例解析方式                       | 排序行为     | 渲染方式                   | 支持主要操作                     |
| -------- | ---------------------------------- | ------------ | -------------------------- | -------------------------------- |
| Text     | "Hello" 或 Hello                   | 字母序       | 纯文本                     | contains, replace, lower, length |
| Number   | 42 或 3.14                         | 数值序       | 数字                       | +, -, *, /, round, floor, ceil   |
| Date     | 2024-12-31 或 2024-12-31T10:00:00  | 时间序       | 可配置格式（默认人类可读） | .year, .month, + dur, date(now)  |
| Duration | 2 hours 30 minutes                 | 时间长度序   | 持续时间字符串             | date + dur, date2 - date1        |
| Link     | [[Page]] 或 [[Page\|Display]] 或 [[Page#Header\|Display Text]] | 链接文本序   | 可点击链接                 | link(path), embed(link)；Link对象可访问子属性：link.path（文本路径）、link.display（显示文本）、link.subpath（子路径，如 #Header）、link.type（链接类型）、link.embed（是否嵌入，布尔值） |
| List     | [a, b, c]（YAML数组）或内联字段格式。YAML中纯文本无空格可不引号，含空格或特殊字符需引号 | 逐元素排序   | 数组                       | length, contains, map, filter, +（连接列表） |
| Object   | {a: 1, b: "x"}（仅YAML支持）       | 子字段排序   | 嵌套结构                   | .subfield, extract()             |
| Boolean  | true / false                       | false 在 true 之前（升序） | 渲染为 true/false           | 支持逻辑运算：!field, field1 && field2, field1 || field2 |
| Null     | null 或 字段缺失                   | 最低值       | 空或null                   | == null, != null                 |

### 类型自动转换陷阱

⚠️ **重要警告**：Dataview会尝试自动推断类型，可能导致意外行为：
- **日期格式**：建议使用 ISO 格式（2024-12-31），其他格式可能解析失败
- **日期解析**：`date()` 解析失败返回 null，使用 `date(text, format)` 时格式需使用 Luxon 标记（如 "yyyy-MM-dd"）
- **纯数字文本**：建议加引号如 "42" 避免被解析为数字
- **空值处理**：所有用户自定义字段都必须用 `default()` 或 `!= null` 检查
- **字符串比较**：`contains()` 对字符串是模糊匹配，精确匹配应使用 `econtains()`
- **字符串转义**：在字符串中包含特殊字符时需转义，如：`"It\'s a test"`（单引号）、`"Path: C:\\folder\\file"`（反斜杠）、`"She said \"hello\""`（双引号）



## 隐式文件元数据字段（Implicit File Fields）

所有笔记自动获得 file.* 字段：

| 字段                   | 类型    | 描述                                       |
| ---------------------- | ------- | ------------------------------------------ |
| file.name              | Text    | 文件名（不含扩展名）                       |
| file.folder            | Text    | 从根目录开始的完整路径（不含文件名），如 "folder/subfolder" |
| file.path              | Text    | 完整路径                                   |
| file.ext               | Text    | 扩展名（通常 "md"）                        |
| file.link              | Link    | 到本文件的链接，包含 display name（显示文本），排序时按显示文本排序 |
| file.size              | Number  | 文件大小（字节）                           |
| file.ctime / file.cday | Date    | 创建时间（带/不带时间）                    |
| file.mtime / file.mday | Date    | 修改时间（带/不带时间）                    |
| file.tags              | List    | 所有标签（子标签拆分，如 #a/b → #a, #a/b） |
| file.etags             | List    | 显式标签（不拆分子标签）                   |
| file.inlinks           | List    | 传入链接（谁链接到我）                     |
| file.outlinks          | List    | 传出链接（我链接到谁）                     |
| file.aliases           | List    | YAML aliases                               |
| file.tasks             | List    | 本文件所有任务（仅包含当前文件物理存在的任务，不包含子页面任务） |
| file.lists             | List    | 本文件所有列表项（含任务，仅包含当前文件物理存在的列表项，不包含子页面任务） |
| file.frontmatter       | Object  | 包含文件中所有原始的 YAML 数据（原始键值对） |
| file.day               | Date    | 当文件名匹配 yyyy-MM-dd 或 yyyymmdd 格式，或存在 YAML date 字段时自动生成 |
| file.starred           | Boolean | **已基本废弃**。Obsidian 1.0+ 移除 Starred 核心插件后，file.starred 在新版本中通常始终返回 false。建议改用自定义字段（如 starred:: true）或配合 Bookmarks 插件的其他方式实现收藏标记功能。 |

## 任务隐式字段（Task Implicit Fields）

任务（Task）查询拥有特殊的隐式字段，与文件字段不同。**重要**：Tasks 继承所有页面级的 frontmatter 和内联字段（如页面中的 project:: 字段可在任务中直接访问），页面级的同名字段优先级低于任务行自己的内联字段。

**状态字段关系**：
- `completed`：只在方括号为 `[x]` 时为 true（最严格的完成判断）
- `checked`：只要方括号内**非空**（包括 `/`、`>`、`-` 等自定义状态）就为 true
- `status`：直接返回方括号内的原始字符，便于支持 Tasks 插件等多种自定义状态

| 字段              | 类型     | 描述                                      |
| ----------------- | -------- | ----------------------------------------- |
| completed         | Boolean  | 是否完成（仅 "x" 为 true，其他自定义状态如 "/" 为 false） |
| fullyCompleted    | Boolean  | 子任务是否全部完成                        |
| text              | Text     | 任务文本（不含元数据）                    |
| visual            | Text     | 渲染文本（含链接格式化）                  |
| line              | Number   | 行号                                      |
| lineCount         | Number   | 任务占用的行数                            |
| path              | Text     | 所属文件路径                              |
| status            | Text     | 方括号中的字符（如 " ", "x", "/" 等） |
| checked           | Boolean  | 方括号中是否有非空格字符                  |
| subtasks          | List     | 子任务数组                                |
| children          | List     | 子任务（同 subtasks）                     |
| real              | Boolean  | 是否真实任务（非列表项）                  |
| task              | Boolean  | 是否为真实任务（同 real）                 |
| section           | Link     | 指向包含任务的标题链接                    |
| tags              | List     | 任务文本中的标签                          |
| outlinks          | List     | 任务中的传出链接                          |
| link              | Link     | 指向任务块的链接                          |
| parent            | Task     | 父任务（如果有）                          |
| blockId           | Text     | 任务块的ID（如果有）                      |
| annotated         | Boolean  | 任务是否包含内联字段                      |
| due               | Date     | 任务截止日期。Dataview自动识别任务行中的📅表情符号日期（如📅 2024-01-01），也支持[due:: YYYY-MM-DD]内联字段格式。注意：只有格式正确时才会被索引为Date类型，否则会被索引为字符串 |
| created           | Date     | 任务创建日期。Dataview自动识别任务行中的➕表情符号日期（如➕ 2024-01-01），也支持[created:: YYYY-MM-DD]内联字段格式。注意：只有格式正确时才会被索引为Date类型，否则会被索引为字符串 |
| completion        | Date     | 任务完成日期。Dataview自动识别任务行中的✅表情符号日期（如✅ 2024-01-01），也支持[completion:: YYYY-MM-DD]内联字段格式。注意：只有格式正确时才会被索引为Date类型，否则会被索引为字符串 |
| start             | Date     | 任务开始日期。Dataview自动识别任务行中的🛫表情符号日期（如🛫 2024-01-01），也支持[start:: YYYY-MM-DD]内联字段格式。注意：只有格式正确时才会被索引为Date类型，否则会被索引为字符串 |
| scheduled         | Date     | 任务计划日期。Dataview自动识别任务行中的⏳表情符号日期（如⏳ 2024-01-01），也支持[scheduled:: YYYY-MM-DD]内联字段格式。注意：只有格式正确时才会被索引为Date类型，否则会被索引为字符串 |



## DQL 查询结构

```dataview
<查询类型> [字段列表 或 表达式]
FROM <来源>                   # 必须第一，限制范围防卡顿
WHERE <条件>                  # 可多个
SORT <字段> [ASC|DESC]        # 可多个
GROUP BY <字段> [AS 别名]      # 可多个
FLATTEN <数组字段> [AS 别名]  # 将数组展开为多行。新字段（如 task）仅可在 FLATTEN 之后的 WHERE、SORT、GROUP BY 中使用
LIMIT <数字>                  # 最多一个
```

**重要规则**：
- **FROM若存在必须在最前**（除查询类型外），用于缩小查询范围，防止全库扫描导致卡顿
- **FROM 是可选的（语法上非必须）**，但强烈推荐几乎所有实际场景都使用 FROM。
  - 无 FROM 时，查询默认扫描**整个知识库**，在大仓库（几千甚至上万笔记）中极易导致严重卡顿甚至 Obsidian 崩溃。
  - 只有在极少数明确只需要查询**当前页面**（例如内联字段计算、当前文件任务统计）或测试非常小规模查询的场景，才可以省略 FROM。
  - **最佳实践**：**默认永远写 FROM**，除非你 100% 确定当前查询只针对当前文件且不需要扫描其他笔记。
- **命令顺序执行**：FLATTEN创建的新字段只能在后续WHERE中使用
- **NULL值处理**：SORT默认将NULL值排最后
- **LIMIT保护**：建议使用LIMIT限制输出，防止返回结果过多导致崩溃

## 查询类型（Query Types）

- **LIST**：简单列表（默认显示file.link）
- **TABLE**：表格（需指定列，可用AS别名）
- **TABLE WITHOUT ID**：创建表格时不包含默认的file.link列（第一列），适用于自定义主键场景
- **TASK**：交互任务列表（支持勾选）
- **CALENDAR**：日历视图（需一个date字段，如CALENDAR file.ctime）

## 数据命令详解（Data Commands）

### DQL 语法的严格约束
- **操作符优先级**：DQL 的 & 和 | 是位运算符，逻辑运算必须使用 AND 和 OR
- **相等判断**：在 DQL 中，= 和 == 都是等值判断，但在 JavaScript (DataviewJS) 中 = 是赋值

### FROM 逻辑的深度补充
- **交集**：`FROM #tag AND #tag2`（同时包含两个标签的页面）
- **并集**：`FROM #tag OR #tag2`（包含任一标签的页面）
- **排除**：`FROM -#tag`（不包含该标签的页面）
- **组合**：`FROM "folder" AND [[Link]]`（文件夹内且链接到某处的页面）

### WHERE 逻辑运算
**注意：始终使用 AND / OR 进行逻辑判断**
- `WHERE field1 = 1 AND field2 = 2` ✅
- `WHERE field1 = 1 && field2 = 2` ❌ (在某些版本中可能产生解析歧义)

### 其他命令
- **SORT**：排序，支持多字段、ASC/DESC
- **GROUP BY**：分组，结果每组一行，rows为数组。
  **关键规则**：任何非分组键字段在 GROUP BY 后都必须通过 `rows` 访问（如 `rows.file.link`），否则会报错或返回 Null。分组键通过 `key` 访问（或使用 AS 定义的别名）。
  **访问分组键**：使用 `key` 字段或 `TABLE WITHOUT ID key AS "分组名称"`，也可使用 `GROUP BY field AS 别名` 直接设置别名。
- **FLATTEN**：展平数组字段（如file.tasks），每元素一行
  **复杂行为**：
  - **多级展平**：`FLATTEN file.tasks AS task` 后，`task.subtasks` 仍需要再次FLATTEN
  - **与WHERE的顺序**：FLATTEN创建的新字段只能在后续WHERE中使用，不能在FLATTEN前的WHERE中使用
  - **空数组处理**：FLATTEN空数组会移除该行（类似SQL的UNNEST），这是预期行为
  - **笛卡尔积效应**：如果一个页面有 5 个任务，使用 FLATTEN file.tasks 后，该页面在表格中会占据 5 行
  - **空值剔除**：如果对一个不存在的字段使用 FLATTEN，该行会从结果中完全消失（类似于 Inner Join）。应建议配合 default() 使用：`FLATTEN default(myField, [null])`
- **LIMIT**：限制行数

## 表达式与运算符（Expressions）

- 字面量：数字、字符串" "、date("2024-01-01")、dur("1 day")、true/false、[[link]]
- 字段访问：status、file.mtime.year、rows.file.link（GROUP BY后使用rows访问原字段）、key（访问分组键）
- 运算符：+ - * / % > < = != && || ! =~ !~（& 和 | 为位运算符，不建议用于逻辑运算）
- this：当前上下文文件（仅在DQL中有效；DataviewJS中使用 dv.current()）
- 链式：file.tasks.text

## 函数参考（Functions）

### 构造函数

- date(any) / date(text, format)
- dur(any)
- link(path, [display])
- list(...values)
- object(key1, value1, ...)

### 数值函数

- round(num, [digits]), floor, ceil, trunc
- min/max/sum/product/average/reduce

### 数组/列表函数

- length, contains, filter, map, flat, sort, reverse, unique, join, concat(list1, list2)
- econtains(list, item) - 精确包含（不同于contains的模糊匹配）
- flat(array) - 将嵌套数组拉平
- nonnull(array) - 移除数组中的所有 null 值（在清理 rows 时极度有用）

### 字符串函数

- lower/upper, replace, regexreplace, split, startswith, endswith, truncate
- regexmatch(pattern, string) - 正则匹配

### 逻辑与类型

- typeof, contains, icontains, all/any/none
- default(field, value) - 处理null值，空值提供默认值
- choice(condition, ifTrue, ifFalse) - DQL中的三元运算符

### 日期函数

- dateformat(date, string) - 自定义日期格式（函数名必须全小写：dateformat，注意：JavaScript 区分大小写，不要写成 dateFormat；格式字符串使用 Luxon 日期格式（类Moment.js语法，注意大小写敏感：yyyy=年份, MM=月份, dd=日期, HH=小时, mm=分钟），如 "yyyy-MM-dd"）
- localtime(date) - 转换为本地时区
- striptime(date) - 去除时间部分，仅保留日期

### 日期处理陷阱

- **格式严格**：`date("2024-01-01")` 严格要求ISO格式，模糊日期如 "Jan 1, 2024" 解析会失败
- **file.day识别**：file.day 识别逻辑：文件名匹配 yyyy-MM-dd 或 yyyyMMdd，或YAML中的date字段
- **时间比较**：`date(today)` 包含当前时间，应使用 `date(today).startOf("day")` 进行日期比较
- **时区问题**：默认使用UTC时区，如需本地时区应使用 `localtime()` 函数
- **Luxon格式化**：dateformat 使用的是 Luxon 标记，注意大小写：
  - `yyyy` (4位年), `MM` (2位月), `dd` (2位日), `HH` (24h时), `mm` (分)
  - 常见错误：用户常写成 `YYYY-MM-DD`（Moment.js 格式），这在 Luxon 中会导致年份解析异常
- **今天比较**：进行"是否是今天"的比较时，必须使用 `striptime(file.mtime) = date(today)`，因为 file.mtime 包含具体时分秒

### 链接函数

- meta(link) - 获取链接的元数据对象
- embed(link, [embed?]) - 创建嵌入式链接（第二参数控制是否真正嵌入）

（完整函数列表超过50个，优先使用官方分类）

## 完整示例

### 任务跟踪器（Task Tracker）

```dataview
TASK
FROM "Tasks" OR #task
WHERE !completed
SORT priority DESC, due ASC
```

**重要限制**：TASK查询不支持GROUP BY（会报错），如需分组应使用变通方案（如使用TABLE查询或按file.path分组）。

### 阅读列表（Reading List）

```dataview
TABLE WITHOUT ID file.link AS "书名", author, rating, status AS "状态", date-finished AS "完成日期"
FROM #book OR "Reading"
WHERE status != "dropped"
SORT rating DESC
LIMIT 20
```

### 项目概览（Project Overview）

```dataview
TABLE file.name AS "项目", status, 
       length(file.tasks) AS "任务数", 
       round(
         length(filter(file.tasks, (t) => t.completed)) / length(file.tasks) * 100, 
         1
       ) + "%" AS "完成率"
FROM "Projects"
WHERE length(file.tasks) > 0
GROUP BY status
SORT key ASC
```

### 日志索引（Daily Log Index）

```dataview
CALENDAR file.day
FROM "Daily"
WHERE mood
```

**CALENDAR WHERE 实际限制**：
 虽然语法上支持复杂表达式，但**复杂过滤条件（比较、函数、逻辑运算等）在日历视图中经常不生效或结果不稳定**。
 **推荐做法**：
 - 优先使用 FROM + 标签/文件夹/链接 来缩小范围
 - WHERE 只用于简单的存在性检查（如 `WHERE mood`、`WHERE rating`、`WHERE !completed`）

## 内联查询（Inline Queries）

- DQL 内联：`= file.mtime.relative()` 或 `= length(filter(file.tasks, (t)=>!t.completed))`
- DataviewJS 内联：`$= dv.current().file.name`

### 重要限制

- **渲染限制**：仅在阅读/预览模式渲染，编辑模式显示源码
- **上下文限制**：执行上下文中的 `this` 指向包含该查询的页面
- **性能陷阱**：内联查询每次视图更新都会执行，大量内联查询会导致严重卡顿
- **复杂性限制**：复杂的内联查询可能降低文档可读性，建议复杂逻辑使用DataviewJS块

## DataviewJS 简介

用于复杂逻辑、自定义渲染，提供完整的API：

```dataviewjs
const pages = dv.pages("#book").where(p => p.rating > 8);
dv.table(["书名", "评分"], pages.map(p => [p.file.link, p.rating]));
```

### 核心API

#### 渲染函数
- `dv.table(headers, rows)` - 创建表格（headers: string[], rows: any[][]）
- `dv.list(elements)` - 创建列表（elements: 可包含link, text等）
- `dv.taskList(tasks, [groupByFile?])` - 创建任务列表（第二参数控制是否按文件分组）
- `dv.view(viewName, [input])` - 调用views文件夹中的自定义JS视图
- `dv.el(elementType, text)` - 创建DOM元素（如"div", "span"）
- `dv.header(level, text)` - 创建标题（1-6）

#### 数据获取
- `dv.pages([source])` - 查询页面（必须指定来源，否则全库扫描会卡顿），返回 **DataArray** 对象（Obsidian 特有代理类），拥有 .where(), .groupBy() 等特有方法
  - **重要**：DataArray 不是普通数组，直接使用原生 JS 数组方法（如 .reduce, .forEach）可能报错或行为异常，建议：
    - 使用 DataArray 专用 API（.where(), .sort(), .limit() 等）
    - 如需原生数组方法，可先转换：`Array.from(pages)` 或 `pages.array()`
  - **DataArray 陷阱**：
    - `.where()` 是 DataArray 方法（推荐）
    - `.filter()` 是原生 JS 数组方法（在 DataArray 上可能失效）
    - 后果：如果在 `dv.pages().array()` 之后调用 `.where()` 会报错
- `dv.current()` - 当前文件对象
- `dv.page(path)` - 单页查询
- `dv.tryQuery(dqlString)` - 执行DQL字符串
- `dv.query(dqlString)` / `dv.queryMarkdown(dqlString)` - **异步**操作，返回 Promise，必须配合 await 使用
  - **异步示例**：
    ```javascript
    const result = await dv.query("LIST FROM #tag");
    if (result.successful) { /* 处理结果 */ }
    ```

#### 工具函数
- `dv.array(any)` - 转换为数组
- `dv.isArray(any)` - 类型检查
- `dv.date(any)` - 日期解析
- `dv.duration(any)` - 持续时间
- `dv.compare(a, b)` - 比较函数
- `dv.equal(a, b)` - 深度相等

### 安全使用示例

```dataviewjs
// ✅ 安全：始终限制来源
const pages = dv.pages("#project")
  .where(p => p.status === "active")
  .sort(p => p.due)
  .limit(100);  // 硬限制防止内存溢出

// ✅ 安全：空值处理（严格null检查避免0/false误判）
// 方法1：严格null检查（推荐，避免0/false误判）
dv.table(
  ["项目", "截止日期", "优先级"],
  pages.map(p => [
    p.file.link,
    p.due != null ? dv.dateformat(p.due, "yyyy-MM-dd") : "未设置",
    p.priority > 3 ? "🔴 高" : "🟢 正常"
  ])
);
// 方法2：使用空值合并运算符（??）（推荐）
// p.due ?? "未设置"

// 注意：?? 只在左侧为 null 或 undefined 时生效
// 如果你想把 0、空字符串等 falsy 值也视为"未设置"，仍需使用三元运算符：
// p.due != null ? dv.dateformat(p.due, "yyyy-MM-dd") : "未设置"

// ❌ 危险：以下代码在大库中会导致崩溃
// const all = dv.pages();  // 永远不要这样做！
// dv.table(["Name"], all.map(p => [p.file.name]));
```

## 常见模式与最佳实践

- 始终加FROM #tag 或 "folder" 限制范围
- 使用FLATTEN处理file.tasks / file.lists
- GROUP BY + rows.length / sum(rows.rating)
- 日期比较：due <= date(today) + dur("7 days")
- 避免全库查询：无FROM可能导致卡顿
- 任务交互：TASK视图支持勾选
- 性能：优先WHERE过滤，LIMIT限制输出

## 性能与限制的严重警告

### 必须避免的危险操作

- **全库扫描**：永远不要使用 `dv.pages()` 无参数或 `FROM` 无限制，在大库中会导致Obsidian卡顿数秒甚至崩溃
- **无限结果**：查询返回结果过多（>10000行）可能导致Obsidian崩溃
- **复杂正则**：FROM中使用复杂正则（如 `/^Diary/\d{4}/`）比文件夹路径慢
- **WHERE正则**：避免在 WHERE 中使用正则匹配（如 `file.name =~ /.../`），性能极差；优先用文件夹路径 `FROM "Daily"`

### 重要限制

- **索引延迟**：新创建的笔记可能在Dataview索引更新前有1-3秒延迟。当修改元数据后，Dataview需要几秒钟重新索引。在DataviewJS中频繁调用dv.view可能会导致UI闪烁。
- **递归查询限制**：`file.inlinks` 和 `file.outlinks` 不会递归解析，仅直接链接。如果你需要查询"链接的链接"，Dataview DQL无法直接通过单行命令实现（除非手动链式 file.inlinks.inlinks），这种场景必须推荐DataviewJS。
- **任务查询限制**：TASK查询不支持GROUP BY，SORT对任务特定字段需要放在FLATTEN之后
- **CALENDAR限制**：WHERE只支持简单的文件级过滤，不支持复杂表达式

### 性能优化建议

- **限制范围**：始终使用FROM缩小查询范围
- **早过滤**：优先使用WHERE过滤，减少后续处理的数据量
- **硬限制**：使用LIMIT限制输出行数
- **缓存**：对于复杂计算，考虑使用内联字段缓存结果
- **批量处理**：避免在一个文件中使用大量内联查询（会导致渲染卡顿）

## 常见场景的函数用法示例

### 空值处理

```dataview
TABLE file.name AS "文件", 
       default(author, "未知作者") AS "作者",
       default(rating, 0) AS "评分"
FROM #book
```

### 条件值

```dataview
TABLE file.name AS "项目",
       choice(priority > 3, "🔴 高优先级", "🟢 正常") AS "优先级",
       choice(status = "completed", "✅ 完成", "⏳ 进行中") AS "状态"
FROM #project
```

### 完成率计算

```dataview
TABLE file.name AS "项目",
       length(file.tasks) AS "任务数",
       round(
         length(filter(file.tasks, t => t.completed)) / length(file.tasks) * 100,
         1
       ) + "%" AS "完成率"
FROM "Projects"
WHERE length(file.tasks) > 0
```

### 日期格式化

```dataview
TABLE file.name AS "文件",
       dateformat(file.ctime, "yyyy-MM-dd") AS "创建日期",
       dateformat(due, "yyyy-MM-dd") AS "截止日期"
FROM #task
WHERE due
```

### 字符串操作

```dataview
TABLE file.name AS "文件名",
       lower(file.name) AS "小写",
       replace(file.name, "-", " ") AS "替换空格",
       startswith(file.name, "2024") AS "2024年文件"
FROM "Daily"
```

### 数组操作

```dataview
TABLE file.name AS "文件",
       length(file.tags) AS "标签数",
       join(file.tags, ", ") AS "标签列表",
       contains(file.tags, "important") AS "重要文件"
FROM #note
```

### 安全的数据获取

```dataviewjs
// 安全的文件查询
const urgentTasks = dv.pages("#task")
  .filter(p => p.priority && p.priority > 3)
  .sort(p => p.due)
  .limit(50);

// 安全的表格渲染
dv.table(
  ["任务", "截止日期", "优先级"],
  urgentTasks.map(p => [
    p.file.link,
    p.due ? dv.dateformat(p.due, "yyyy-MM-dd") : "未设置",
    p.priority ? p.priority : "未设置"
  ])
);
```

## 参考

- 官方文档：https://blacksmithgu.github.io/obsidian-dataview/
- 函数：https://blacksmithgu.github.io/obsidian-dataview/reference/functions
- 元数据：https://blacksmithgu.github.io/obsidian-dataview/annotation/metadata-pages
- 查询结构：https://blacksmithgu.github.io/obsidian-dataview/queries/structure