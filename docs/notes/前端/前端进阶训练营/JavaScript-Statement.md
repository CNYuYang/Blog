# JavaScript Statement

## block

**块语句**（或其他语言的**复合语句**）用于组合零个或多个语句。该块由一对大括号界定，可以是labelled：

### 语法

**块声明**

> { StatementList }

**标记块声明**

> LabelIdentifier: { StatementList }

**StatementList**

在块语句中分组的语句。

**LabelIdentifier**

用于视觉识别的可选label或break的目标。

### 描述

其他语言中通常将语句块称为`复合语句`。它允许你使用多个语句，其中 JavaScript 只需要一个语句。将语句组合成块是 JavaScript 中的常见做法。相反的做法是可以使用一个空语句，你不提供任何语句，虽然一个是必需的。

#### 块级作用域

**在非严格模式(non-strict mode)下的var 或者函数声明时**

通过var声明的变量或者非严格模式下(non-strict mode)创建的函数声明没有块级作用域。在语句块里声明的变量的作用域不仅是其所在的函数或者 script 标签内，所设置变量的影响会在超出语句块本身之外持续存在。 换句话说，这种语句块不会引入一个作用域。尽管单独的语句块是合法的语句，但在JavaScript中你不会想使用单独的语句块，因为它们不像你想象的C或Java中的语句块那样处理事物。例如：

```javascript
var x = 1;
{
  var x = 2;
}
console.log(x); // 输出 2
```

输出结果是 2，因为块中的 `var x`语句与块前面的`var x`语句作用域相同。在 C 或 Java中，这段代码会输出 1。

#### 使用let和 const

相比之下，使用 `let`和`const`声明的变量是有块级作用域的。

```javascript
let x = 1;
{
  let x = 2;
}
console.log(x); // 输出 1
```

`x = 2`仅限在定义它的块中。

`const`的结果也是一样的：

```javascript
const c = 1;
{
  const c = 2;
}
console.log(c); // 输出1, 而且不会报错
```
注意，位于块范围之内的 `const c = 2` 并不会抛出`SyntaxError: Identifier 'c' has already been declared`这样的语法错误，因为在它自己的块中它可能是唯一一个被声明的常量。

**使用let声明的变量在块级作用域内能强制执行更新变量，下面的两个例子对比：**

```javascript
var a = [];
for (var i = 0; i < 10; i++) {
      a[i] = function () {console.log(i);};
}
a[0]();                // 10
a[1]();                // 10
a[6]();                // 10

/********************/

var a = [];
for (let i = 0; i < 10; i++) {
      a[i] = function () {console.log(i);};
}
a[0]();                // 0
a[1]();                // 1
a[6]();                // 6
```

#### 使用function

函数声明同样被限制在声明他的语句块内:

```javascript
foo('outside');  // TypeError: foo is not a function
{
  function foo(location) {
   console.log('foo is called ' + location);
  }
  foo('inside'); // 正常工作并且打印 'foo is called inside'
}
```

## Iteration

### for语句

一个 for 循环会一直重复执行，直到指定的循环条件为 false。 JavaScript 的 for 循环，和 Java、C 的 for 循环，是很相似的。一个 for 语句是这个样子的：

```javascript
for ([initialExpression]; [condition]; [incrementExpression])
  statement
```

### do...while 语句

`do...while` 语句一直重复直到指定的条件求值得到假值（`false`）。 一个 `do...while` 语句看起来像这样：

```javascript
do
  statement
while (condition);
```

`statement` 在检查条件之前会执行一次。要执行多条语句（语句块），要使用块语句（{ ... }）包括起来。 如果 `condition` 为真（`true`），`statement` 将再次执行。 在每个执行的结尾会进行条件的检查。当 `condition` 为假（`false`），执行会停止并且把控制权交回给 `do...while` 后面的语句。

### while 语句

一个 `while` 语句只要指定的条件求值为真（true）就会一直执行它的语句块。一个 `while` 语句看起来像这样：

```javascript
while (condition)
  statement
```

如果这个条件变为假，循环里的 `statement` 将会停止执行并把控制权交回给 `while` 语句后面的代码。

条件检测会在每次 `statement` 执行之前发生。如果条件返回为真， `statement` 会被执行并紧接着再次测试条件。如果条件返回为假，执行将停止并把控制权交回给 `while` 后面的语句。

要执行多条语句（语句块），要使用语句块 (`{ ... }`) 包括起来。

### for...in 语句

`for...in` 语句循环一个指定的变量来循环一个对象所有可枚举的属性。JavaScript 会为每一个不同的属性执行指定的语句。

```javascript
for (variable in object) {
  statements
}
```

### for...of 语句

`for...of` 语句在`可迭代对象`（包括`Array`、`Map`、`Set`、`arguments` 等等）上创建了一个循环，对值的每一个独特属性调用一次迭代。

```javascript
for (variable of object) {
  statement
}
```

下面的这个例子展示了 `for...of` 和 `for...in` 两种循环语句之间的区别。 `for...in` 循环遍历的结果是数组元素的下标，而 `for...of` 遍历的结果是元素的值：

```javascript
let arr = [3, 5, 7];
arr.foo = "hello";

for (let i in arr) {
   console.log(i); // 输出 "0", "1", "2", "foo"
}

for (let i of arr) {
   console.log(i); // 输出 "3", "5", "7"
}

// 注意 for...of 的输出没有出现 "hello"
// 官方文档不知为何在此使用三个空格来缩进…
```

## 声明

### function 声明

函数声明定义一个具有指定参数的函数。

你还可以使用  Function 构造函数和 一个function expression 定义函数。

```javascript
function name([param,[, param,[..., param]]]) {
   [statements]
}
```

### 生成器函数

`function*` 这种声明方式(`function`关键字后跟一个星号）会定义一个**生成器函数** (`generator function`)，它返回一个`Generator`对象。

```javascript
function* generator(i) {
  yield i;
  yield i + 10;
}

const gen = generator(10);

console.log(gen.next().value);
// expected output: 10

console.log(gen.next().value);
// expected output: 20

```

#### 语法

```javascript
function* name([param[, param[, ... param]]]) { statements }
```

#### 描述

**生成器函数**在执行时能暂停，后面又能从暂停处继续执行。

调用一个**生成器函数**并不会马上执行它里面的语句，而是返回一个这个生成器的 迭代器 （ `iterator` ）对象。当这个迭代器的 `next()` 方法被首次（后续）调用时，其内的语句会执行到第一个（后续）出现`yield`的位置为止，`yield` 后紧跟迭代器要返回的值。或者如果用的是 `yield*`（多了个星号），则表示将执行权移交给另一个生成器函数（当前生成器暂停执行）。

`next()`方法返回一个对象，这个对象包含两个属性：`value` 和 `done`，`value` 属性表示本次 `yield` 表达式的返回值，`done` 属性为布尔类型，表示生成器后续是否还有 `yield` 语句，即生成器函数是否已经执行完毕并返回。

调用 `next()`方法时，如果传入了参数，那么这个参数会传给上一条执行的 `yield`语句左边的变量，例如下面例子中的 `x` ：

```javascript
function *gen(){
    yield 10;
    x=yield 'foo';
    yield x;
}

var gen_obj=gen();
console.log(gen_obj.next());// 执行 yield 10，返回 10
console.log(gen_obj.next());// 执行 yield 'foo'，返回 'foo'
console.log(gen_obj.next(100));// 将 100 赋给上一条 yield 'foo' 的左值，即执行 x=100，返回 100
console.log(gen_obj.next());// 执行完毕，value 为 undefined，done 为 true
```
当在生成器函数中显式`return`时，会导致生成器立即变为完成状态，即调用 `next()` 方法返回的对象的 `done` 为 `true`。如果 `return` 后面跟了一个值，那么这个值会作为当前调用 `next()` 方法返回的 `value` 值。

### async函数

async函数是使用`async`关键字声明的函数。 `async`函数是`AsyncFunction`构造函数的实例， 并且其中允许使用`await`关键字。`async`和`await`关键字让我们可以用一种更简洁的方式写出基于`Promise`的异步行为，而无需刻意地链式调用`promise`。

```javascript
function resolveAfter2Seconds() {
  return new Promise(resolve => {
    setTimeout(() => {
      resolve('resolved');
    }, 2000);
  });
}

async function asyncCall() {
  console.log('calling');
  const result = await resolveAfter2Seconds();
  console.log(result);
  // expected output: "resolved"
}

asyncCall();
```

#### 语法

```javascript
async function name([param[, param[, ... param]]]) {
    statements 
}
```

### 迭代异步生成器 

异步生成器已经实现了异步迭代器协议, 所以可以用 `for await...of`循环。

```javascript
async function* asyncGenerator() {
  var i = 0;
  while (i < 3) {
    yield i++;
  }
}

(async function() {
  for await (num of asyncGenerator()) {
    console.log(num);
  }
})();
// 0
// 1
// 2
```

有关使用`for await... of`考虑迭代API中获取数据的异步 `generator` 更具体的例子。这个例子首先为一个数据流创建了一个异步 `generator`，然后使用它来获得这个API的响应值的大小。

```javascript
async function* streamAsyncIterator(stream) {
  const reader = stream.getReader();
  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) {
        return;
      }
      yield value;
    }
  } finally {
    reader.releaseLock();
  }
}
// 从url获取数据并使用异步 generator 来计算响应值的大小
async function getResponseSize(url) {
  const response = await fetch(url);
  // Will hold the size of the response, in bytes.
  let responseSize = 0;
  // 使用for-await-of循环. 异步 generator 会遍历响应值的每一部分
  for await (const chunk of streamAsyncIterator(response.body)) {
    // Incrementing the total response length.
    responseSize += chunk.length;
  }

  console.log(`Response Size: ${responseSize} bytes`);
  // expected output: "Response Size: 1071472"
  return responseSize;
}
getResponseSize('https://jsonplaceholder.typicode.com/photos');
```

## 类声明

**class 声明**创建一个基于原型继承的具有给定名称的新类。

```javascript
class Polygon {
  constructor(height, width) {
    this.area = height * width;
  }
}

console.log(new Polygon(4, 3).area);
// expected output: 12
```

### 语法

```
class name [extends] {
  // class body
}
```

### 示例

在下面的例子中，我们首先定义一个名为Polygon的类，然后继承它来创建一个名为Square的类。注意，构造函数中使用的 super() 只能在构造函数中使用，并且必须在使用 this 关键字前调用。

```javascript
class Polygon {
  constructor(height, width) {
    this.name = 'Polygon';
    this.height = height;
    this.width = width;
  }
}

class Square extends Polygon {
  constructor(length) {
    super(length, length);
    this.name = 'Square';
  }
}
```