# JavaScipt Type

最新的 ECMAScript 标准定义了 8 种数据类型:

- 7种原始类型:
  - Boolean
  - Null
  - Undefined
  - Number
  - BigInt
  - String
  - Symbol 
- 和 Object

所有基本类型的值都是不可改变的。但需要注意的是，基本类型本身和一个赋值为基本类型的变量的区别。变量会被赋予一个新值，而原值不能像数组、对象以及函数那样被改变。

这个示例会帮助你了解基本类型**不可改变**的事实。

```javascript
// 使用字符串方法不会改变一个字符串
var bar = "baz";
console.log(bar);               // baz
bar.toUpperCase();
console.log(bar);               // baz

// 使用数组方法可以改变一个数组
var foo = [];
console.log(foo);               // []
foo.push("plugh");
console.log(foo);               // ["plugh"]

// 赋值行为可以给基本类型一个新值，而不是改变它
bar = bar.toUpperCase();       // BAZ
```
基本类型值可以被替换，但不能被改变。下面的示例将让你体会到JavaScript是如何处理基本类型的。

```javascript
// 基本类型
let foo = 5;

// 定义一个貌似可以改变基本类型值的函数
function addTwo(num) {
   num += 2;
}
// 和前面的函数一样
function addTwo_v2(foo) {
   foo += 2;
}

// 调用第一个函数，并传入基本类型值作为参数
addTwo(foo);
// Getting the current Primitive value
console.log(foo);   // 5

// 尝试调用第二个函数...
addTwo_v2(foo);
console.log(foo);   // 5
```

你是否认为会得到`7`，而不是`5`？如果是，请看看代码是如何运行的：

- `addTwo`和`addTwo_v2`函数调用时，JavaScript会检查标识符`foo`的值，从而准确无误的找到第一行实例化变量的声明语句。
- 找到以后，JavaScript将其作为参数传递给函数的形参。
- 在执行函数体内语句之前，**JavaScript会将传递进来的参数（基本类型的值）复制一份**，创建一个本地副本。这个副本只存在于该函数的作用域中，我们能够通指定在函数中的标识符访问到它（`addTwo`中的`num`，`addTwo_v2`中的`foo`）。
- 接下来，函数体中的语句开始执行：
  - 第一个函数中，创建了本地`num`参数，`num`的值加`2`，但这个值并不是原来的`foo`的值。
  - 第二个函数中，创建了本地参数`foo`，并将它的值加`2`，这个值不是外部`foo`的值。在这种情况下，外部的`foo`变量不能以任何方式被访问到。这是因为JavaScript的词法作用域（lexical scoping）所导致的变量覆盖，本地的变量`foo`覆盖了外部的变量`foo`。
- 综上所述，函数中的任何操作都不会影响到最初的`foo`，我们操作的只不过是它的副本。
这就是为什么说所有基本类型的值都是无法改变的。

**JavaScript 中的基本类型包装对象**

除了`null`和`undefined`之外，所有基本类型都有其对应的包装对象：

`String`为字符串基本类型。

`Number`为数值基本类型。

`BigInt`为大整数基本类型。

`Boolean`为布尔基本类型。

`Symbol`为字面量基本类型。

这个包裹对象的`valueOf()`方法返回基本类型值。

## 原始值(primitive values)

除 Object 以外的所有类型都是不可变的（值本身无法被改变）。例如，与 C 语言不同，JavaScript 中字符串是不可变的（译注：如，JavaScript 中对字符串的操作一定返回了一个新字符串，原始字符串并没有被改变）。我们称这些类型的值为“原始值”。

### 布尔类型

布尔表示一个逻辑实体，可以有两个值：`true` 和 `false`。

### Null 类型

Null 类型只有一个值： `null`。

值`null`是一个字面量，不像`undefined`，它不是全局对象的一个属性，`null`是表示缺少的标识，指示变量未指向任何对象。把`null`作为尚未创建的对象，也许更好理解。在`API`中，`null`常在返回类型应是一个对象，但没有关联的值的地方使用。

```javascript
// foo 不存在，它从来没有被定义过或者是初始化过：
foo;
"ReferenceError: foo is not defined"

// foo 现在已经是知存在的，但是它没有类型或者是值：
var foo = null;
foo;
null
```

### Undefined 类型

一个没有被赋值的变量会有个默认值 `undefined`。

**null 与 undefined 的不同点：**

当检测`null`或`undefined`时，注意相等（==）与全等（===）两个操作符的区别 ，前者会执行类型转换：

```javascript
typeof null        // "object" (因为一些以前的原因而不是'null')
typeof undefined   // "undefined"
null === undefined // false
null  == undefined // true
null === null // true
null == null // true
!null //true
isNaN(1 + null) // false
isNaN(1 + undefined) // true
```

### 数字类型

根据 ECMAScript 标准，JavaScript 中只有一种数字类型：基于 IEEE 754 标准的双精度 64 位二进制格式的值（-(2^53 -1) 到 2^53 -1）。**它并没有为整数给出一种特定的类型**。除了能够表示浮点数外，还有一些带符号的值：`+Infinity`，`-Infinity` 和 `NaN` (非数值，Not-a-Number)。

要检查值是否大于或小于 `+/-Infinity`，你可以使用常量 `Number.MAX_VALUE` 和 `Number.MIN_VALUE`。另外在 ECMAScript 6 中，你也可以通过 `Number.isSafeInteger()` 方法还有 `Number.MAX_SAFE_INTEGER` 和 `Number.MIN_SAFE_INTEGER` 来检查值是否在双精度浮点数的取值范围内。 超出这个范围，JavaScript 中的数字不再安全了，也就是只有 second mathematical interger 可以在 JavaScript 数字类型中正确表现。

数字类型中只有一个整数有两种表示方法： 0 可表示为 -0 和 +0（"0" 是 +0 的简写）。 在实践中，这也几乎没有影响。 例如 `+0 === -0` 为真。 但是，你可能要注意除以0的时候：

```javascript
42 / +0; // Infinity
42 / -0; // -Infinity
```
### BigInt 类型

`BigInt`类型是 JavaScript 中的一个基础的数值类型，可以用任意精度表示整数。使用 BigInt，您可以安全地存储和操作大整数，甚至可以超过数字的安全整数限制。BigInt是通过在整数末尾附加`n`或调用构造函数来创建的。

通过使用常量`Number.MAX_SAFE_INTEGER`，您可以获得可以用数字递增的最安全的值。通过引入 BigInt，您可以操作超过`Number.MAX_SAFE_INTEGER`的数字。您可以在下面的示例中观察到这一点，其中递增`Number.MAX_SAFE_INTEGER`会返回预期的结果:

```javascript
> const x = 2n ** 53n;
9007199254740992n
> const y = x + 1n;
9007199254740993n
```

可以对`BigInt`使用运算符`+`、`*`、`-`、`**`和`%`，就像对数字一样。BigInt 严格来说并不等于一个数字，但它是松散的。

在将`BigInt`转换为`Boolean`时，它的行为类似于一个数字：`if`、`||`、`&&`、`Boolean` 和`!`。

`BigInt`不能与数字互换操作。否则，将抛出TypeError。

### 字符串类型

JavaScript的字符串类型用于表示文本数据。它是一组16位的无符号整数值的“元素”。在字符串中的每个元素占据了字符串的位置。第一个元素的索引为0，下一个是索引1，依此类推。字符串的长度是它的元素的数量。

不同于类 C 语言，JavaScript 字符串是不可更改的。这意味着字符串一旦被创建，就不能被修改。但是，可以基于对原始字符串的操作来创建新的字符串。例如：

获取一个字符串的子串可通过选择个别字母或者使用`String.substr()`.
两个字符串的连接使用连接操作符 (`+`) 或者 `String.concat()`.

### 符号类型

符号(Symbols)是ECMAScript 第6版新定义的。符号类型是唯一的并且是不可修改的, 并且也可以用来作为Object的key的值(如下). 在某些语言当中也有类似的原子类型(Atoms). 你也可以认为为它们是C里面的枚举类型.

数据类型 “**symbol**” 是一种基本数据类型，该类型的性质在于这个类型的值可以用来创建匿名的对象属性。该数据类型通常被用作一个对象属性的键值——当你想让它是私有的时候。例如，**symbol** 类型的键存在于各种内置的 JavaScript 对象中。同样，自定义类也可以这样创建私有成员。**symbol** 数据类型具有非常明确的目的，并且因为其功能性单一的优点而突出；一个 **symbol** 实例可以被赋值到一个左值变量，还可以通过标识符检查类型，这就是它的全部特性。不能对该类型实例使用其他操作符（将“**Symbol**”类型的实例与 “**Number**” 类型的实例对比，例如整数 42，该实例就具有将值与其他类型的值进行比较或组合的运算符）。

一个具有数据类型 “**symbol**” 的值可以被称为 “符号类型值”。在 JavaScript 运行时环境中，一个符号类型值可以通过调用函数 `Symbol()` 创建，这个函数动态地生成了一个匿名，唯一的值。Symbol类型唯一合理的用法是用变量存储 symbol的值，然后使用存储的值创建对象属性。以下示例使用"`var`"创建一个变量来保存 symbol。

```javascript
var  myPrivateMethod  = Symbol();
this[myPrivateMethod] = function() {...};
```

当一个 symbol 类型的值在属性赋值语句中被用作标识符，该属性（像这个 symbol 一样）是匿名的；并且是不可枚举的。因为这个属性是不可枚举的，它不会在循环结构 “`for( ... in ...)`” 中作为成员出现，也因为这个属性是匿名的，它同样不会出现在 “`Object.getOwnPropertyNames()`” 的返回数组里。这个属性可以通过创建时的原始 symbol 值访问到，或者通过遍历 “`Object.getOwnPropertySymbols()`” 返回的数组。在之前的代码示例中，通过保存在变量 `myPrivateMethod`的值可以访问到对象属性。

内置函数 “[`Symbol`](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol)`()`” 是一个不完整的类，当作为函数调用时会返回一个 symbol 值，试图通过语法 “`new Symbol()`” 作为构造函数调用时会抛出一个错误。它有一些静态方法来访问 JavaScript 全局 symbol 表，还有一些静态属性用来保存已存在通用对象里的特定 symbol 地址。通过 `Symbol()` 函数创建 symbol 值已在上面解释过。理解试图将 `Symbol()`作为构造函数使用会抛出错误可以提前预防意外（当作构造函数使用）创建对象导致的混乱。访问全局 symbol 注册表的方法是 `Symbol.for()`和 `Symbol.keyFor()`；它们斡旋在全局 symbol 表（或注册表）与运行时环境之间。symbol 注册表通常构建在 JavaScript 编译器基础设施，所以 symbol 注册表的内容不会出现 JavaScript 运行时环境，除了通过它们的反射方法。*`Symbol.for("tokenString")`* 方法从注册表返回一个 symbol 值，*`Symbol.keyFor(symbolValue)`* 方法从注册表返回 token 字符串；正好相反，所以下面为 true：

```javascript
Symbol.keyFor(Symbol.for("tokenString"))=="tokenString";  // true
```

**Symbol** 类具有一些静态属性，对于匿名命名来说，这带有一点讽刺意味。这类属性只有几个; 它们是所谓的“众所周知”的 symbol。 它们是在某些内置对象中找到的某些特定方法属性的 symbol。 暴露出这些 symbol 使得可以直接访问这些行为；这样的访问可能是有用的，例如在定义自定义类的时候。 普遍的 symbol 的例子有：“`Symbol.iterator`”用于类似数组的对象，“`Symbol.search`”用于字符串对象。

`Symbol()`函数及其创建的 symbol 值可能对设计自定义类的编程人员有用。 symbol 值提供了一种自定义类可以创建私有成员的方式，并维护一个仅适用于该类的 symbol 注册表。 自定义类可以使用 symbol 值来创建“自有”属性，这些属性避免了不必要的被偶然发现带来的影响。 在类定义中，动态创建的 symbol 值将保存到作用域变量中，该变量只能在类定义中私有地使用。 没有 token 字符串; 作用域变量起到 token 的等同作用。


### 对象

在计算机科学中, 对象是指内存中的可以被 标识符引用的一块区域.

#### 属性

在 Javascript 里，对象可以被看作是一组属性的集合。用对象字面量语法来定义一个对象时，会自动初始化一组属性。（也就是说，你定义一个var a = "Hello"，那么a本身就会有a.substring这个方法，以及a.length这个属性，以及其它；如果你定义了一个对象，var a = {}，那么a就会自动有a.hasOwnProperty及a.constructor等属性和方法。）而后，这些属性还可以被增减。属性的值可以是任意类型，包括具有复杂数据结构的对象。属性使用键来标识，它的键值可以是一个字符串或者符号值（Symbol）。

ECMAScript定义的对象中有两种属性：数据属性和访问器属性。

## 有序集: 数组和类型数组

`数组`是一种使用整数作为键(integer-key-ed)属性和长度(length)属性之间关联的常规对象。此外，数组对象还继承了 Array.prototype 的一些操作数组的便捷方法。例如, `indexOf` (搜索数组中的一个值) or `push` (向数组中添加一个元素)，等等。 这使得数组是表示列表或集合的最优选择。

类型数组(Typed Arrays)是ECMAScript Edition 6中新定义的 JavaScript 内建对象，提供了一个基本的二进制数据缓冲区的类数组视图。下面的表格能帮助你找到对等的 C 语言数据类型：

| Type                                                         | Value Range                     | Size in bytes | Description                                                  | Web IDL type          | Equivalent C type               |
| :----------------------------------------------------------- | :------------------------------ | :------------ | :----------------------------------------------------------- | :-------------------- | :------------------------------ |
| [`Int8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int8Array) | `-128` to `127`                 | 1             | 8-bit two's complement signed integer                        | `byte`                | `int8_t`                        |
| [`Uint8Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8Array) | `0` to `255`                    | 1             | 8-bit unsigned integer                                       | `octet`               | `uint8_t`                       |
| [`Uint8ClampedArray`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint8ClampedArray) | `0` to `255`                    | 1             | 8-bit unsigned integer (clamped)                             | `octet`               | `uint8_t`                       |
| [`Int16Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int16Array) | `-32768` to `32767`             | 2             | 16-bit two's complement signed integer                       | `short`               | `int16_t`                       |
| [`Uint16Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint16Array) | `0` to `65535`                  | 2             | 16-bit unsigned integer                                      | `unsigned short`      | `uint16_t`                      |
| [`Int32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Int32Array) | `-2147483648` to `2147483647`   | 4             | 32-bit two's complement signed integer                       | `long`                | `int32_t`                       |
| [`Uint32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Uint32Array) | `0` to `4294967295`             | 4             | 32-bit unsigned integer                                      | `unsigned long`       | `uint32_t`                      |
| [`Float32Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float32Array) | `1.2`×`10-38` to `3.4`×`1038`   | 4             | 32-bit IEEE floating point number (7 significant digits e.g., `1.1234567`) | `unrestricted float`  | `float`                         |
| [`Float64Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Float64Array) | `5.0`×`10-324` to `1.8`×`10308` | 8             | 64-bit IEEE floating point number (16 significant digits e.g., `1.123...15`) | `unrestricted double` | `double`                        |
| [`BigInt64Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigInt64Array) | `-263` to `263-1`               | 8             | 64-bit two's complement signed integer                       | `bigint`              | `int64_t (signed long long)`    |
| [`BigUint64Array`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/BigUint64Array) | `0` to `264-1`                  | 8             | 64-bit unsigned integer                                      | `bigint`              | `uint64_t (unsigned long long)` |

