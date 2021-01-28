# JavaScript Expressions

- a.b
- a[b]
- foo`string`
- super.b
- super['b']
- new.target
- new Foo()

## new运算符

**`new`运算符**创建一个用户定义的对象类型的实例或具有构造函数的内置对象的实例。

```javascript
function Car(make, model, year) {
  this.make = make;
  this.model = model;
  this.year = year;
}

const car1 = new Car('Eagle', 'Talon TSi', 1993);

console.log(car1.make);
// expected output: "Eagle"
```

### 语法

> new constructor[([arguments])]

**参数**

`constructor`:

一个指定对象实例的类型的类或函数。

`arguments`:

一个用于被 constructor 调用的参数列表。

### 描述

`new`关键字会进行如下的操作：

1. 创建一个空的简单JavaScript对象（即{}）；
2. 链接该对象（设置该对象的constructor）到另一个对象 ；
3. 将步骤1新创建的对象作为this的上下文 ；
4. 如果该函数没有返回对象，则返回this。

（注：关于对象的 constructor，参见 Object.prototype.constructor）

创建一个用户自定义的对象需要两步：

1. 通过编写函数来定义对象类型。
2. 通过`new`来创建对象实例。

创建一个对象类型，需要创建一个指定其名称和属性的函数；对象的属性可以指向其他对象，看下面的例子：

当代码`new Foo(...)`执行时，会发生以下事情：

1. 一个继承自`Foo.prototype的新对象被创建。
2. 使用指定的参数调用构造函数 `Foo`，并将 `this` 绑定到新创建的对象。`new Foo` 等同于 `new Foo()`，也就是没有指定参数列表，`Foo` 不带任何参数调用的情况。
3. 由构造函数返回的对象就是 new 表达式的结果。如果构造函数没有显式返回一个对象，则使用步骤1创建的对象。（一般情况下，构造函数不返回值，但是用户可以选择主动返回对象，来覆盖正常的对象创建步骤）

你始终可以对已定义的对象添加新的属性。例如，`car1.color = "black"` 语句给 `car1` 添加了一个新的属性 `color`，并给这个属性赋值 `"black"`。但是，这不会影响任何其他对象。要将新属性添加到相同类型的所有对象，你必须将该属性添加到 `Car` 对象类型的定义中。

你可以使用 `Function.prototype` 属性将共享属性添加到以前定义的对象类型。这定义了一个由该函数创建的所有对象共享的属性，而不仅仅是对象类型的其中一个实例。下面的代码将一个值为 `null` 的 `color` 属性添加到 `car` 类型的所有对象，然后仅在实例对象 `car1` 中用字符串 `"black"` 覆盖该值。

```javascript
function Car() {}
car1 = new Car();
car2 = new Car();

console.log(car1.color);    // undefined

Car.prototype.color = "original color";
console.log(car1.color);    // original color

car1.color = 'black';
console.log(car1.color);   // black

console.log(car1.__proto__.color) //original color
console.log(car2.__proto__.color) //original color
console.log(car1.color)  // black
console.log(car2.color) // original color
```

> 如果你没有使用 `new` 运算符， **构造函数会像其他的常规函数一样被调用**， 并不会创建一个对象。在这种情况下， `this` 的指向也是不一样的。

## new.target

`new.target`属性允许你检测函数或构造方法是否是通过`new`运算符被调用的。在通过`new`运算符被初始化的函数或构造方法中，`new.target`返回一个指向构造方法或函数的引用。在普通的函数调用中，`new.target` 的值是`undefined`。

```javascript
function Foo() {
  if (!new.target) throw "Foo() must be called with new";
  console.log("Foo instantiated with new");
}

Foo(); // throws "Foo() must be called with new"
new Foo(); // logs "Foo instantiated with new"
```

在类的构造方法中，new.target指向直接被new执行的构造函数。并且当一个父类构造方法在子类构造方法中被调用时，情况与之相同。

```javascript
class A {
  constructor() {
    console.log(new.target.name);
  }
}

class B extends A { constructor() { super(); } }

var a = new A(); // logs "A"
var b = new B(); // logs "B"

class C { constructor() { console.log(new.target); } }
class D extends C { constructor() { super(); } }

var c = new C(); // logs class C{constructor(){console.log(new.target);}}
var d = new D(); // logs class D extends C{constructor(){super();}}
```

## super

`super`关键字用于访问和调用一个对象的父对象上的函数。

`super.prop`和`super[expr]`表达式在类和对象字面量任何方法定义中都是有效的。

### 语法

> super([arguments]);
> // 调用 父对象/父类 的构造函数
> 
> super.functionOnParent([arguments]);
> // 调用 父对象/父类 上的方法

### 描述

在构造函数中使用时，super关键字将单独出现，并且必须在使用this关键字之前使用。super关键字也可以用来调用父对象上的函数。

### 示例

**在类中使用super**

```javascript
class Polygon {
  constructor(height, width) {
    this.name = 'Rectangle';
    this.height = height;
    this.width = width;
  }
  sayName() {
    console.log('Hi, I am a ', this.name + '.');
  }
  get area() {
    return this.height * this.width;
  }
  set area(value) {
    this._area = value;
  }
}

class Square extends Polygon {
  constructor(length) {
    this.height; // ReferenceError，super 需要先被调用！

    // 这里，它调用父类的构造函数的,
    // 作为Polygon 的 height, width
    super(length, length);

    // 注意: 在派生的类中, 在你可以使用'this'之前, 必须先调用super()。
    // 忽略这, 这将导致引用错误。
    this.name = 'Square';
  }
}
```

**调用父类上的静态方法**

```javascript
class Rectangle {
  constructor() {}
  static logNbSides() {
    return 'I have 4 sides';
  }
}

class Square extends Rectangle {
  constructor() {}
  static logDescription() {
    return super.logNbSides() + ' which are all equal';
  }
}
Square.logDescription(); // 'I have 4 sides which are all equal'
```

**删除 super 上的属性将抛出异常**

你不能使用 `delete` 操作符 加 `super.prop` 或者 `super[expr]` 去删除父类的属性，这样做会抛出 `ReferenceError`。

```javascript
class Base {
  constructor() {}
  foo() {}
}
class Derived extends Base {
  constructor() {}
  delete() {
    delete super.foo; // this is bad
  }
}

new Derived().delete(); // ReferenceError: invalid delete involving 'super'.
```

**super.prop 不能覆写不可写属性**

当使用 `Object.defineProperty` 定义一个属性为不可写时，`super`将不能重写这个属性的值。

```javascript
class X {
  constructor() {
    Object.defineProperty(this, 'prop', {
      configurable: true,
      writable: false,
      value: 1
    });
  }
}

class Y extends X {
  constructor() {
    super();
  }
  foo() {
    super.prop = 2;   // Cannot overwrite the value.
  }
}

var y = new Y();
y.foo(); // TypeError: "prop" is read-only
console.log(y.prop); // 1
```

**在对象字面量中使用super.prop**

Super也可以在object initializer / literal 符号中使用。在下面的例子中，两个对象各定义了一个方法。在第二个对象中, 我们使用super调用了第一个对象中的方法。 当然，这需要我们先利用 Object.setPrototypeOf() 设置obj2的原型为obj1，然后才能够使用super调用 obj1上的method1。

```javascript
var obj1 = {
  method1() {
    console.log("method 1");
  }
}

var obj2 = {
  method2() {
   super.method1();
  }
}

Object.setPrototypeOf(obj2, obj1);
obj2.method2(); // logs "method 1"
```

