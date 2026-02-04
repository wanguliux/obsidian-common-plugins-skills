---
name: obsidian-excalidraw
description: 生成符合当前 Excalidraw JSON Schema 的白板文件（.excalidraw），支持形状、智能连线、文本绑定、Obsidian 内部链接。兼容 Obsidian Excalidraw 插件 2.0+，用于绘制架构图、流程图、思维导图。
---

## 核心规范（基于 Excalidraw v0.17+ / Obsidian 插件最新版）

### 文件格式标准
Excalidraw 文件为 JSON 格式，扩展名 `.excalidraw`。**必须**包含以下顶级字段：

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [],
  "appState": {
    "gridSize": 20,
    "viewBackgroundColor": "#ffffff",
    "theme": "light",
    "currentItemFontFamily": 1,
    "exportBackground": true,
    "viewModeEnabled": false
  },
  "files": {}
}
```

**警告**：如果elements中引用了图片（type: "image"），必须在files对象中提供对应的二进制数据（DataURL），否则打开文件会导致插件崩溃或无限加载。

**渲染顺序**：elements数组索引 0 为最底层，N 为最顶层

**gridSize**：强制对齐网格

**files**：仅在涉及图片嵌入时使用，否则为空对象

**关键约束：**

- **所有元素必须有唯一 id**：推荐使用 nanoid(10) 或随机字母数字 8-12 位（Excalidraw 内部使用类似），UUID 过长不必要。禁止重复。
- **所有元素必须有 seed**：正整数（建议 100000 ~ 999999999），用于确定手绘抖动的随机种子，保证渲染一致性。
- **每个元素需 version 与 versionNonce**：version 从 1 开始递增（每次修改+1），versionNonce 为随机整数，用于同步冲突解决。
- **坐标系**：原点 (0,0) 左上角，X 右，Y 下，单位像素。**强烈建议 Math.round() 所有 x/y/width/height（除 angle）**，避免亚像素模糊。
- **角度单位**：angle 使用**弧度**（0 ~ 2π）。
- **updated**：Unix 毫秒时间戳，建议使用当前时间（Date.now()），用于元素修改时间排序。生成多个元素时建议递增（如+1ms）避免同步冲突。Obsidian 同步主要依赖 versionNonce 和 version，而非 updated 的单调性。
- **渲染层级 (Z-Index)**：由 `elements` 数组顺序决定。
  - **建议顺序**：为避免遮挡，建议顺序为：背景容器 → 前景容器 → 箭头 → 文本。
  - **绑定关系**：绑定关系仅依赖 ID 引用，与数组顺序无关。箭头绑定依赖目标元素已存在 ID。



### 元素常见字段（大部分元素具有，类型特定字段见后文）

大部分元素对象包含以下字段，部分字段为类型特定或可选：

| **字段**          | **类型/值域** | **默认值**      | **严格约束说明**                                    |
| ----------------- | ------------- | --------------- | --------------------------------------------------- |
| `id`              | `string`      | -               | 全局唯一。                                          |
| `type`            | `enum`        | -               | 见下方类型详解。                                    |
| `x`, `y`          | `float`       | -               | 建议取整以防抗锯齿模糊。                            |
| `width`, `height` | `float`       | -               | 必须 > 0。                                          |
| `angle`           | `float`       | `0`             | **弧度制** (0 ~ 2π)。顺时针旋转。                   |
| `strokeColor`     | `hex`         | `"#1e1e1e"`     | 避免纯黑(#000)。                                    |
| `backgroundColor` | `hex`         | `"transparent"` | 支持 8位 Hex 透明度 (e.g. `#ff000080`)。            |
| `fillStyle`       | `enum`        | `"solid"`       | `"hachure"`, `"cross-hatch"`, `"solid"`, `"zigzag"` |
| `strokeWidth`     | `int`         | `1`             | 1(细), 2(标准), 4(粗)。                             |
| `strokeStyle`     | `enum`        | `"solid"`       | `"solid"`, `"dashed"`, `"dotted"`                   |
| `roughness`       | `int`         | `1`             | 0(几何), 1(手绘), 2(素描)。                         |
| `opacity`         | `int`         | `100`           | 0-100。                                             |
| `groupIds`        | `string[]`    | `[]`            | 属于哪个分组。用于把多个元素编组（类似 Ctrl+G），移动/删除时整体操作。**严格逻辑**：同一个组内的所有元素必须拥有完全相同的group ID字符串。生成时一般为空，除非需要批量移动的复合图形。复制元素时需同时复制group ID。 |
| `roundness`       | `object`      | `null`          | 见下方详细对象结构。                                |
| `seed`            | `int`         | `random`        | 正整数（建议 100000 ~ 999999999），用于确定手绘抖动的随机种子，保证渲染一致性。 |
| `version`         | `int`         | `1`             | 每次修改自增。生成时默认为 1。                      |
| `versionNonce`    | `int`         | `random`        | 0 ~ 2^32-1。                                        |
| `isDeleted`       | `bool`        | `false`         | 若为 true，Obsidian 将不渲染该元素。                |
| `boundElements`   | `array`       | `[]`            | **关键**：用于双向绑定。                            |
| `updated`         | `int`         | `time`          | 毫秒时间戳。                                        |
| `link`            | `string`      | `null`          | Obsidian 内部链接格式。                             |
| `locked`          | `bool`        | `false`         | 是否锁定。                                          |
| `frameId`         | `string|null` | `null`          | 若元素属于某个 Frame，则为此 Frame 的 ID；否则为 null。非必填，但若省略则默认 null。 |

**重要说明**：
- 部分字段仅适用于特定类型（如roundness仅限rectangle；points/startArrowhead仅限arrow/line）。text元素缺少fillStyle/roughness等字段。JSON中可省略非适用字段。
- **width/height计算时机**：width 和 height 必须反映元素的实际渲染包围盒，生成时应根据内容实时计算（尤其是文本和箭头），不能随意填值。
  - 文本：根据字体、字号、文本内容计算实际宽高
  - 箭头：根据points数组计算包围盒（width = max_x - min_x, height = max_y - min_y）

示例

```json
{
  "id": "unique-string",
  "type": "...",
  "x": 100,
  "y": 100,
  "width": 120,
  "height": 80,
  "angle": 0,
  "strokeColor": "#1e1e1e",
  "backgroundColor": "transparent",
  "fillStyle": "solid",
  "strokeWidth": 2,
  "strokeStyle": "solid",
  "roughness": 1,
  "opacity": 100,
  "groupIds": [],
  "frameId": null,
  "seed": 123456789,
  "version": 1,
  "versionNonce": 987654321,
  "isDeleted": false,
  "updated": 1735689600000,
  "link": null,
  "locked": false
}
```



### 元素类型详解与几何逻辑

**坐标计算公式（避免亚像素模糊）**：
- `x = Math.round(targetX)`
- `y = Math.round(targetY)`
- `width = Math.ceil(targetWidth)`
- `height = Math.ceil(targetHeight)`

**文本居中公式（严苛版）**：
```javascript
textElement.x = container.x + (container.width - textElement.width) / 2;
textElement.y = container.y + (container.height - textElement.height) / 2;
```

容器类 (Rectangle, Ellipse, Diamond)

**圆角规则 (`roundness`)**：
- **直角**：`roundness: null`
- **旧版比例圆角**：`roundness: { "type": 1 }` (PROPORTIONAL_RADIUS / LEGACY) - 旧版格式
- **自适应圆角 (推荐)**：`roundness: { "type": 2 }` (ADAPTIVE_RADIUS) - 当前标准建议值，最平滑，无需 value
- **固定像素圆角**：`roundness: { "type": 3, "value": 20 }` (FIXED_RADIUS) - 配合 value 使用，提供特定像素的圆角

**注意**：在2024年后的版本中，菱形(Diamond)也支持roundness对象。

#### 1. 矩形（Rectangle）

```json
{
  "type": "rectangle",
  "roundness": null
}
```

**圆角选项**：
- 直角：`roundness: null`
- 固定圆角：`roundness: { "type": 1, "value": 20 }`
- 自适应圆角：`roundness: { "type": 2 }`

#### 2. 菱形（Diamond）——独立类型（**非** rectangle + roundness）

```json
{
  "type": "diamond",
  "roundness": null
}
```

**菱形圆角选项**：
- 直角：`roundness: null`
- 自适应圆角：`roundness: { "type": 2 }`
- 固定圆角：`roundness: { "type": 3, "value": 10 }`

**注意**：在2024年后的版本中，菱形也支持roundness对象。

#### 3. 椭圆（Ellipse）

```json
{
  "type": "ellipse",
  "roundness": null
}
```

**注意**：x, y 为外接矩形左上角。

#### 4. 文本（Text）与容器绑定（**强制双向绑定**）
极高风险项，文本是渲染最容易出错的部分。必须严格遵循以下计算
```json
{
  "type": "text",
  "fontSize": 20,
  "fontFamily": 1,
  "lineHeight": 1.25,
  "text": "第一行\n第二行",
  "textAlign": "center",
  "verticalAlign": "middle",
  "containerId": "box-1",
  "baseline": 18,
  "width": 60,
  "height": 45
}
```

**文本元素说明**：
- `fontFamily`：1=Virgil（手绘）, 2=Helvetica（无衬线）, 3=Cascadia（代码字体）。Obsidian 插件通常不支持其他值。
- `lineHeight`：必须提供，否则多行文本对齐会乱。
- `text`：必须显式包含 `\n`，Excalidraw 不会自动换行。
- `verticalAlign`："top", "middle", "bottom" (仅当 containerId 不为 null 时生效)。
- `baseline`：计算值，对于多行文本，仍然是第一行的基线高度。
- `width`：必须预计算文本实际宽度。
- `height`：必须包含所有行高之和。

**绑定规则（严格）**：
1. 容器 boundElements 必须包含 { "type": "text", "id": "text-id" }
2. 文本 containerId 指向容器 id
3. 文本坐标（居中示例）：x = container.x + container.width/2 - text.width/2, y 同理
4. **创建顺序**：先容器，后文本
5. **删除**：容器 isDeleted: true 时必须同步删除绑定文本

**文本元素重要说明**：
- **lineHeight**：必须提供，否则多行文本对齐会乱。默认值 1.25。
- **baseline 动态性**：对于多行文本，baseline 仍然是第一行的基线高度，而不是整个块的中心。
- **对齐规则**：
  - 只有当 containerId 不为 null 时，verticalAlign ("middle", "top", "bottom") 才会生效。
  - 独立文本的 verticalAlign 通常被忽略，由 y 坐标决定。
- **宽高计算**：width 和 height 必须准确反映文本的实际渲染大小，height 必须包含所有行高之和。

Baseline 计算公式 (近似值)：
- fontFamily: 1 (Virgil): baseline ≈ fontSize * 0.84
- fontFamily: 2 (Helvetica): baseline ≈ fontSize * 0.76

**警告**：不要依赖硬编码的 baseline 近似值！实际场景中（多行文本、verticalAlign 不同、lineHeight 变化）都会导致明显偏移。

**强烈建议**：
- Baseline 是首行基线到文本元素顶部的距离。
- **最佳实践**：
  1. 使用 Excalidraw Automate API 的 measureText() 或 getTextSize() 获取精确宽高和 baseline
  2. 或在 Obsidian 中先手动创建好文本框 → 导出 JSON → 参考实际值
  3. 作为临时 fallback，可使用 fontSize * 0.78 ~ 0.85（Virgil）区间，优先偏小值（偏上一点在视觉上通常更可接受）
- 若手动生成，可按以下规则：
  - 单行居中：x = container.x + container.width/2 - text.width/2, y 同理

#### 5. 箭头与连线 (Arrow / Line)
箭头坐标是相对坐标体系。

```json
{
  "type": "arrow",
  "x": 100,
  "y": 100,
  "width": 150,
  "height": 50,
  "angle": 0,
  "points": [
    [0, 0],
    [150, 0]
  ],
  "lastCommittedPoint": [150, 0],
  "startArrowhead": null,
  "endArrowhead": "triangle",
  "startBinding": {
    "elementId": "rect-1",
    "focus": 0,
    "gap": 8
  },
  "endBinding": {
    "elementId": "rect-2",
    "focus": 0,
    "gap": 8
  },
  "strokeSharpness": "round"
}
```

**箭头元素说明**：
- `type`："arrow" 或 "line"（直线，无默认箭头样式）。
- `x`, `y`：起点基准坐标。
- `width`, `height`：由points计算出的包围盒（必须正确）。
- `points`：
  - 起点永远是[0,0]，否则Obsidian渲染时箭头会发生不可预知的偏移。
  - 后续点为相对偏移，绝对坐标 = x+偏移值, y+偏移值。
  - 弯折示例：[[0,0], [50,50], [150,0]]。
- `lastCommittedPoint`：必须等于points最后一项，始终与points数组的最后一个元素保持一致。
- `startArrowhead`：有效值："arrow", "triangle", "dot", "bar", "diamond", "circle", "triangle_filled", "arrow_filled", null。
- `endArrowhead`：默认值：null（无箭头）；常用："triangle"（尖头）、"arrow"（经典箭头）。
- `focus`：-1 ~ 1，0=中心最稳定。
- `gap`：必填，推荐4~12，避免默认间距过大。
- `strokeSharpness`："round"（圆角折线/贝塞尔感）、"sharp"（直折角）。
- 其他通用字段：根据需要添加。

**说明**：
- 双向绑定必须同时设置箭头的start/endBinding + 目标元素的boundElements: [{id: "arrow-id", type: "arrow"}]。
- 箭头头可能值随Excalidraw更新而增加（2025年后新增triangle、diamond等），null等价于"none"。
- **strokeSharpness**：控制箭头折线的样式
  - "round" → 贝塞尔曲线/圆角折线
  - "sharp" → 直角折线
- **最新特性**：最新版Excalidraw支持智能折线，通过strokeSharpness和points数组的合理设置，可以实现更自然的连线效果。

**常用推荐组合**：
- 流程图：startArrowhead: null, endArrowhead: "triangle" 或 "arrow"
- 双向箭头：startArrowhead: "triangle", endArrowhead: "triangle"
- 无箭头连线：都设为 null（或使用 type: "line"）

### 其他常见元素类型

- **line**：与arrow相同结构，但默认startArrowhead/endArrowhead为null。
- **frame**：type: "frame"，额外字段 "name": "框架标题"（显示在框架顶部），用于分组大块区域。
- **image**：type: "image"，需 "fileId": "文件ID"（对应files对象key）、"scale": [1,1]。
  **严格要求**：使用image元素时，必须在顶级files对象中提供对应fileId的DataURL数据，格式如下：
  ```json
"files": {
  "fileId": {
    "type": "image/png",
    "data": "data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVR42mNkYPhfDwAChwGA60e6kgAAAABJRU5ErkJggg=="
  }
}
```

**图片文件说明**：
- `type`：必须准确，与DataURL的mimeType一致。
- `data`：Base64编码的图片数据，格式为data:image/{format};base64,{data}。
  **注意**："type"字段的mimeType必须与DataURL中的mimeType完全一致，否则可能导致图片无法加载。
- **freedraw**：type: "freedraw"，points: [[x,y], ...]（绝对坐标），用于手绘路径。

**说明**：以上类型覆盖90%架构图/流程图/思维导图场景，其余类型可参考Excalidraw导出JSON。

#### 6. 双向绑定握手协议 (The Binding Handshake)
**核心原则**：在 Excalidraw 中，绑定必须是双向声明的。任何一方缺失都会导致“假绑定”（视觉接触但拖动分离）。

**双向绑定规则**：
- **容器 -> 成员**：容器（Rectangle/Ellipse/Diamond/Frame）的 boundElements 数组内必须包含对象：{"id": "member-id", "type": "text" | "arrow"}。
- **成员 -> 容器**：
  - 文本需设置 containerId: "container-id"。
  - 箭头需设置 startBinding 或 endBinding。

**严苛警告**：
- 如果容器被 isDeleted: true，但其绑定的文本 isDeleted: false，Obsidian 会渲染一个漂浮在原地的文本。必须级联删除。
- 箭头指向容器时，容器的 boundElements 必须包含箭头的 ID，否则箭头不会跟随容器移动。

##### 场景 A：文本放入矩形 (Text inside Rectangle)

1. **矩形侧**：`boundElements` 必须包含文本 ID。

   JSON

   ```
   "id": "rect-1",
   "boundElements": [ { "id": "text-1", "type": "text" } ]
   ```

2. **文本侧**：`containerId` 必须指向矩形 ID。

   JSON

   ```
   "id": "text-1",
   "containerId": "rect-1"
   ```

3. **坐标对齐**：文本的 `x, y` 必须经过计算，使其位于矩形中心（或指定的 align 位置）。

##### 场景 B：箭头连接矩形 (Arrow connects Rectangles)

1. **箭头侧**：声明 `startBinding` 或 `endBinding`。

   JSON

   ```
   "startBinding": {
     "elementId": "rect-1",
     "focus": 0,
     "gap": 8
   }
   ```

2. **矩形侧**：`boundElements` 必须包含箭头 ID。

   JSON

   ```
   "boundElements": [
     { "id": "text-1", "type": "text" },
     { "id": "arrow-1", "type": "arrow" }
   ]
   ```

##### 推荐文件格式

**优先使用 `.excalidraw` 文件格式**，这是纯JSON格式，兼容性最好，不易出错：

- **文件扩展名**：`.excalidraw`
- **文件内容**：纯JSON格式，无任何额外内容
- **格式要求**：严格遵循以下结构

```json
{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [],
  "appState": {},
  "files": {}
}
```

**重要说明**：
- **避免使用 `.excalidraw.md` 格式**，因为AI生成compressed-json结构容易出错
- **纯JSON格式优势**：更简单、更可靠，不易受编辑器或IDE干扰
- **兼容性**：`.excalidraw` 是Excalidraw的原生格式，兼容性最好

**生成规范**：
- 生成的文件必须是纯JSON格式，无任何额外内容
- 禁止在文件中包含编辑器标记、行号引用或代码块标记
- 确保文件编码为UTF-8，无BOM
- 验证生成的JSON格式正确，可使用JSON验证工具检查


------

## 5. Obsidian 深度集成规范

### 内部链接 (Internal Links)

- **格式**：支持多种 Obsidian WikiLink 格式：
  - `"link": "[[Note]]"`
  - `"link": "[[Note|显示别名]]"`
  - `"link": "[[Note#Heading]]"`
  - `"link": "[[Note#^block-id]]"`
  - `"link": "[[Folder/Note#Section|^alias]]"`
  - `"link": "exec:脚本名"`（用于执行 Obsidian 脚本）
- **行为**：点击元素时，Obsidian 会跳转到对应笔记或执行指定脚本。
- **注意**：不需要前缀 `file://` 或 `obsidian://`，直接写 WikiLink。
- **限制**：link 字段的跳转功能仅在 .excalidraw.md 文件中生效。纯 JSON 文件（.excalidraw）中的 link 不会被 Obsidian 解析为可点击链接。

### Obsidian 特有元数据

- **customData**：Obsidian Excalidraw 插件会在元素中存储 customData。虽然生成时可以省略，但若要支持某些高级功能（如特定脚本触发器），应了解其存在。
  ```json
  "customData": {
    "someKey": "someValue"
  }
  ```
- **内部链接格式**：除了标准的 WikiLink 格式外，若要在元素上添加类似“点击按钮执行脚本”的功能，Obsidian 插件支持在 link 字段中使用 exec:脚本名。

### 颜色规范（Excalidraw 官方调色板，禁止 #000/#fff）

- 主文本/线条：#1e1e1e（亮色主题）/ #2e2e2e（暗色主题）
- 蓝：#1971c2 / #a5d8ff
- 红：#e03131 / #ffec99
- 绿：#2f9e44 / #b2f2bb
- 灰：#495057 / #e9ecef
- 黄：#f08c00 / #fff3bf
- 紫：#7048e8 / #e5dbff

**建议**：暗色主题下优先使用 #2e2e2e 作为 strokeColor，提高可读性。

**暗色主题推荐搭配**：
- strokeColor: "#e0e0e0" 或 "#d0d0d0"（浅灰，提高对比度）
- backgroundColor: "#2d2d2d" 或 "#1e1e1e"（深色底）
- 文本颜色："#e0e0e0"

### Obsidian 集成规范

- **推荐文件格式**：Diagram.excalidraw（纯 JSON）

  **优势**：
  - 原生格式，兼容性最好
  - 纯JSON结构，不易出错
  - 生成和解析速度快
  - 不受编辑器或IDE干扰

- **嵌入语法**

  Markdown

  ```
  ![[Diagram.excalidraw]]           <!-- 可编辑画布 -->
  ![[Diagram.excalidraw|600]]       <!-- 指定宽度 -->
  ![[Diagram.excalidraw|600x400]]   <!-- 宽高 -->
  ```

**重要提示**：
- 避免使用 `.excalidraw.md` 格式，因为AI生成compressed-json结构容易出错
- 优先使用纯 `.excalidraw` 文件，以获得最佳的兼容性和可靠性

**内部链接**（link 字段） 支持 [[Note]], [[Note#heading]], [[Note|alias]], [[Note#^block]]

### 常见错误与调试（扩展版）

- ID 重复 → 崩溃
- 文本 containerId 指向不存在 ID 或未在 boundElements 声明 → 文本错位/不随动
- 文本溢出：是否根据字符数估算了 text width/height？如果文本过长，是否在 JSON 字符串中插入了 `\n`？
- 箭头 points 非 [0,0] 开头 → 位置偏移
- updated 时间戳重复或倒序 → 同步丢失变更
- 缺少 seed/version/versionNonce → 同步异常
- roundness.type = 3 → 无效（旧版遗留，已废弃）
- link 使用 javascript: → XSS 风险，禁止
- 浮点坐标过多 → 建议整数以避免微小错位
- Ghost Binding（幽灵绑定）：箭头声明了绑定 `rect-1`，但 `rect-1` 的 `boundElements` 里没有箭头 ID？（修正：必须双向添加）。
- 坐标非整数：是否出现了 `x: 100.3333333`？（修正：除 `angle` 外，尽量 `Math.round()` 坐标，提升渲染性能）。
- 颜色对比度：是否在白色背景上使用了白色线条？（修正：确保 `strokeColor` 有足够对比度）。

### 最佳实践

- 水平节点间距 180–250px，垂直 120–180px

- 字体：标题 28–36px，正文 20px，注释 16px

- fillStyle: "solid"（推荐）、"hachure"（手绘斜线）、"cross-hatch"

- roughness: 1（标准），0（锐利），2（极粗）

- 生成时：先容器 → 文本 → 箭头（保证绑定顺序）

- 使用当前时间戳 + 递增小偏移作为 updated

  

### 箭头绑定高级技巧

**双向绑定必做**：

- 箭头：设置 startBinding / endBinding（elementId + focus + gap）。
- 容器：boundElements 数组中添加 { "type": "arrow", "id": "..." }（支持多个箭头）。
- 缺少任一侧 → 箭头不智能吸附，容器移动时箭头位置固定。

**focus 参数计算建议**（-1 ~ 1）：

- 0：从容器中心吸附（最稳定，推荐默认）。
- 正值（0~1）：偏向右/下（水平箭头偏下，垂直偏右）。
- 负值（-1~0）：偏向左/上。
- 高级：如果容器宽高不同，focus 可微调到 ±0.2~±0.5，避免箭头从角落拉出。极端 ±1 可能导致箭头贴边但视觉不佳。

**gap 参数建议**：

- 0：箭头端点紧贴容器边缘（可能穿模）。

- 8~12：最佳视觉间隙（推荐值）。

- > 15：箭头太远，显得松散。

**points / width / height 计算**（生成时必须正确）：

- points 第一点永远 [0, 0]。
- 后续点为相对偏移（e.g. 水平箭头 [[0,0], [距离, 0]]）。
- width = max_x - min_x（points 中 x 差）。
- height = max_y - min_y。
- 简单直线箭头示例：points [[0,0], [150,0]] → width=150, height=0。
- 弯折箭头：points [[0,0], [50,50], [150,50]] → width=150, height=50。

**创建顺序**： 先创建所有容器 → 再创建箭头（绑定依赖容器已存在 ID）。 文本绑定同理：容器先，文本后。

**常见绑定失败调试**（添加到“常见错误与调试”）：

- 箭头拖动容器不跟随 → 检查容器 boundElements 是否缺少箭头 ID。
- 箭头位置跳跃/不吸附 → focus/gap 极端，或 points width/height 计算错。
- 箭头绑定后重绘异常 → updated 时间戳未递增，或 versionNonce 重复。
- Obsidian 渲染箭头不绑 → 嵌入 .excalidraw.md 测试，确认 JSON 双向完整。

**版本兼容性**：基于 Excalidraw JSON v2 + Obsidian Excalidraw 插件 2.0+（2025 年最新）。

## AI 生成强制约束 (防止解析错误)

当使用 AI 生成 Excalidraw JSON 文件时，必须严格遵守以下约束，以防止解析错误：

1. **禁止注释**：生成的 JSON 代码块中严禁出现 `//` 或 `/* */` 注释。
2. **禁止虚假占位**：严禁出现 `...` 或 `(其余内容同上)`，必须输出完整的、合法的 JSON 对象。
3. **负号规范**：所有数值字段（x, y, width, height）必须为数字。禁止出现 `"x": -` 这种非法格式。负数必须带有前导数字（如 `-0.5` 而不是 `-.5`）。
4. **代码块隔离**：JSON 数据必须包裹在 ```json ... ``` 之间，且上方必须有完整的 YAML Frontmatter 区。

## 错误自查清单 (Final Check)

在生成或修改 Excalidraw 文件后，请检查以下项目，以避免常见错误：

- **ID 冲突**：是否使用了短随机 ID（如 nanoid）且在当前 elements 数组内唯一？
- **顺序 (Z-Index)**：背景元素是否在数组前端？文本和箭头是否在后端？
- **Seed**：每个元素是否有唯一的 seed？（若所有元素 seed 相同，手绘抖动外观会完全一致，显得机械）。
- **VersionNonce**：是否为每个元素生成了随机整数？
- **Updated**：时间戳是否单调递增？
- **Arrow points**：箭头的第一个点是否为 [0,0]？
- **BoundElements 闭环**：箭头指向容器时，容器的 boundElements 里写了箭头的 ID 吗？（双向绑定）。
- **级联删除**：删除容器时，是否同时删除了其绑定的文本？
- **图片数据**：使用 image 元素时，是否在 files 对象中提供了对应的 DataURL？
- **坐标计算**：是否使用了 Math.round() 避免亚像素模糊？
- **文本对齐**：多行文本是否提供了 lineHeight？baseline 是否为第一行的基线高度？

## 版本兼容性问题分析与解决方案

### 问题原因

1. **JSON 语法错误**：
   - 原始 SKILL.md 文件中存在 JSON 内部注释和警告文本，这违反了 JSON 语法规范
   - 生成的文件包含这些非标准内容会导致解析失败

2. **版本差异**：
   - SKILL.md 基于较新版本的 Excalidraw 规范
   - 某些特性可能在旧版本中不被支持

3. **绑定关系复杂性**：
   - 不正确的双向绑定会导致文档无法正确加载
   - 缺少必要的绑定声明会造成"假绑定"现象

4. **必填字段缺失**：
   - 某些版本对必填字段要求更严格
   - 缺少关键字段会导致解析错误

### 解决方案

1. **严格遵循 JSON 规范**：
   - 生成的 JSON 文件中禁止包含注释
   - 确保所有 JSON 结构完整且格式正确
   - 使用验证工具检查 JSON 语法

2. **版本兼容策略**：
   - 对于 Obsidian Excalidraw 插件 2.0+ 版本，使用以下兼容特性：
     - 保持 `version: 2` 格式
     - 避免使用最新版特有的元素类型
     - 使用基础的绑定关系，避免复杂的高级特性

3. **简化绑定关系**：
   - 对于基本图形，可暂时省略复杂的双向绑定
   - 确保绑定声明的一致性和完整性
   - 按照"先容器后成员"的顺序创建元素

4. **必填字段检查**：
   - 确保每个元素都包含以下字段：
     - `id`：唯一标识符
     - `type`：元素类型
     - `x`, `y`：坐标位置
     - `width`, `height`：尺寸
     - `seed`：随机种子
     - `version`：版本号
     - `versionNonce`：版本随机数
     - `updated`：时间戳

5. **调试技巧**：
   - 从简单图形开始测试
   - 逐步添加复杂元素
   - 使用简化版本验证基本功能
   - 检查浏览器控制台的错误信息

### 推荐实践

1. **使用模板文件**：
   - 基于本 SKILL.md 生成的模板文件
   - 确保模板文件经过验证可以正常加载

2. **版本标记**：
   - 在生成的文件中添加版本信息
   - 记录使用的 Excalidraw 规范版本

3. **兼容性测试**：
   - 在不同版本的 Excalidraw 中测试生成的文件
   - 确保在目标版本中正常显示

4. **错误处理**：
   - 提供详细的错误信息
   - 指导用户如何修复常见问题

通过遵循以上解决方案，可以确保生成的 Excalidraw 文件在当前版本中正常显示，同时保持向前兼容性。