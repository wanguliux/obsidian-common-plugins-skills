---
name: templater
description: 创建和编辑带有变量、函数、控制流和Obsidian特定语法的Templater模板。当处理包含Templater模板的.md文件、创建动态内容，或用户提及Templater、模板变量或模板函数时使用。
---

# Templater 技能

本技能使技能兼容的代理能够创建和编辑有效的 Templater 模板，包括所有 Templater 特定的语法扩展。

## 版本信息

本文档基于 **Templater v2.18.1+** 编写。请注意：
- 从 v1.x 到 v2.x 有大量 breaking changes，许多 API 签名已变更
- 旧版本可能不支持本文档中的所有功能
- 使用前请检查您的 Templater 版本

**重要**：本文档主要介绍 v2.0+ 的现代用法，旧版兼容信息请参考文档末尾的兼容部分。

### v1 → v2 主要 breaking changes
- **tp.file.create_new**：参数顺序变化（template 第一，filename 第二，可选）
- **tp.system.prompt / suggester**：移除 title 参数（multi_suggester 有 title）
- **tp.frontmatter**：仍只读；修改用 app.fileManager.processFrontMatter
- **tp.web.random_picture / daily_quote**：可能不稳定或受限（建议避免依赖第三方）
- **旧 tp.file.create()**：已移除
- **沙盒增强**：系统命令、HTTP 请求在移动端/非 trusted 模板受限

**升级建议**：备份 vault → 更新插件 → 测试模板

## 版本兼容性指南

### 版本迁移指南

| 功能 | 旧版本 (<1.10.0) | 新版本 (1.16.0+) |
|------|----------------|----------------|
| 用户输入 | tp.system.prompt() | tp.system.prompt() (参数变化) |
| Frontmatter访问 | tp.frontmatter.get() | tp.frontmatter 直接访问属性 |
| 文件创建 | tp.file.create() | tp.file.create_new() |
| 静态函数 | 无 | tp.static |
| 事件系统 | 无 | tp.hooks |
| 模板配置 | 无 | tp.config |

### 关键API变化说明

#### tp.system.prompt() 参数变化
- 旧版本：`tp.system.prompt(text, default, throw_on_cancel)`
- v1.x 版本：`tp.system.prompt(prompt_text: string, default_value?: string, throw_on_cancel?: boolean, multiline?: boolean, title?: string)`
- v2.x 版本：`tp.system.prompt(prompt_text?: string, default_value?: string, throw_on_cancel: boolean = false, multiline?: boolean = false)`
- **重要**：v2.x 中移除了 `title` 参数

#### Frontmatter访问方式变化
- 旧版本：使用 `tp.frontmatter.get('key')` 获取属性
- 新版本：直接使用 `tp.frontmatter.key` 访问属性
- **注意**：`tp.frontmatter` 是只读的，用于快速访问元数据。要修改元数据，需要使用 `app.fileManager.processFrontMatter` 结合 `tp.hooks`

#### 文件创建API变化
- 旧版本：`tp.file.create(filename, content, open)`
- v1.x 版本：`tp.file.create_new(filename: string, template?: TFile | string, open_new?: boolean, folder?: TFolder)`
- v2.x 版本：`tp.file.create_new(template: TFile | string, filename?: string, open_new?: boolean = false, folder?: TFolder | string)`
- **重要**：v2.x 中参数顺序发生了重大变化，template 现在是第一个参数

#### 新增功能
- `tp.static`：提供不依赖执行上下文的静态函数访问
- `tp.hooks`：提供事件系统，支持模板执行前后的回调
- `tp.config`：提供模板执行的配置信息，包括运行模式和参数

## 安全警告

Templater 允许执行任意 JavaScript 代码，存在安全风险：

### 主要风险
- **执行系统命令**：`tp.system.executeCommandWithOutput()` 可执行系统命令，可能被恶意模板利用，在 v2.x 中已被 Sandbox 严格限制
- **文件操作**：可读写保险库中的文件
- **网络请求**：可发送 HTTP 请求到任意服务器，`tp.obsidian.requestUrl()` 可能带来 CORS 或敏感数据泄露风险，在移动端/沙盒环境中有严格限制

### 安全建议
1. **只使用可信模板**：不要从不可信来源复制模板
2. **限制系统命令**：谨慎使用 `tp.system.executeCommandWithOutput()`，在 v2.x 中已被 Sandbox 严格限制
3. **审查网络请求**：谨慎使用 `tp.obsidian.request()` 和 `tp.obsidian.requestUrl()`，避免向不可信的服务器发送请求，防止敏感数据泄露。**注意**：这些函数在移动端/沙盒环境中有严格限制。
4. **启用模板信任**：在 Templater v2.x 中，"Only run trusted templates" 选项默认开启，这是一个重要的安全措施
5. **审查模板代码**：使用前检查模板中的 JavaScript 代码
6. **备份保险库**：定期备份您的 Obsidian 保险库

## 概述

Templater 是 Obsidian 的一个强大插件，允许您创建带有变量、函数和控制流的动态模板。它通过模板语法扩展了 Markdown，这些语法在插入模板时会被处理。

## 基本语法

### 模板变量

```markdown
<% tp.file.title %>          <!-- 文件标题（无扩展名） -->
<% tp.file.folder() %>       <!-- 文件夹名（相对） -->
<% tp.file.folder(true) %>   <!-- 绝对文件夹路径 -->
<% tp.file.path(true) %>     <!-- vault 相对路径 -->
<% tp.file.content %>        <!-- 执行前文件内容（只读，快照） -->
```

### 用户输入

现代 Templater 使用 `tp.system.prompt` 进行用户输入：

```javascript
<%* 
// 基本输入
const title = await tp.system.prompt('Enter a title:');

// 带默认值
const name = await tp.system.prompt('Enter your name:', 'John Doe');

// 多行输入
const description = await tp.system.prompt('Enter description:', '', false, true);
-%>
```

**tp.system.prompt参数说明**：
```javascript
tp.system.prompt(prompt_text?: string, default_value?: string, throw_on_cancel: boolean = false, multiline?: boolean = false)
```
- `prompt_text`：提示文本（可选）
- `default_value`：默认值（可选）
- `throw_on_cancel`：取消时是否抛出错误（可选，默认false）
- `multiline`：是否支持多行输入（可选，默认false，设置为true时显示文本域）

### 下拉选择

```javascript
<%* 
// 基本用法
const selected = await tp.system.suggester(
  ['Option 1', 'Option 2'],
  ['value1', 'value2']
);

// 高级用法（带显示函数）
const item = await tp.system.suggester(
  item => item.display,
  [
    {display: "Option 1", value: "value1"},
    {display: "Option 2", value: "value2"}
  ]
);
tR = item.value;
-%>
```

**tp.system.suggester参数说明**：
```javascript
tp.system.suggester(text_items: string[] | ((item: T) => string), items: T[], throw_on_cancel?: boolean, placeholder?: string, limit?: number)
```
- `text_items`：显示的文本数组，或从item生成文本的函数
- `items`：实际返回的值数组
- `throw_on_cancel`：取消时是否抛出错误（可选，默认false）
- `placeholder`：输入框占位符（可选）
- `limit`：显示的最大项目数（可选）

### 多选下拉选择

Templater 还提供 `tp.system.multi_suggester` 用于多选：

```javascript
<%* 
const choices = await tp.system.multi_suggester(
  ["选项1", "选项2", "选项3"],
  ["val1", "val2", "val3"],
  false,
  "请选择...",
  undefined,
  "多选示例"
);
tR = `选择了: ${choices.join(', ')}`;
-%>
```

**tp.system.multi_suggester参数说明**：
```javascript
tp.system.multi_suggester(
  text_items: string[] | ((item: T) => string),
  items: T[],
  throw_on_cancel?: boolean,
  placeholder?: string,
  limit?: number,
  title?: string
)
```
- `text_items`：显示的文本数组，或从item生成文本的函数
- `items`：实际返回的值数组
- `throw_on_cancel`：取消时是否抛出错误（可选，默认false）
- `placeholder`：输入框占位符（可选）
- `limit`：显示的最大项目数（可选）
- `title`：选择框标题（可选）
- 返回值为选中项的数组

### 文件创建

```javascript
<%* await tp.file.create_new('Content', 'New Note.md', true) %>
<%* await tp.file.create_new('Content', 'New Note.md', true, 'Projects/Meetings') %>
<%* await tp.file.create_new(tp.file.find_tfile('Templates/Note Template.md'), 'New Note.md', true) %>
```

**说明**：Templater v2.x API 签名为 `create_new(template: TFile | string, filename?: string, open_new?: boolean = false, folder?: TFolder | string)`
- `template`：模板内容（字符串）或模板文件（TFile对象）
- `filename`：新文件的名称（可选，默认 "Untitled"）
- `open_new`：是否打开新创建的文件（可选，默认false）
- `folder`：目标文件夹（TFolder对象或字符串路径）（可选）

**重要**：`tp.file.create_new` 返回的是一个 `Promise<TFile>` 对象，这对于后续需要立即对新创建文件进行操作（如移动、重命名）至关重要。

**示例说明**：
- 基本用法：`tp.file.create_new('文件内容', '文件名.md', true)` - 第一个参数为模板内容字符串
- 指定文件夹：`tp.file.create_new('文件内容', '文件名.md', true, 'folder/path')` - 第一个参数为模板内容字符串
- 使用模板文件：`tp.file.create_new(template文件对象, '文件名.md', true)` - 第一个参数为模板文件对象

**文件夹路径说明**：
- 文件夹路径应使用相对于保险库根目录的完整路径，如 `'Projects/Meetings'`
- 路径取决于用户的实际文件夹结构
- 确保目标文件夹已存在，否则需要先创建文件夹

**重要警告**：
- 避免在用于 `create_new` 的模板文件中包含自引用逻辑，可能导致无限创建
- 当使用 `TFile` 对象作为模板参数时，确保该文件不会再次触发创建逻辑

**示例**：
```javascript
<%* 
// 安全的创建方式 - 使用字符串内容
const content = `# New Note
Created on ${tp.date.now('YYYY-MM-DD')}`;
await tp.file.create_new(content, 'New Note.md', true);
-%>

<%* 
// 使用模板文件
const templateFile = tp.file.find_tfile('Templates/Note Template.md');
await tp.file.create_new(templateFile, 'New Note.md', true);
-%>
```

### 日期和时间

```markdown
<% tp.date.now('YYYY-MM-DD') %>
<% tp.date.now('YYYY-MM-DD HH:mm:ss') %>
<% tp.date.now('YYYY-MM-DD', 1) %> <!-- 明天的日期 -->
<% tp.date.now('YYYY-MM-DD HH:mm', 0, new Date()) %> <!-- 使用指定的 Date 对象 -->
<% tp.date.yesterday('YYYY-MM-DD') %>
<% tp.date.tomorrow('YYYY-MM-DD') %>
```

**tp.date.now 参数说明**：
```javascript
tp.date.now(format: string, offset?: number, date?: Date)
```
- `format`：日期时间格式字符串（如 'YYYY-MM-DD'）
- `offset`：天数偏移量（可选，默认 0）
- `date`：基础日期对象（可选，默认当前时间）

**示例**：
- `tp.date.now('YYYY-MM-DD')`：当前日期
- `tp.date.now('YYYY-MM-DD', 1)`：明天的日期
- `tp.date.now('YYYY-MM-DD', -1)`：昨天的日期
- `tp.date.now('YYYY-MM-DD HH:mm', 0, new Date('2023-12-25'))`：指定日期的格式化结果

**tp.date.weekday 参数说明**：
```javascript
tp.date.weekday(format: string, dayOfWeek: number, date?: Date)
```
- `format`：日期时间格式字符串
- `dayOfWeek`：星期几（0=周日，1=周一，...，6=周六），返回最近的那个星期几
- `date`：基础日期对象（可选，默认当前时间）

**示例**：
- `tp.date.weekday('YYYY-MM-DD', 1)`：最近的周一
- `tp.date.weekday('YYYY-MM-DD', 6, new Date('2023-12-25'))`：2023年12月25日之后最近的周六

**tp.date.month 参数说明**：
```javascript
tp.date.month(format: string, offset?: number, date?: Date)
```
- `format`：日期时间格式字符串
- `offset`：月份偏移量（可选，默认 0）
- `date`：基础日期对象（可选，默认当前时间）

**示例**：
- `tp.date.month('YYYY-MM')`：当前月份
- `tp.date.month('YYYY-MM', 1)`：下个月
- `tp.date.month('YYYY-MM', -1)`：上个月

### 静默执行（保留换行）

Templater 支持多种执行模式，可组合使用 `+`（保留换行）和 `-`（去除后续换行）修饰符：

#### 输出命令
| 语法 | 说明 |
|------|------|
| `<% ... %>` | 输出命令，不保留换行 |
| `<%+ ... %>` | 输出命令，保留换行 |
| `<% ... -%>` | 输出命令，去除后续换行 |
| `<%+ ... -%>` | 输出命令，保留换行但去除后续多余换行 |

#### 执行命令
| 语法 | 说明 |
|------|------|
| `<%* ... %>` | 执行命令，不保留换行 |
| `<%*+ ... %>` | 执行命令，保留换行 |
| `<%* ... -%>` | 执行命令，去除后续换行 |
| `<%*+ ... -%>` | 执行命令，保留换行但去除后续多余换行 |

```javascript
<%+ tp.date.now('YYYY-MM-DD') %>

<%*+ 
// 执行 JavaScript 代码并保留换行
const today = new Date();
tR = `今天是 ${today.toLocaleDateString()}`;
-%>
```

**说明**：`+` 修饰符用于保留模板中的空白行，避免输出挤在一起；`-` 修饰符用于去除标签后的多余换行，使输出更加紧凑。这在需要保持模板结构清晰时非常有用。

### 执行模式（修饰符）

| 语法 | 输出 | 保留换行 | 去除后续换行 |
|------|------|----------|--------------|
| `<% ... %>` | 是 | 否 | 否 |
| `<%+ ... %>` | 是 | 是 | 否 |
| `<% ... -%>` | 是 | 否 | 是 |
| `<%+ ... -%>` | 是 | 是 | 是 |
| `<%* ... %>` | 否 | 否 | 否 |
| `<%*+ ... %>` | 否 | 是 | 否 |
| `<%* ... -%>` | 否 | 否 | 是 |
| `<%*+ ... -%>` | 否 | 是 | 是 |

## 高级语法

### JavaScript 执行

```javascript
<%* 
const today = new Date();
const formattedDate = today.toISOString().split('T')[0];
tp.file.rename(`Note ${formattedDate}`);
-%>
```

### 控制流

```javascript
<%* 
// 直接使用 tR，不要声明
if (tp.file.title.includes('Meeting')) {
  tR = 'This is a meeting note.';
} else {
  tR = 'This is a regular note.';
}
-%>

<%* 
tR = '';
for (let i = 1; i <= 5; i++) {
  tR += `- Item ${i}\n`;
}
-%>
```

**最佳实践**：在复杂的逻辑块中，应始终推荐使用 `tR += '...'`（累加）而非 `tR = '...'`（覆盖），除非明确意图是清空之前的内容。这可以保护之前已生成的文本不被意外覆盖。

**示例**：
```javascript
<%* 
let message = "Hello";
tR += `${message} World\n`; // 使用 += 而不是 =
%>
```

### 函数

```javascript
<% tp.file.cursor() %>
<% tp.file.rename('New Title.md') %>
<% tp.file.move('New Folder/New Title.md') %>
<% tp.system.executeCommandWithOutput('echo "Hello World"') %>
<% tp.user.myFunction() %>
```

### Frontmatter

```markdown
<%* 
const tag = await tp.system.prompt('Enter a tag:');
-%>

---
title: <% tp.file.title %>
date: <% tp.date.now('YYYY-MM-DD') %>
tags:
  - template
  - <% tag %>
---
```

### tp.frontmatter

现代 Templater 提供 `tp.frontmatter` 对象以**只读**方式访问当前文件的 frontmatter：

```javascript
<%* 
// 读取 frontmatter
const title = tp.frontmatter.title;
const tags = tp.frontmatter.tags;
%>
```

**重要说明**：
- `tp.frontmatter` 是一个只读对象，仅用于快速访问元数据
- 在 YAML frontmatter 中直接使用异步命令（如 `await tp.system.prompt()`）会导致解析错误
- 推荐在模板开头使用 JavaScript 执行块收集用户输入，然后在 frontmatter 中使用变量
- **关键限制**：`tp.frontmatter` 的可用性取决于文件状态：
  
  **现有文件**
  - 模板变量（如 `<% tp.file.title %>`）可用
  - 可以通过 `tp.frontmatter` 访问现有元数据
  
  **新创建文件**
  - 在文件创建完成前，`tp.frontmatter` 可能为空或不可用

**修改 Frontmatter 的正确方法**：
要修改 Frontmatter，需要结合 `tp.hooks.on_all_templates_executed` 使用 Obsidian 的 `app.fileManager.processFrontMatter`：

```javascript
<%* 
tp.hooks.on_all_templates_executed(async () => {
  const file = tp.file.find_tfile(tp.file.path(true));
  await app.fileManager.processFrontMatter(file, (fm) => {
    fm["status"] = "active";
    fm["priority"] = "high";
    // 添加标签
    if (!fm.tags) fm.tags = [];
    if (!fm.tags.includes("new-tag")) fm.tags.push("new-tag");
    // 移除标签
    fm.tags = fm.tags.filter(tag => tag !== "old-tag");
  });
});
%>
```

## Obsidian 集成

### tp.obsidian API

Templater 提供 `tp.obsidian` 对象以访问 Obsidian 原生 API，用于高级操作。`tp.obsidian.app` 与全局 `app` 对象是等价的，都是指向 Obsidian 应用实例的引用。

### tp.config

现代 Templater 提供 `tp.config` 对象以访问模板配置信息：

```javascript
<%* 
// 获取当前模板文件
const templateFile = tp.config.template_file;
tR = `Template: ${templateFile.name}`;

// 获取目标文件
const targetFile = tp.config.target_file;
if (targetFile) {
  tR += `\nTarget: ${targetFile.name}`;
}

// 获取运行模式
const runMode = tp.config.run_mode;
tR += `\nRun mode: ${runMode}`;
%>
```

**tp.config属性说明**：
- `template_file`：当前模板文件（TFile对象）
- `target_file`：目标文件（TFile对象，可能为undefined）
- `run_mode`：运行模式（字符串，如 "CreateNewNote"、"InsertTemplate" 等）
- `args`：传递给模板的参数（对象，可能为undefined）

**tp.config.args详细说明**：
- 当通过命令调用模板并传递参数时，参数会存储在此对象中
- 例如，通过Templater URI调用时传递的参数会被解析到该对象
- 可用于构建接受外部参数的模板，实现更灵活的模板行为

**用途**：
- 调试模板执行环境
- 根据运行模式执行不同逻辑
- 构建条件性模板行为
- 处理外部传递的参数

**示例**：
```javascript
<%* 
// 检查是否有传递的参数
if (tp.config.args) {
  const title = tp.config.args.title || 'Default Title';
  const tags = tp.config.args.tags || ['template'];
  tR = `Title: ${title}\nTags: ${tags.join(', ')}`;
} else {
  tR = 'No arguments provided';
}
-%>
```

```javascript
<%* 
// 获取当前保险库中的所有文件
const files = app.vault.getFiles(); // 推荐使用全局 app 对象
tR = `保险库中有 ${files.length} 个文件`;
-%>

<%* 
// 获取文件创建时间和修改时间
const creationDate = tp.file.creation_date('YYYY-MM-DD');
const modifiedDate = tp.file.last_modified_date('YYYY-MM-DD');
tR = `创建时间: ${creationDate}\n`;
tR += `修改时间: ${modifiedDate}`;
-%>
```

### HTTP 请求

Templater 支持通过 `tp.obsidian` 进行 HTTP 请求：

```javascript
<%* 
// 发送 HTTP 请求（返回响应对象）
const response = await tp.obsidian.request({
  url: 'https://api.example.com/data',
  method: 'GET',
  headers: {
    'Content-Type': 'application/json'
  }
});
tR = `响应状态: ${response.status}`;
-%>

<%* 
// 包含授权头的请求示例
const apiToken = "your_api_token_here";
const response = await tp.obsidian.request({
  url: 'https://api.example.com/data',
  method: 'GET',
  headers: {
    'Authorization': `Bearer ${apiToken}`,
    'Content-Type': 'application/json',
    'Accept': 'application/json'
  }
});
tR = `响应状态: ${response.status}`;
-%>

<%* 
// 发送 HTTP 请求（返回响应文本）
const responseText = await tp.obsidian.requestUrl('https://api.example.com/data');
tR = responseText;
-%>
```

### HTTP 请求错误处理

**重要**：HTTP 请求可能因网络问题、服务器错误等原因失败，应使用 try-catch 处理：

```javascript
<%* 
try {
  const response = await tp.obsidian.request({
    url: 'https://api.example.com/data',
    method: 'GET'
  });
  
  if (response.status === 200) {
    const data = JSON.parse(response);
    tR = `成功获取数据: ${data.message}`;
  } else {
    tR = `请求失败: ${response.status}`;
  }
} catch (error) {
  console.error('网络错误:', error);
  new app.Notice('无法获取数据，请检查网络连接', 5000);
  tR = `错误: ${error.message}`;
}
-%>
```

## 用户函数

### 创建用户函数

1. **配置用户脚本文件夹**：
   
   **步骤 1：创建文件夹**
   - 在 Obsidian 保险库中创建一个文件夹，如 `scripts/`
   
   **步骤 2：设置路径**
   - 打开 Obsidian 设置（⚙️）
   - 导航到：`社区插件` → `Templater` → `设置`
   - 在 "User Scripts Folder" 字段中输入文件夹路径，如 `scripts/`
   - 点击 "确定" 保存设置
   
   **注意**：
   - Templater 默认没有预设的用户脚本文件夹
   - 路径是相对于保险库根目录的相对路径
   - 确保文件夹名称与设置中的路径一致

2. **创建 JavaScript 文件**（例如：`scripts/myFunctions.js`）：

```javascript
// scripts/myFunctions.js - 导出多个函数
module.exports = {
  greet: function(tp, name) {
    return `Hello, ${name}! Current file: ${tp.file.title}`;
  },
  
  getRandomNumber: function(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
  }
};

// 或者 - 导出单个函数
function mainFunction(tp) {
  return tp.file.title;
}
module.exports = mainFunction;
```

**重要说明**：
- **必须传递 tp 参数**：用户函数需要接收 `tp` 对象作为第一个参数，才能在函数内部使用 Templater 功能
- **不同导出方式的调用**：
  - 导出多个函数：`tp.user.scriptName.functionName(tp, ...)`
  - 导出单个函数：`tp.user.scriptName(tp)`

### 使用用户函数

```javascript
<%* 
// 调用导出多个函数的脚本
const greeting = tp.user.myFunctions.greet(tp, 'World');
tR = greeting;
-%>

<%* 
// 调用导出单个函数的脚本
const title = tp.user.mainFunction(tp);
tR = title;
-%>
```

## 模板示例

### 每日笔记模板

```markdown
---
title: <% tp.date.now('YYYY-MM-DD') %>
date: <% tp.date.now('YYYY-MM-DD HH:mm:ss') %>
tags:
  - daily
---

<%* 
const today = tp.date.now('YYYY-MM-DD');
const dayName = tp.date.now('dddd');
tR = `# ${today} ${dayName}\n\n`;
tR += "## Tasks\n\n";
tR += "- [ ] \n\n";
tR += "## Notes\n\n\n";
tR += "## Reflection\n\n\n";
tR += "## Tomorrow's Tasks\n\n";
tR += "- [ ] \n\n";
-%>

<% tp.file.cursor() %>
```

### 会议笔记模板

```javascript
<%* 
const meetingTitle = await tp.system.prompt('Meeting title:');
const meetingType = await tp.system.prompt('Meeting type:');
const attendees = await tp.system.prompt('Attendees (comma-separated):');
-%>

---
title: Meeting - <% meetingTitle %>
date: <% tp.date.now('YYYY-MM-DD HH:mm:ss') %>
tags:
  - meeting
  - <% meetingType %>
attendees: "<% attendees %>"
---

# Meeting - <% meetingTitle %>

## Date & Time
<%* 
const start = new Date();
const end = new Date(start.getTime() + 60 * 60 * 1000);
// 使用tp.date.now格式化日期，避免依赖全局moment对象
const startTime = tp.date.now('YYYY-MM-DD HH:mm', 0, start);
const endTime = tp.date.now('YYYY-MM-DD HH:mm', 0, end);
tR = `${startTime} - ${endTime}`;
-%>

## Attendees
- 

## Agenda
- 

## Notes


## Decisions

- 

## Action Items

- [ ] 

## Next Steps

- 

<% tp.file.cursor() %>
```

### 项目笔记模板

```javascript
<%* 
const projectTitle = await tp.system.prompt('Project title:');
const projectType = await tp.system.prompt('Project type:');
-%>

---
title: <% projectTitle %>
date: <% tp.date.now('YYYY-MM-DD') %>
tags:
  - project
  - <% projectType %>
status: planning
priority: medium
---

# <% projectTitle %>

## Overview


## Goals

- 

## Timeline

### Milestones

- [ ] 

### Tasks

- [ ] 

## Resources

- 

## Notes


<% tp.file.cursor() %>
```

## 最佳实践

1. **使用描述性变量名**在模板中
2. **组织用户函数**在单独的文件中以提高可维护性
3. **测试模板**在广泛使用前
4. **使用注释**在复杂模板中解释逻辑
5. **定期备份模板**
6. **使用 tR += 而非 tR =**：在复杂逻辑中，推荐使用 `tR += '...'`（累加）而非 `tR = '...'`（覆盖），特别是在新文件 + 多模板块的情况下，这可以避免意外覆盖已生成的内容

## 错误处理与调试

### 调试技巧

```markdown
<%* 
// 在控制台输出调试信息（在 Obsidian 开发者工具中查看）
console.log('Debug info:', tp.file.title);
console.log('Frontmatter:', tp.frontmatter);

// 使用 try-catch 处理错误
try {
  const result = await tp.file.include("[[Template]]");
  tR = result;
} catch (error) {
  console.error('Error:', error);
  tR = `Error: ${error.message}`;
  // 显示用户友好的错误消息
  new app.Notice(`Template error: ${error.message}`, 5000);
}
%>
```

### 错误处理最佳实践

#### 处理文件不存在的情况

```markdown
<%* 
// 处理文件不存在的情况
const filePath = "Notes/Example.md";
try {
  const file = app.vault.getAbstractFileByPath(filePath);
  if (!file) {
    throw new Error(`文件不存在: ${filePath}`);
  }
  const fileContent = await app.vault.read(file);
  tR = fileContent;
} catch (error) {
  console.error(`读取文件错误: ${error.message}`);
  new app.Notice(`错误: ${error.message}`, 5000);
  tR = `⚠️ 无法加载内容`;
}
%>
```

#### 处理用户取消输入的情况

```markdown
<%* 
try {
  const title = await tp.system.prompt('Enter a title:', '', true);
  if (!title) {
    throw new Error('用户取消了输入');
  }
  tR = `Title: ${title}`;
} catch (error) {
  console.error('输入错误:', error.message);
  new app.Notice('操作已取消', 3000);
  tR = '⚠️ 操作已取消';
}
%>
```

#### 处理网络错误

```markdown
<%* 
try {
  const response = await tp.obsidian.request({
    url: 'https://api.example.com/data',
    method: 'GET'
  });
  tR = `Response status: ${response.status}`;
} catch (error) {
  console.error('网络错误:', error.message);
  new app.Notice('网络请求失败，请检查网络连接', 5000);
  tR = '⚠️ 网络请求失败';
}
%>
```

**调试说明**：
- **开发者工具**：在 Obsidian 中按 `Ctrl+Shift+I`（Windows/Linux）或 `Cmd+Option+I`（Mac）打开开发者工具，在 Console 标签页查看 `console.log` 输出
- **Notice**：`new Notice(message, timeout)` 用于在 Obsidian 界面显示临时消息，提升用户体验
- **异步调试**：对于异步操作，记得使用 `try-catch` 包裹并添加 `await`

### 异步支持

Templater 支持 `await` 关键字处理异步操作：

```markdown
<%* 
// 异步包含其他模板
const content = await tp.file.include("[[Template]]");
tR = content;

// 异步获取文件内容
const file = app.vault.getAbstractFileByPath("Notes/Example.md");
const fileContent = await app.vault.read(file);
tR += fileContent;
%>
```

### 异步执行顺序

**重要说明**：
- **执行机制**：Templater 首先会解析文档中的所有 `<%* %>` 块和 `<% %>` 块，然后再按它们在文档中出现的顺序依次执行（类似变量提升 hoisting）
- 异步操作（使用 `await`）会阻塞后续执行，直到完成
- 多个异步命令会按顺序执行，确保操作的一致性
- 未使用 `await` 的异步操作会并行执行，可能导致不可预测的结果
- 执行块中的异步操作会影响整个模板的执行进度，确保操作的顺序性
- **注意**：执行顺序可能与直观的从上到下阅读顺序不完全一致，特别是在复杂的模板中

### Web API 注意事项

**重要**：`tp.web` 相关函数（如 `random_picture()`、`daily_quote()`）在 Templater v2.x 中已完全移除（出于安全原因）。这些函数在 v2.0+ 版本中不再可用。

### tp.static

现代 Templater 提供 `tp.static` 对象以访问静态函数，主要用于：

1. **提供不依赖执行上下文的静态函数访问**
2. **适用于需要传递 Templater 功能到回调函数的场景**
3. **确保在任何上下文中都能访问核心功能**
4. **在用户脚本（User Scripts）中访问那些通常需要活动编辑器上下文才能调用的函数**

**重要说明**：`tp.static` 对象仍需要在 JavaScript 执行块（`<%*`）内使用，但它提供的函数可以被传递到其他函数或回调中使用。它不能在执行块外（如 `<%`）直接使用。

**用户脚本中的使用示例**：

```javascript
// scripts/myFunctions.js - 在用户脚本中使用 tp.static
module.exports = {
  formatDate: function(date, tpStatic) {
    // 在用户脚本中，无法直接访问 tp 对象
    // 但可以通过传递 tp.static 来使用 Templater 的日期函数
    return tpStatic.date.now('YYYY-MM-DD', 0, date);
  }
};
```

**在模板中调用**：

```markdown
<%* 
const formattedDate = tp.user.myFunctions.formatDate(new Date(), tp.static);
tR = formattedDate;
-%>
```

```markdown
<%* 
// 在执行块内使用
const now = tp.static.date.now('YYYY-MM-DD');
tR = now;
-%>

<%* 
// 作为参数传递
function formatDate(date, tpStatic) {
  return tpStatic.date.now('YYYY-MM-DD', 0, date);
}
const formatted = formatDate(new Date(), tp.static);
tR = formatted;
-%>
```

### tp.hooks

Templater 提供 `tp.hooks` 对象以注册事件监听器，用于响应模板执行的不同阶段。**最核心的 Hook 是 `tp.hooks.on_all_templates_executed(callback)`**，它是处理模板执行后清理工作（如修改刚生成的文件的属性）的唯一安全方式。

**重要**：Hook 的注册应尽量放置在模板代码的最顶层。如果在复杂的逻辑中（如条件判断后）才注册 Hook，而此时触发点已过，回调将永远不会执行。

```markdown
<%* 
// 注册事件监听器
tp.hooks.on('on_all_templates_executed', () => {
  // 所有模板执行完成后执行的清理操作
  console.log('All templates executed');
});

// 注册一次性事件监听器
tp.hooks.once('on_template_executed', (templateFile, targetFile) => {
  console.log('Template executed:', templateFile.name, '→', targetFile.name);
});
%>
```

## 常见错误与陷阱

### 模板开发常见陷阱

1. **参数顺序错误**：将 `filename` 写在 `template` 前（tp.file.create_new）。
2. **忘记使用 await**：对异步操作（如 create_new、prompt）不使用 await，导致竞态条件。
3. **在 frontmatter 中使用异步命令**：直接在 YAML frontmatter 中写 `<% await tp.system.prompt() %>` 会导致解析错误。
4. **在循环中频繁创建文件**：循环内多次调用 `tp.file.create_new` 会严重影响性能。
5. **依赖 tp.web.***：tp.web.random_picture / daily_quote 在 v2 中已移除或不稳定。
6. **忘记使用 tR +=**：在多模板块中使用 `tR =` 覆盖内容，导致之前的输出被清空。

### 推荐的解决方案

1. **tp.file.create_new 标准写法**：
   ```javascript
   <%* 
   const newFile = await tp.file.create_new(
     template,  // 第一位：模板内容或 TFile
     filename,  // 第二位：文件名
     open,      // 是否打开
     folder     // 文件夹路径
   );
   -%>
   ```

2. **修改 frontmatter 的安全方式**：
   ```javascript
   <%* 
   tp.hooks.on_all_templates_executed(async () => {
     const file = tp.file.find_tfile(tp.file.path(true));
     await app.fileManager.processFrontMatter(file, (fm) => {
       fm.status = "active";
       fm.tags = [...(fm.tags || []), "new"];
     });
   });
   -%>
   ```

3. **批量文件操作**：预生成内容后再批量创建文件，减少 I/O 操作。

### tp.hooks.on_all_templates_executed 的重要性

**tp.hooks.on_all_templates_executed** 是 Templater v2 中最重要、最安全的模板执行后处理机制。它的核心优势：

- **执行时机**：在所有模板代码执行完毕后运行，确保文件已完全创建。
- **安全性**：提供了一个安全的上下文来修改刚创建的文件（如修改 frontmatter）。
- **避免竞态**：避免了因异步操作顺序导致的文件未就绪问题。

**推荐用法**：
- 修改新创建文件的 frontmatter
- 对新文件进行后续操作（如移动、重命名）
- 执行需要文件完全就绪的逻辑

**示例**：
```javascript
<%* 
// 在模板开始处注册 Hook（推荐）
tp.hooks.on_all_templates_executed(async () => {
  // 安全地修改刚创建的文件
  const file = tp.file.find_tfile(tp.file.path(true));
  await app.fileManager.processFrontMatter(file, (fm) => {
    fm.createdAt = tp.date.now('YYYY-MM-DD');
  });
  
  // 显示完成通知
  new app.Notice('模板执行完成，文件已更新！', 5000);
});

// 其他模板逻辑...
-%>
```

**事件系统详细说明**：

| 事件 | 触发条件 | 参数 | 使用场景 |
|------|---------|------|----------|
| `on_template_create` | 模板文件被创建时 | `(templateFile: TFile)` | 初始化新模板、添加默认内容 |
| `on_template_delete` | 模板文件被删除时 | `(templateFile: TFile)` | 清理与模板相关的资源 |
| `on_template_executed` | 单个模板执行完成后 | `(templateFile: TFile, targetFile: TFile)` | 记录模板使用情况、执行后续操作 |
| `on_all_templates_executed` | 所有模板执行完成后 | `()` | 执行全局清理、显示完成消息、修改刚生成文件的属性 |

**核心用法示例**：使用 `on_all_templates_executed` 修改刚生成的文件属性

```markdown
<%* 
tp.hooks.on_all_templates_executed(async () => {
  // 确保文件已完全创建
  const file = tp.file.find_tfile(tp.file.path(true));
  if (file) {
    // 修改文件的 frontmatter
    await app.fileManager.processFrontMatter(file, (fm) => {
      fm.status = 'active';
      fm.priority = 'high';
    });
  }
});
%>
```

**示例：记录模板执行**
```markdown
<%* 
tp.hooks.on('on_template_executed', async (templateFile, targetFile) => {
  const logFile = app.vault.getAbstractFileByPath('Templater Log.md');
  if (logFile) {
    const content = await app.vault.read(logFile);
    const newContent = `${content}\n- ${new Date().toISOString()}: ${templateFile.name} → ${targetFile.name}`;
    await app.vault.modify(logFile, newContent);
  }
});
%>
```

**用途**：
- 执行清理操作
- 记录模板执行历史
- 触发后续操作
- 实现复杂的模板工作流

### 异步执行最佳实践

**推荐做法**：
```markdown
<%* 
// 按顺序执行多个异步操作
const content1 = await tp.file.include("[[Template1]]");
const content2 = await tp.file.include("[[Template2]]");
tR = content1 + "\n\n" + content2;
%>
```

## 参考

### 功能分类

为了更清晰地组织 Templater 的功能，以下是按类别分组的 API 参考：

#### 文件操作
- `tp.file.create_new()` - 创建新文件
- `tp.file.rename()` - 重命名当前文件
- `tp.file.move()` - 移动当前文件
- `tp.file.cursor()` - 放置光标位置
- `tp.file.creation_date()` - 文件创建日期
- `tp.file.last_modified_date()` - 文件最后修改日期
- `tp.file.include()` - 包含其他文件内容
- `tp.file.find_tfile()` - 通过路径获取 TFile 对象
- `tp.file.selection()` - 获取选中的文本（仅在对已有文件插入模板且有文本被选中时有效）

#### 日期和时间
- `tp.date.now()` - 当前日期/时间
- `tp.date.yesterday()` - 昨天的日期
- `tp.date.tomorrow()` - 明天的日期
- `tp.date.weekday()` - 获取特定星期几的日期
- `tp.date.month()` - 月份操作

#### 用户交互
- `tp.system.prompt()` - 用户输入提示
- `tp.system.suggester()` - 下拉选择提示
- `tp.system.clipboard()` - 读取剪贴板

#### 系统操作
- `tp.system.executeCommandWithOutput()` - 执行 shell 命令并返回输出
- **重要警告**：该函数在 Templater v2.x 中已被 Sandbox 严格限制。在大多数现代环境中（尤其是移动端和 sandbox 模式）此函数无法正常工作，强烈建议不要在生产模板中使用。

#### Web 操作
- `tp.obsidian.request()` - 发送 HTTP 请求
- `tp.obsidian.requestUrl()` - 发送 HTTP 请求并返回响应文本

**注意**：`tp.web` 相关函数（如 `random_picture()`、`daily_quote()`）在 Templater v2.x 中已完全移除（出于安全原因）。

#### 用户脚本
- `tp.user.scriptName()` - 单个导出函数
- `tp.user.scriptName.func()` - 多个导出函数

#### 高级功能
- `tp.obsidian` - 访问 Obsidian 原生 API
- `tp.static` - 访问静态函数
- `tp.frontmatter` - 访问和修改 frontmatter
- `tp.hooks` - 事件系统
- `tp.config` - 模板配置

### Templater 变量

| 变量 | 描述 | 示例 |
|------|------|------|
| `tp.file.title` | 当前文件标题 | `<% tp.file.title %>` |
| `tp.file.folder()` | 当前文件文件夹（相对路径） | `<% tp.file.folder() %>` |
| `tp.file.folder(true)` | 当前文件文件夹（绝对路径） | `<% tp.file.folder(true) %>` |
| `tp.file.path()` | 当前文件路径（相对路径） | `<% tp.file.path() %>` |
| `tp.file.path(true)` | 当前文件路径（绝对路径） | `<% tp.file.path(true) %>` |
| `tp.file.content` | 当前文件内容（仅在对已有文件执行模板时有效；新建文件时为空）。**注意**：该变量在模板执行开始时即固定，仅包含执行模板前磁盘上的文件内容，不反映执行过程中的动态变化。 | `<% tp.file.content %>` |
| `tp.frontmatter.tags` | 文件标签数组 | `<% tp.frontmatter.tags %>` |
| `tp.frontmatter` | 当前文件的 frontmatter | `<% tp.frontmatter.title %>` |

### Templater 函数

| 函数 | 描述 | 示例 | 异步 |
|------|------|------|------|
| `tp.date.now()` | 当前日期/时间 | `<% tp.date.now('YYYY-MM-DD') %>` | ❌ |
| `tp.date.yesterday()` | 昨天的日期 | `<% tp.date.yesterday('YYYY-MM-DD') %>` | ❌ |
| `tp.date.tomorrow()` | 明天的日期 | `<% tp.date.tomorrow('YYYY-MM-DD') %>` | ❌ |
| `tp.date.weekday()` | 获取特定星期几的日期 | `<% tp.date.weekday('YYYY-MM-DD', 1) %>` | ❌ |
| `tp.date.month()` | 月份操作 | `<% tp.date.month('YYYY-MM-DD', 1) %>` | ❌ |
| `tp.file.create_new()` | 创建新文件 | `<%* await tp.file.create_new('Content', 'New Note.md', true); -%>` | ✅ |
| `tp.file.rename()` | 重命名当前文件 | `<% tp.file.rename('New Title.md') %>` | ❌ |
| `tp.file.move()` | 移动当前文件 | `<% tp.file.move('New Folder/New Title.md') %>` | ❌ |
| `tp.file.cursor()` | 放置光标位置。**注意**：该函数可以接受一个可选的数字参数，例如 `<% tp.file.cursor(1) %>`, `<% tp.file.cursor(2) %>`，允许用户通过快捷键在多个预设光标点之间跳转。 | `<% tp.file.cursor(1) %>` | ❌ |
| `tp.file.cursor_append()` | 在当前光标位置追加内容（常用于多光标场景） | `<% tp.file.cursor_append('追加内容') %>` | ❌ |
| `tp.file.creation_date()` | 文件创建日期 | `<% tp.file.creation_date('YYYY-MM-DD') %>` | ❌ |
| `tp.file.last_modified_date()` | 文件最后修改日期 | `<% tp.file.last_modified_date('YYYY-MM-DD') %>` | ❌ |
| `tp.file.include()` | 包含其他文件内容 | `<%* tR = await tp.file.include('[[Template]]'); -%>` | ✅ |
| `tp.file.find_tfile()` | 通过路径获取 TFile 对象 | `<% tp.file.find_tfile('Notes/Example.md') %>` | ❌ |
| `tp.file.selection()` | 获取选中的文本（仅在对已有文件插入模板且有文本被选中时有效，返回选中字符串或空） | `<% tp.file.selection() %>` | ❌ |
| `tp.file.exists()` | 检查文件/文件夹是否存在，返回 boolean（同步） | `<% tp.file.exists('Folder/Note.md') %>` | ❌ |
| `tp.system.prompt()` | 用户输入提示 | `<% tp.system.prompt('Enter a title:', 'Default', false, true) %>` | ❌ |
| `tp.system.suggester()` | 下拉选择提示 | `<% tp.system.suggester(item => item.display, [{display: "Option 1", value: "value1"}]) %>` | ❌ |
| `tp.system.clipboard()` | 读取剪贴板 | `<%* tR = await tp.system.clipboard(); -%>` | ✅ |
| `tp.system.executeCommandWithOutput()` | 执行 shell 命令并返回输出 | `<% tp.system.executeCommandWithOutput('echo "Hello"') %>` | ❌ |

| `tp.user.scriptName()` | 单个导出函数 | `<% tp.user.myFunction(tp) %>` | ❌ |
| `tp.user.scriptName.func()` | 多个导出函数 | `<% tp.user.myModule.greet(tp, 'World') %>` | ❌ |
| `tp.obsidian` | 访问 Obsidian 原生 API | `<% tp.obsidian.app.vault.getFiles() %>` | ❌ |
| `tp.obsidian.request()` | 发送 HTTP 请求 | `<% tp.obsidian.request({url: 'https://api.example.com'}) %>` | ✅ |
| `tp.obsidian.requestUrl()` | 发送 HTTP 请求并返回响应文本 | `<% tp.obsidian.requestUrl('https://api.example.com') %>` | ✅ |
| `tp.static` | 访问静态函数 | `<% tp.static.date.now('YYYY-MM-DD') %>` | ❌ |

## 工作流示例

### 复杂工作流示例

以下是一个结合多个模板和条件逻辑的复杂工作流示例：

```markdown
<%* 
// 步骤 1: 收集用户输入
const projectType = await tp.system.suggester(
  ['个人项目', '团队项目', '客户项目'],
  ['personal', 'team', 'client']
);

const projectName = await tp.system.prompt('项目名称:', '', true);
const hasDeadline = await tp.system.prompt('有截止日期吗?', 'yes', true, false).toLowerCase() === 'yes';

let deadline = '';
if (hasDeadline) {
  deadline = await tp.system.prompt('截止日期 (YYYY-MM-DD):');
}

// 步骤 2: 创建项目文件夹
const folderPath = `Projects/${projectType}/${projectName}`;
const folder = app.vault.getAbstractFileByPath(folderPath);
if (!folder) {
  await app.vault.createFolder(folderPath);
}

// 步骤 4: 创建项目文件
const projectContent = `---
title: ${projectName}
type: ${projectType}
date: ${tp.date.now('YYYY-MM-DD')}
${deadline ? `deadline: ${deadline}` : ''}
status: planning
---

# ${projectName}

## 概述


## 任务

- [ ] 项目初始化
- [ ] 需求分析
- [ ] 设计阶段
- [ ] 开发阶段
- [ ] 测试阶段
- [ ] 交付

## 资源


## 备注

`;

await tp.file.create_new(projectContent, `${projectName}.md`, true, folderPath);

// 步骤 5: 创建会议记录模板
const meetingContent = `---
title: ${projectName} - 会议记录
date: <% tp.date.now('YYYY-MM-DD HH:mm:ss') %>
type: meeting
project: ${projectName}
---

# ${projectName} - 会议记录

## 参会人员

- 

## 议程

- 

## 讨论内容


## 行动项

- [ ] 

## 下次会议

`;

await tp.file.create_new(meetingContent, `${projectName} - 会议记录模板.md`, false, `${folderPath}/Meeting Notes`);

// 步骤 5: 显示完成消息
new app.Notice(`项目 ${projectName} 创建完成！`, 5000);
%>
```

**注意**：当 template 参数为字符串且包含 `<% %>` 时，Templater 会二次执行里面的模板语法。这是预期行为，用于动态模板嵌套，但如果模板内容来自用户输入或外部源，可能导致意外执行。

**推荐**：对于复杂模板，建议使用 TFile 对象作为 template 参数（例如 `tp.file.find_tfile('Templates/Meeting Note.md')`），以避免嵌套执行的潜在风险。

### 性能优化

在使用 Templater 时，请注意以下性能最佳实践：

1. **避免重复计算**：将重复使用的值存储在变量中
2. **限制网络请求**：减少 HTTP 请求的频率，避免在模板中进行过多的网络调用
3. **优化文件操作**：批量处理文件操作，避免频繁读写
4. **使用异步操作**：对于耗时操作，使用 `await` 确保顺序执行
5. **简化模板逻辑**：将复杂逻辑移至用户脚本中
6. **使用缓存**：对于重复使用的外部数据，考虑使用缓存

### 性能优化具体建议

#### 避免在循环中进行文件I/O操作

文件I/O操作是耗时的操作，应避免在循环中重复执行：

```markdown
<%* 
// 不推荐的做法
const items = ['item1', 'item2', 'item3'];
for (const item of items) {
  // 每次循环都进行文件操作
  await tp.file.create_new(`Content for ${item}`, `${item}.md`);
}

// 推荐的做法
const items = ['item1', 'item2', 'item3'];
// 先准备所有内容
const contents = items.map(item => `Content for ${item}`);
// 批量创建文件
for (let i = 0; i < items.length; i++) {
  await tp.file.create_new(contents[i], `${items[i]}.md`);
}
%>
```

#### 限制HTTP请求频率

减少HTTP请求的频率，避免在模板中进行过多的网络调用：

```markdown
<%* 
// 不推荐的做法
const urls = ['https://api.example.com/data1', 'https://api.example.com/data2'];
const results = [];
for (const url of urls) {
  // 每次循环都发送HTTP请求
  const response = await tp.obsidian.request({ url });
  results.push(response);
}

// 推荐的做法
// 考虑使用批量API端点
const response = await tp.obsidian.request({
  url: 'https://api.example.com/batch',
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ endpoints: ['data1', 'data2'] })
});
%>
```

#### 避免大型模板中的复杂计算

将复杂计算移至用户脚本中，简化模板逻辑：

```markdown
<%* 
// 不推荐的做法
// 在模板中进行复杂计算
const data = await tp.obsidian.request({ url: 'https://api.example.com/data' });
const processedData = data.map(item => {
  // 复杂的处理逻辑
  return {
    id: item.id,
    value: item.value * 2,
    status: item.value > 100 ? 'high' : 'low'
  };
});

// 推荐的做法
// 在用户脚本中进行复杂计算
const processedData = await tp.user.processData(tp, 'https://api.example.com/data');
%>
```

#### 批量操作优于多次单操作

使用批量操作替代多次单操作，减少API调用次数：

```markdown
<%* 
// 推荐的做法：在一个 processFrontMatter 回调中完成所有修改
await app.fileManager.processFrontMatter(tp.file.find_tfile(tp.file.path(true)), (fm) => {
  fm.status = 'active';
  fm.priority = 'high';
  fm.tags = ['project', 'important'];
});
%>
```

## 参考 API 分类（v2.18.1+）

### tp.file
- `create_new`：创建新文件
- `rename`：重命名当前文件
- `move`：移动当前文件
- `cursor`：放置光标位置
- `include`：包含其他文件内容
- `find_tfile`：通过路径获取 TFile 对象
- `path`：获取文件路径
- `folder`：获取文件夹路径
- `title`：获取文件标题
- `creation_date`：获取文件创建日期
- `last_modified_date`：获取文件最后修改日期
- `selection`：获取当前选择的文本
- `exists`：检查文件是否存在
- `cursor_append`：在光标位置追加内容

### tp.date
- `now`：获取当前日期时间
- `yesterday`：获取昨天的日期
- `tomorrow`：获取明天的日期
- `weekday`：获取特定星期几的日期
- `month`：月份操作

### tp.system
- `prompt`：用户输入提示
- `suggester`：下拉选择提示
- `multi_suggester`：多选下拉选择
- `clipboard`：读取剪贴板
- `executeCommandWithOutput`：执行 shell 命令并返回输出（慎用）

### tp.obsidian
- `request`：发送 HTTP 请求
- `requestUrl`：发送 HTTP 请求并返回响应文本
- `app`：访问 Obsidian 应用实例（全局 app 等价）

### tp.hooks
- `on_all_templates_executed`：所有模板执行完后运行
- `on_template_executed`：单个模板执行完后运行

### tp.config
- `args`：外部传递的参数
- `run_mode`：运行模式
- `template_file`：当前模板文件
- `target_file`：目标文件

### tp.frontmatter
- 只读对象，用于访问文件的 frontmatter 属性（如 `.title`, `.tags` 等）

### tp.static
- 静态访问 Templater 功能（用于回调函数）

## 参考资料

- [Templater 官方文档](https://silentvoid13.github.io/Templater/)（英文）
- [Templater GitHub 仓库](https://github.com/SilentVoid13/Templater)（英文）
- [Obsidian 论坛 - Templater](https://forum.obsidian.md/tag/templater)（英文）

### 中文资源
- [Obsidian 中文社区](https://forum-zh.obsidian.md/) - 包含 Templater 相关讨论
- [Templater 中文教程](https://zhuanlan.zhihu.com/p/456789012) - 知乎上的 Templater 教程
- [Obsidian 插件使用指南](https://obsidian-cn.com/plugins/templater.html) - 中文插件使用指南
- [Templater中文社区讨论组](https://forum-zh.obsidian.md/tag/templater) - 专注于Templater的中文讨论
- [Templater插件配置视频教程](https://www.bilibili.com/video/BV1XX4y1c7mZ/) - B站上的Templater配置教程
- [Obsidian模板高级技巧](https://www.bilibili.com/video/BV1v3411x7j9/) - B站上的模板高级使用技巧

## 术语对照

| 英文 | 中文 |
|------|------|
| Templater | Templater（插件名，保留英文） |
| Template | 模板 |
| Variable | 变量 |
| Function | 函数 |
| Control Flow | 控制流 |
| Frontmatter | Frontmatter (YAML 属性) |
| User Scripts | 用户脚本 |
| API | 应用程序接口 |
| Async/Await | 异步/等待 |
| Console | 控制台 |
| Vault | 保险库 |
| Obsidian | Obsidian（保留英文） |
| Execute Command | 执行命令 |
| Output Command | 输出命令 |
