# 深入类和继承

在 JavaScript 中，没有原始类型，我们创建的所有都是对象：

```js
const name = 'SessionStack';
```

我们可以直接调用在对象上的方法：

```js
console.log(name.repeat(2));
console.log(name.toLowerCase());
```

与其他语言不通，JavaScript 中，声明一个字符串或者数字，会自动创建一个对象来包裹值并且为其提供方法。

数组也是对象，其位置索引是对象的一个 `key`：

```js
let names = ['SessionStack'];

let names = {
  0: 'SessionStack',
  length: 1
}
```

因此，访问数组元素的效率和访问对象属性的效率是几乎一致的。

## 使用 prototypes 来模拟类

JavaScript 中，使用 prototype 来的实现继承：

![](https://cdn-images-1.medium.com/max/1600/0*SufKRGfPZIDlw1OG)

在 JavaScript 中，每个对象都通过 prototype 连接到其他对象。如果你尝试在对象上访问属性或者方法，首先会查找在自身上查找是否具有属性或方法，如果没有查找到，就会在对象原型上进行查找。

我们创建一个组件类的构造函数：

```js
function Component(content) {
  this.content = content;
}

Component.prototype.render = function () {
  console.log(this.content);
}
```

![](https://cdn-images-1.medium.com/max/1600/0*hZbijxS0vXu8vUmz)

我们新建一个组件：

```js
function InputField(value) {
  this.content = `<input type='text' value='${value}' />`
}
```

继承 `Component` ，来获得组件的基础能力：

```js
InputField.prototype = Object.create(new Component());
```

![](https://cdn-images-1.medium.com/max/1600/0*avLiOV_zXLxOgBee)

在 JavaScript 中，我们继承一个类，需要做到：

- 设置子类的原型为其父类的一个实例
- 在子类的构造函数中，调用父类的构造函数
- 引入某种方式来访问父类方法。这样，避免方法被重载后，找不到父类原始方法。

这个过程并不简单。因此，开发者并不喜欢基于原型的继承，而更青睐于基于类的继承。

## `class` 与 Babel

在 ES6 中，提供了 `class` 语法糖来模拟类继承：

```js
class Component {
  constructor(content) {
    this.content = content
  }
  
  render() {
    console.log(this.content)
  }
}

const component = new Component('SessionStack')
component.render()
```

Babel 将这段代码转换为：

```js
var Component = function () {
  function Component(content) {
    _classCallCheck(this, Component);
    
    this.content = content;
  }
  
  _createClass(Component, [{
    key: 'render',
    value: function render() {
      console.log(this.content);
    }
  }])
  
  return Component;
}();
```

这是一段 ES5 代码，其中，`_classCallCheck`、`_createClass` 为 Babel 标准库中提供的方法。

`_classCallCheck` 保证构造函数不会被当做函数被调用，其工作过程为：检查函数的上下文是否是 `Component` 的一个实例。

`_createClass` 负责对象中属性的创建。

为了探索继承时怎么工作的，我们创建一个 `InputField`：

```js
class InputField extends Component {
  constructor(value) {
    const content = `<input type='text' value='${value}' />`
    super(content)
  }
}
```

Babel 转换后的代码为：

```js

var InputField = function (_Component) {
  _inherits(InputField, _Component);

  function InputField(value) {
    _classCallCheck(this, InputField);

    var content = '<input type="text" value="' + value + '" />';
    return _possibleConstructorReturn(this, (InputField.__proto__ || Object.getPrototypeOf(InputField)).call(this, content));
  }

  return InputField;
}(Component);
```

继承逻辑被放在了 `_inherits` 中。这个函数就封装了上文提到的继承过程，设置了子类原型为父类实例。

Babel 的代码转换过程为：

- 将 ES6 代码转换为 AST
- AST 被转换为最后的代码

## Babel 中的 AST

下面这段代码：

```js
class Component {
  constructor(content) {
    this.content = content;
  }
  
  render() {
    console.log(this.content);
  }
}
```

Babel 编译的 AST 约为：

![](https://cdn-images-1.medium.com/max/1600/0*-OqUfzpRtgDJQjXY)

AST 创建后，Babel 使用深度优先遍历将每个节点转换为对应的 ES5 节点。在这个例子中，两个 MethodDefinition 将会先被转换，之后会是 ClassBody，最后是整个 ClassDeclaration。

## TypeScript 中的 AST

下面的 TypeScript 代码：

```typescript
class Component {
  content: string;
  constructor(content: string) {
    this.content = content;
  }
  render() {
    console.log(this.content);
  }
}
```

生成的 AST 为：

![](https://cdn-images-1.medium.com/max/1600/0*j3zkSjnrL4fnCK3A)

TypeScript 也支持继承：

```js
class InputField extends Component {
  constructor(value: string) {
    const content = `<input type="text" value="${value}">`
    super(content)
  }
}
```

转换结果为：

```js
var InputField = /** @class */ (function (_super) {
    __extends(InputField, _super);
    function InputField(value) {
        var _this = this;
        var content = "<input type=\"text\" value=\"" + value + "\" />";
        _this = _super.call(this, content) || this;
        return _this;
    }
    return InputField;
}(Component));
```

## V8 的支持

V8 原生提供了对 ES6 class 的支持，V8 生成 AST 种包含了 ClassLiteral 节点。这个节点存储了构造函数和类的属性，这些属性可以是方法，getter，setter，public 或者 private field。节点也存储了父类的引用。

最终，ClassLiteral 仍将被转化为函数和 prototype。