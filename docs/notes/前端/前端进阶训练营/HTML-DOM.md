# HTML DOM

**文件物件模型（Document Object Model，DOM）**是HTML，XML和SVG文件的程序介面。它提供了一个文件（树）的结构化表示法，并定义让程序可以访问并更改文件架构，样式和内容DOM的提供了文件以拥有属性与函式的中断与对象组成的结构化表示。例程也可以附加事件处理程序，一旦触发事件就会执行处理程序。语言连结在一起。

虽然常常使用JavaScript来访问DOM，但它本身并不是JavaScript语言的一部分，而且它也可以被其他语言访问（虽然不太常见就是了）。

## 节点类型

DOM 的最小组成单位叫做节点（node）。文档的树形结构（DOM 树），就是由各种不同类型的节点组成。每个节点可以看作是文档树的一片叶子。

节点的类型有七种。

- `Document`：整个文档树的顶层节点
- `DocumentType`：doctype标签（比如\<!DOCTYPE html\>）
- `Element`：网页的各种HTML标签（比如\<body\>、\<a\>等）
- `Attribute`：网页元素的属性（比如class="right"）
- `Text`：标签之间或标签包含的文本
- `Comment`：注释
- `DocumentFragment`：文档的片段

浏览器提供一个原生的节点对象`Node`，上面这七种节点都继承了`Node`，因此具有一些共同的属性和方法。

**如何确定节点类型**

`Node`有一个属性`nodeType`表示`Node`的类型，不同节点的`nodeType`属性值和对应的常量如下

- 文档节点（document）：9，对应常量`Node.DOCUMENT_NODE`
- 元素节点（element）：1，对应常量`Node.ELEMENT_NODE`
- 属性节点（attr）：2，对应常量`Node.ATTRIBUTE_NODE`
- 文本节点（text）：3，对应常量`Node.TEXT_NODE`
- 文档片断节点（DocumentFragment）：11，对应常量`Node.DOCUMENT_FRAGMENT_NODE`
- 文档类型节点（DocumentType）：10，对应常量`Node.DOCUMENT_TYPE_NODE`
- 注释节点（Comment）：8，对应常量`Node.COMMENT_NODE`

确定节点类型时，使用`nodeType`属性是常用方法。

```javascript
var node = document.documentElement.firstChild;
if (node.nodeType === Node.ELEMENT_NODE) {
  console.log('该节点是元素节点');
}
```

## 创建元素

### createElement

`document.createElement`方法用来生成元素节点，并返回该节点。

```javascript
var div = document.createElement('div')
```

使用`createElement`要注意：通过`createElement`创建的元素并不属于`html`文档，它只是创建出来，并未添加到`html`文档中，要调用`appendChild`或`insertBefore`等方法将其添加到HTML文档树中。

### createTextNode

createTextNode用来创建一个文本节点，用法如下

```javascript
var newDiv = document.createElement('div');
var newContent = document.createTextNode('Hello');
newDiv.appendChild(newContent);
// <div>Hello</div>
```

上面代码新建一个`div`节点和一个文本节点，然后将文本节点插入`div`节点。

这个方法可以确保返回的节点，被浏览器当作文本渲染，而不是当作 `HTML` 代码渲染。因此，可以用来展示用户的输入

```javascript
var div = document.createElement('div');
div.appendChild(document.createTextNode('<span>Foo & bar</span>'));
console.log(div.innerHTML)
// &lt;span&gt;Foo &amp; bar&lt;/span&gt;
// <span>Foo & bar</span>
```

### insertAdjacentElement

`Element.insertAdjacentElement`方法在相对于当前元素的指定位置，插入一个新的节点。该方法返回被插入的节点，如果插入失败，返回`null`。

```javascript
element.insertAdjacentElement(position, element);
```

`Element.insertAdjacentElement`方法一共可以接受两个参数，第一个参数是一个字符串，表示插入的位置，第二个参数是将要插入的节点。第一个参数只可以取如下的值。

- `beforebegin`：当前元素之前
- `afterbegin`：当前元素内部的第一个子节点前面
- `beforeend`：当前元素内部的最后一个子节点后面
- `afterend`：当前元素之后

```javascript
// HTML 代码：<body><div>some text</div></body>
var body = document.querySelector('body')
var p1 = document.createElement('p')
body.insertAdjacentElement('afterbegin', p1)
// 执行代码之后
// <body><p></p><div>some text</div></body>
```

### insertAdjacentHTML, insertAdjacentText

`Element.insertAdjacentHTML`方法用于将一个 `HTML` 字符串，解析生成 `DOM` 结构，插入相对于当前节点的指定位置。

`element.insertAdjacentHTML(position, text);`
该方法接受两个参数，第一个是一个表示指定位置的字符串，第二个是待解析的 `HTML` 字符串。`position`参数的值与 `insertAdjacentElement`的 `position` 取值相同

```javascript
// HTML 代码：<div id="one">one</div>
var d1 = document.getElementById('one');
d1.insertAdjacentHTML('afterend', '<div id="two">two</div>');
// 执行后的 HTML 代码：
// <div id="one">one</div><div id="two">two</div>
```

该方法只是在现有的 `DOM` 结构里面插入节点，这使得它的执行速度比`innerHTML`方法快得多。

注意，该方法不会转义 `HTML` 字符串，这导致它不能用来插入用户输入的内容，否则会有安全风险。

`Element.insertAdjacentText`方法在相对于当前节点的指定位置，插入一个文本节点，用法与`Element.insertAdjacentHTML`方法完全一致。

```javascript
// HTML 代码：<div id="one">one</div>
var d1 = document.getElementById('one');
d1.insertAdjacentText('afterend', 'two');
// 执行后的 HTML 代码：
// <div id="one">one</div>two
```

## 修改元素

### appendChild

`appendChild()`方法接受一个节点对象作为参数，将其作为最后一个子节点，插入当前节点。该方法的返回值就是插入文档的子节点。

```javascript
var p = document.createElement('p');
document.body.appendChild(p);
```

### insertBefore

`insertBefore`方法用于将某个节点插入父节点内部的指定位置。`insertBefore`方法接受两个参数，第一个参数是所要插入的节点`newNode`，第二个参数是父节点`parentNode`内部的一个子节点`referenceNode`。`newNode`将插在`referenceNode`这个子节点的前面。返回值是插入的新节点`newNode`。

```javascript
var p = document.createElement('p');
document.body.insertBefore(p, document.body.firstChild);
// 上面代码中，新建一个<p>节点，插在document.body.firstChild的前面，也就是成为document.body的第一个子节点。
```

### removeChild

`removeChild`方法接受一个子节点作为参数，用于从当前节点移除该子节点。返回值是移除的子节点。

```javascript
var divA = document.getElementById('A');
divA.parentNode.removeChild(divA);
```

### replaceChild

`replaceChild`方法用于将一个新的节点，替换当前节点的某一个子节点。`replaceChild`方法接受两个参数，第一个参数`newChild`是用来替换的新节点，第二个参数`oldChild`是将要替换走的子节点。返回值是替换走的那个节点`oldChild`

```javascript
var divA = document.getElementById('divA');
var newSpan = document.createElement('span');
newSpan.textContent = 'Hello World!';
divA.parentNode.replaceChild(newSpan, divA);
// 将divA 替换成 newSpan
``` 

## 查询节点

### 获取单个节点

- **document.getElementById**

根据元素id返回元素，返回值是Element类型，如果不存在该元素，则返回null。
使用这个接口有几点要注意：

（1）元素的Id是大小写敏感的，一定要写对元素的id

（2）HTML文档中可能存在多个id相同的元素，则返回第一个元素

（3）只从文档中进行搜索元素，如果创建了一个元素并指定id，但并没有添加到文档中，则这个元素是不会被查找到的

```javascript
// HTML代码为
// <span id="myspan">Hello</span>
var span = document.getElementById('myspan');
span.id // "myspan"
span.tagName // "SPAN"
```

- **document.querySelector**

`Element.querySelector`方法接受 `CSS` 选择器作为参数，返回父元素的第一个匹配的子元素。如果没有找到匹配的子元素，就返回null。

```javascript
// 查找元素使用 document.querySelector() 函数
// 这个函数的参数是一个选择器(和 CSS 选择器一样)
// 选择器语法和 CSS 选择器一样, 现在只用 3 个基础选择器
// 元素选择器
var body = document.querySelector('body')
// class 选择器, 用的是 .类名
var form = document.querySelector('.login-form')
// id 选择器, 用的是   #id
var loginButton = document.querySelector('#id-button-login')
// log 出来看看

// 选择多个元素使用函数 querySelectorAll
var buttons = document.querySelectorAll('.radio-button')
// 还可以接受任何复杂的 CSS 选择器
document.body.querySelector("style[type='text/css'], style:not([type])");

// 查找到的元素还可以继续用 querySelector
var ul = document.querySelector('.ul')
ul.querySelector('li')
```

### 获取多个节点

- `document.getElementsByTagName`
- `document.getElementsByClassName`
- `document.getElementsByName`
- `document.querySelectorAll`

### 获取父节点

`parentElement` 和 `parentNode`

### 获取所有的后代节点

`children`属性返回一个`HTMLCollection`实例，成员是当前节点的所有元素子节点。

`childNodes`属性返回一个类似数组的对象（`NodeList`集合），成员包括当前节点的所有子节点，注意，除了元素节点，`childNodes`属性的返回值还包括文本节点和注释节点。

`children` 和 `childNodes` 最大的区别就是：`children` 不会把空白节点算进去。

### 获取兄弟节点

- previousSibling，nextSibling，previousElementSibling，nextElementSibling
- 空白节点的坑，previousSibling，nextSibling会把空白节点算进去

## 操作 CSS

### style

```javascript
// 在单个语句中设置多个样式
elt.style.cssText = "color: blue; border: 1px solid black"; 
// 或者
elt.setAttribute("style", "color:red; border: 1px solid blue;");

// 设置特定样式，同时保持其他内联样式值不变
elt.style.color = "blue";
```

### classList

```javascript
var element = document.querySelector('.active')
if (element != null) {
     // 使用 classList 可以访问一个元素的所有 class
     // remove 可以删除一个 class
     element.classList.remove("active")
}

element.classList.add('active') // 添加 active样式
element.classList.contains('active') //判断是否包含 active 样式
element.classList.toogle('active') // 如果存在 active 样式就删除，否则就添加
```

## 获取元素的位置

获取 DOM 元素相对于文档的位置，可以直接使用 `offsetTop`
获取 DOM 元素相对于视口的位置，可以使用 `getBoundingClientRect()`
获取 SVG 元素或行内元素的 CSS 盒子（比如用来做文本高亮时），可以使用 `getClientRects()`；
获取绝对定位元素、伪元素的渲染后 CSS 属性，可以使用 `getComputedStyle()`

[https://harttle.land/2018/04/22/get-dom-layout.html](https://harttle.land/2018/04/22/get-dom-layout.html)

## 获取网页的总宽高

```javascript
document.body.clientWidth
document.body.clientHeight
```

## 获取视口（浏览器可见区域）的宽高

```javascript
document.documentElement.clientWidth
document.documentElement.clientHeight
```