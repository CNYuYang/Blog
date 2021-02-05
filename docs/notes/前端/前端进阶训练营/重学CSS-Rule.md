# 重学CSS

## @Rule

### @charset

#### 概述

`@charset` CSS @规则  指定样式表中使用的字符编码。它必须是样式表中的第一个元素，而前面不得有任何字符。因为它不是一个嵌套语句，所以不能在@规则条件组中使用。如果有多个 `@charset` @规则被声明，只有第一个会被使用，而且不能在HTML元素或HTML页面的字符集相关 \<style\> 元素内的样式属性内使用。

此 @规则 在某些 CSS 属性中使用非 ASCII 字符时非常有用，例如 `content`。

在样式表中有多种方法去声明字符编码，浏览器会按照以下顺序尝试下边的方法（一旦找到就停止并得出结果）：

1. 文件的开头的 Unicode byte-order 字符值。
2. 由Content-Type：HTTP header 中的 charset 属性给出的值或用于提供样式表的协议中的等效值。
3. CSS @规则  @charset。
4. 使用参考文档定义的字符编码： `<link>` 元素的 charset 属性。 该方法在 HTML5 标准中已废除，无法使用。
5. 假设文档是 UTF-8。

#### 语法

```css
@charset "UTF-8";
@charset "iso-8859-15"; 
```

where:

**charset**

它是一个 \<string\> 表示字符编码被使用。它必须是在被 IANA-registry 声明过的 web-safe 字符编码中的一个, 还必须被双引号包围, 遵循一个空格字符 (U+0020)，并且立即以分号结束。 如果有多个相关的编码名字，只有被标记为 preferred  的那个才会被使用。

**语法格式**

```css
@charset "<charset>";
```

**例子**

```css
@charset "UTF-8";
@charset "utf-8"; /*大小写不敏感*/
/* 设置css的编码格式为Unicode UTF-8 */
@charset 'iso-8859-15'; /* 无效的, 使用了错误的引号 */
@charset 'UTF-8';       /* 无效的, 使用了错误的引号 */
@charset  "UTF-8";      /* 无效的, 多于一个空格 */
 @charset "UTF-8";      /* 无效的, 在at-rule之前多了一个空格 */
@charset UTF-8;         /* Invalid, without ' or ", the charset is not a CSS <string> */
```

### @import

#### 概述

`@import` CSS@规则，用于从其他样式表导入样式规则。这些规则必须先于所有其他类型的规则，`@charset` 规则除外; 因为它不是一个嵌套语句，`@import`不能在条件组的规则中使用。

因此，用户代理可以避免为不支持的媒体类型检索资源，作者可以指定依赖媒体的`@import`规则。这些条件导入在URI之后指定逗号分隔的媒体查询。在没有任何媒体查询的情况下，导入是无条件的。指定所有的媒体具有相同的效果。

#### 语法

```css
@import url;
@import url list-of-media-queries;
```

其中:

**url**

是一个表示要引入资源位置的 \<string\> 或者 \<uri\> 。 这个 URL 可以是绝对路径或者相对路径。 要注意的是这个 URL 不需要指明一个文件； 可以只指明包名，然后合适的文件会被自动选择 (e.g. chrome://communicator/skin/). 

**list-of-media-queries**

是一个逗号分隔的 媒体查询 条件列表，决定通过URL引入的 CSS 规则 在什么条件下应用。如果浏览器不支持列表中的任何一条媒体查询条件，就不会引入URL指明的CSS文件。

#### 示例

```css
@import url("fineprint.css") print;
@import url("bluish.css") projection, tv;
@import 'custom.css';
@import url("chrome://communicator/skin/");
@import "common.css" screen, projection;
@import url('landscape.css') screen and (orientation:landscape);
```

### @media

`@media` CSS @规则 可用于基于一个或多个 媒体查询 的结果来应用样式表的一部分。 使用它，您可以指定一个媒体查询和一个CSS块，当且仅当该媒体查询与正在使用其内容的设备匹配时，该CSS块才能应用于该文档。

#### 语法

`@media` 规则可置于您代码的顶层或位于其它任何@条件规则组内。

```css
/* At the top level of your code */
@media screen and (min-width: 900px) {
  article {
    padding: 1rem 3rem;
  }
}

/* Nested within another conditional at-rule */
@supports (display: flex) {
  @media screen and (min-width: 900px) {
    article {
      display: flex;
    }
  }
}
```

#### 媒体特性

媒体特性（Media features）描述了 user agent、输出设备，或是浏览环境的具体特征。媒体特性表达式是完全可选的，它负责测试这些特性或特征是否存在、值为多少。每条媒体特性表达式都必须用括号括起来。

|   名称                                                             |    简介                                                              |         备注                                                   |
| :----------------------------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`any-hover`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/any-hover) | 是否有任何可用的输入机制允许用户（将鼠标等）悬停在元素上？   | 在 Media Queries Level 4 中被添加。                          |
| [`any-pointer`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/any-pointer) | 可用的输入机制中是否有任何指针设备，如果有，它的精度如何？   | 在 Media Queries Level 4 中被添加。                          |
| [`aspect-ratio`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/aspect-ratio) | 视窗（viewport）的宽高比                                     |                                                              |
| [`color`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/color) | 输出设备每个像素的比特值，常见的有 8、16、32 位。如果设备不支持输出彩色，则该值为 0 |                                                              |
| [`color-gamut`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/color-gamut) | 用户代理和输出设备大致程度上支持的色域                       | 在 Media Queries Level 4 中被添加。                          |
| [`color-index`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/color-index) | 输出设备的颜色查询表（color lookup table）中的条目数量，如果设备不使用颜色查询表，则该值为 0 |                                                              |
| [`device-aspect-ratio`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/device-aspect-ratio) | 输出设备的宽高比                                             | 已在 Media Queries Level 4 中被弃用。                        |
| [`device-height`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/device-height) | 输出设备渲染表面（如屏幕）的高度                             | 已在 Media Queries Level 4 中被弃用。                        |
| [`device-width`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/device-width) | 输出设备渲染表面（如屏幕）的宽度                             | 已在 Media Queries Level 4 中被弃用。                        |
| [`display-mode`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/display-mode) | 应用程序的显示模式，如web app的manifest中的[`display`](https://developer.mozilla.org/zh-CN/docs/Web/Manifest#display) 成员所指定 | 在 [Web App Manifest spec](http://w3c.github.io/manifest/#the-display-mode-media-feature)被定义. |
| [`forced-colors`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/forced-colors) | 检测是user agent否限制调色板                                 | 在 Media Queries Level 5 中被添加。                          |
| [`grid`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/grid) | 输出设备使用网格屏幕还是点阵屏幕？                           |                                                              |
| [`height`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/height) | 视窗（viewport）的高度                                       |                                                              |
| [`hover`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/hover) | 主要输入模式是否允许用户在元素上悬停                         | 在 Media Queries Level 4 中被添加。                          |
| [`inverted-colors`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/inverted-colors) | user agent或者底层操作系统是否反转了颜色                     | 在 Media Queries Level 5 中被添加。                          |
| [`light-level`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/light-level) | 环境光亮度                                                   | 在 Media Queries Level 5 中被添加。                          |
| [`monochrome`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/monochrome) | 输出设备单色帧缓冲区中每个像素的位深度。如果设备并非黑白屏幕，则该值为 0 |                                                              |
| [`orientation`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/orientation) | 视窗（viewport）的旋转方向                                   |                                                              |
| [`overflow-block`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/overflow-block) | 输出设备如何处理沿块轴溢出视窗(viewport)的内容               | 在 Media Queries Level 4 中被添加。                          |
| [`overflow-inline`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/overflow-inline) | 沿内联轴溢出视窗(viewport)的内容是否可以滚动？               | 在 Media Queries Level 4 中被添加。                          |
| [`pointer`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/pointer) | 主要输入机制是一个指针设备吗？如果是，它的精度如何？         | 在 Media Queries Level 4 中被添加。                          |
| [`prefers-color-scheme`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/prefers-color-scheme) | 探测用户倾向于选择亮色还是暗色的配色方案                     | 在 Media Queries Level 5 中被添加。                          |
| [`prefers-contrast`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/prefers-contrast) | 探测用户是否有向系统要求提高或降低相近颜色之间的对比度       | 在 Media Queries Level 5 中被添加。                          |
| [`prefers-reduced-motion`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/prefers-reduced-motion) | 用户是否希望页面上出现更少的动态效果                         | 在 Media Queries Level 5 中被添加。                          |
| [`prefers-reduced-transparency`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/prefers-reduced-transparency) | 用户是否倾向于选择更低的透明度                               | 在 Media Queries Level 5 中被添加。                          |
| [`resolution`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/resolution) | 输出设备的像素密度（分辨率）                                 |                                                              |
| [`scan`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/scan) | 输出设备的扫描过程（适用于电视等）                           |                                                              |
| [`scripting`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/scripting) | 探测脚本（例如 JavaScript）是否可用                          | 在 Media Queries Level 5 中被添加。                          |
| [`update`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/update-frequency) | 输出设备更新内容的渲染结果的频率                             | 在 Media Queries Level 4 中被添加。                          |
| [`width`](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@media/width) | 视窗（viewport）的宽度，包括纵向滚动条的宽度                 |                                                              |


#### 示例

```css
@media print {
  body { font-size: 10pt; }
}

@media screen {
  body { font-size: 13px; }
}

@media screen, print {
  body { line-height: 1.2; }
}

@media only screen
  and (min-width: 320px)
  and (max-width: 480px)
  and (resolution: 150dpi) {
    body { line-height: 1.4; }
}
```

媒体查询第4级引入了一种新的范围语法，在测试接受范围的任何特性时允许更简洁的媒体查询，如下面的示例所示：

```css
@media (height > 600px) {
    body { line-height: 1.4; }
}

@media (400px <= width <= 700px) {
    body { line-height: 1.4; }
}
```

### @page

`@page` 规则用于在打印文档时修改某些CSS属性。你不能用@page规则来修改所有的CSS属性，而是只能修改margin,orphans,widow 和 page breaks of the document。对其他属性的修改是无效的。

```css
@page {
  margin: 1cm;
}

@page :first {
  margin: 2cm;
}
```

#### 语法

**size**

指定页面盒模型所在的容器的大小和方向。一般情况下，因为一个页面盒模型被渲染到一面纸张上，所以这个属性也指示了目标纸张的大小。

**marks**

向文档添加剪切标记和/或注册标记。

**bleed**

为页面框盒指定一个限制区域，超过这个区域的页面内容将被裁剪。

### other

- **@namespace**
- **@supports**
- **@document**
- **@font-face**
- **@keyframes**
- **@viewport**
- **counter-style**

## 其他

### Selector选择器

#### 类选择器（Class selectors）

通过设置元素的`class`属性，可以为元素指定类名。类名由开发者自己指定。 文档中的多个元素可以拥有同一个类名。

在写样式表时，类选择器是以英文句号（.）开头的。

#### ID选择器（ID selectors）

通过设置元素的 `id` 属性为该元素制定ID。ID名由开发者指定。每个ID在文档中必须是唯一的。

在写样式表时，ID选择器是以#开头的。

例：
下面的p标签同时具有 `class` 属性和`id`属性:

```html
<p class="key" id="principal">
```

**id**属性值`principal`必须在文档中是唯一的；但文档中的其他标签可以有和p相同的**class**属性值 `key`.

在一个CSS样式表中, 下面的规则将使所有class属性等于key的元素文字颜色呈现绿色。（这些元素不一定都是 `<p>` 元素。）

```css
.key {
  color: green;
}
```

下面的规则将使 **id** 等于 `principal` 的那个元素的文字变为粗体:

```css
#principal {
  font-weight: bolder;
}
```

如果多于一个规则指定了相同的属性值都应用到一个元素上，CSS规定拥有更高确定度的选择器优先级更高。ID选择器比类选择器更具确定度, 而类选择器比标签选择器（tag selector）更具确定度。

**更多细节**

你也可以将多个选择器组合起来构成更确定的选择器。

比如，选择器`.key` 选中所有`class`属性为 `key`的元素. 选择器 `p.key` 选中所有`class`属性为`key`的`<p>`元素。

除了`class` 和 `id`，你还可以用方括号的形式指定其他属性。比如，选择器 `[type='button']`选中所有 type 属性为 button 的元素。

如果样式中包含冲突的规则，且它们具有相同的确定度。那么，后出现的规则优先级高。

如果你遇到规则冲突，你可以增加其中一条的确定度或将之移到后面以使它具有更高优先级。

#### 伪类选择器（Pseudo-classes selectors）

CSS伪类（pseudo-class）是加在选择器后面的用来指定元素状态的关键字。比如，`:hover` 会在鼠标悬停在选中元素上时应用相应的样式。

伪类和伪元素（pseudo-elements）不仅可以让你为符合某种文档树结构的元素指定样式，还可以为符合某些外部条件的元素指定样式：浏览历史(比如是否访问过 (`:visited`)， 内容状态(如 `:checked` ), 鼠标位置 (如`:hover`). 

```css
selector:pseudo-class {
  property: value;
}
```

- :link
- :visited
- :active
- :hover
- :focus
- :first-child
- :nth-child
- :nth-last-child
- :nth-of-type
- :first-of-type
- :last-of-type
- :empty
- :target
- :checked
- :enabled
- :disabled

#### 基于关系的选择器

CSS还有多种基于元素关系的选择器。通过它们你可以更精确的选择元素。

常见的基于关系的选择器

| **选择器**      | **选择的元素**                                               |
| --------------- | ------------------------------------------------------------ |
| `A E`           | 元素A的任一后代元素E (后代节点指A的子节点，子节点的子节点，以此类推) |
| `A > E`         | 元素A的任一子元素E(也就是直系后代)                           |
| `E:first-child` | 任一是其父母结点的第一个子节点的元素E                        |
| `B + E`         | 元素B的任一下一个兄弟元素E                                   |
| `B ~ E`         | B元素后面的拥有共同父元素的兄弟元素E                         |

你可以任意组合以表达更复杂的关系。

你还可以使用星号（*）来表示”任意元素“。

例

一个HTML表格有`id`属性，但是它的行和单元格没有单独的id:

```html
<table id="data-table-1">
...
<tr>
<td>Prefix</td>
<td>0001</td>
<td>default</td>
</tr>
...
```

下面的规则使表格每行的第一个单元格字体为粗体，使第二个单元格使用等宽字体。这条规则只影响id为data-table-1的表格:

```css
#data-table-1 td:first-child {font-weight: bolder;}
#data-table-1 td:first-child + td {font-family: monospace;}
```

一般情况下，如果你提高了某个选择器的的确定度，你便提高它的优先级。

使用这个技巧，可以避免为大量标签指定 `class` 或 `id` 属性。CSS（引擎）会帮你做的。

在复杂设计中速度非常重要，避免使用复杂的依赖元素关系的规则可以使你的样式更有效率。

#### 伪元素

伪元素是一个附加至选择器末的关键词，允许你对被选择元素的特定部分修改样式。下例中的 `::first-line` 伪元素可改变段落首行文字的样式。

```css
/* 每一个 <p> 元素的第一行。 */
p::first-line {
  color: blue;
  text-transform: uppercase;
}
```