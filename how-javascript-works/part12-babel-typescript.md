# Babel 及 TypeScript 对于 class 的转换

在 JavaScript 中，是没有原始数据类型（primitive type）的，任何我们所创建的都是对象。例如，我们创建的字符串，它也是一个对象，具有一些方法：

```js
const name = 'SessionStack'

console.log(name.repeat(2))
console.log(name.toLowerCase())
```

诸如数组这样复合类型：

```js
const names = ['SessionStack']
```

实际上也是对象：

```js
const names = {
  0: 'SessionStack',
  length: 1
}
```

这也是为什么在 JavaScript 中，访问对象属性和访问数组元素性能上并无差异的原因。

## 使用原型来模拟类

JavaScript 中并没有经典的**基于类的继承**，每个对象之间都通过原型相互连接，当我们尝试调用对象上的方法时，会先在其原型上进行查找，如果没找到，就在连接的原型上依次寻找。

![img](https://cdn-images-1.medium.com/max/1600/0*SufKRGfPZIDlw1OG)

我们创建一个组件，并在其原型上挂载一个渲染函数：

```js
function Component(content) {
  this.content = content
}

Component.prototype.render = function() {
	console.log(this.content) 
}
```

![img](https://cdn-images-1.medium.com/max/1600/0*hZbijxS0vXu8vUmz)

接下来我们创建一个 Input 组件，这个组件尝试继承 Component：

```js
function InputField() {
  this.content = `<input type="text" value="${value}" />`
}
```

为了继承 Component 上已经有的渲染方法，我们就需要连接 InputField 的原型到 Component 的实例上：

```js
InputField.prototype = Object.create(new Component())
```

![img](https://cdn-images-1.medium.com/max/1600/0*avLiOV_zXLxOgBee)

这只是最基础的原型继承，完成一个继承我们往往还需要：

- 设置子类原型
- 在子类的构造函数中，调用父类的构造函数，对父类进行构造
- 决定是否要重载父类方法

## Babel 对于类的转换

在 ES6 中，有了 class 关键字用来定义类：

```js
class Component {
  constructor(content) {
    this.content = content;
  }

  render() {
  	console.log(this.content)
  }
}

const component = new Component('SessionStack');
component.render();
```

Bebel 会将上述代码编译为：

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
  }]);

  return Component;
}();
```

其中，`_classCallCheck` 用来保证类的构造函数不会被当做普通函数调用，其原理是通过判断构造函数的执行上下文是否是类的实例化对象。

`_createClass` 则用来为类创建属性。

此时我们再使用 class 来创建 InputField：

```js
class InputField extends Component {
  constructor(value) {
    const content = `<input type="text" value="${value}" />`;
    super(content);
  }
}
```

Babel 会将这段代码转换为：

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

继承逻辑被放到了 `_inherits` 内部，该函数内部实现即为 ES5 中的原型继承过程。

Babel 转换 ES2015 代码为 ES5 代码的过程如下：

- ES2015 代码被解析为一棵抽象语法树（AST）
- 然后，这棵 AST 又被转换一个节点都对应到 ES5 的 AST
- 最终，新的 AST 被转换为 ES5 代码

下面这段代码：

```js
class Component {
  constructor(content) {
    this.content = content;
  }

  render() {
    console.log(this.content)
  }
}
```

将被 Babel 转为为一棵 AST：

![img](https://cdn-images-1.medium.com/max/1600/0*-OqUfzpRtgDJQjXY)

这棵 AST 将使用**深度优先遍历**进行代码转化，首先类的两个方法定义节点（MethodDefinition）会先被转换，其次是 ClassBody 节点，最后是 ClassDeclaration 节点。

## TypeScript 对于类的转换

如果我们使用 TypeScript 来撰写类：

```ts
class Component {
    content: string;
    constructor(content: string) {
        this.content = content;
    }
    render() {
        console.log(this.content)
    }
}
```

TypeScript 将生成如下的 AST：

![img](https://cdn-images-1.medium.com/max/1600/0*j3zkSjnrL4fnCK3A)

同样是使用深度优先遍历进行代码转换。

TypeScirpt 会将下面的类继承代码：

```ts
class InputField extends Component {
  constructor(value: string) {
    const content = `<input type="text" value="${value}" />`;
    super(content);
  }
}
```

转换为：

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

`_extends` 完成的同样是原型继承必须做的几件事儿。