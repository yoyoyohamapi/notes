# callbag 实现例子

## 握手和对讲

握手过程：

sink 向 source 问好，同时 source 也向 sink 问好，就完成了一次握手过程。顺序也就是  `source(0, sink)` ，之后，在 `source` 内部的实现中，我们调用了  `sink(0, talkback)`。

talkback 会从 sink 收到了 `type=1` （交付数据）和 `type=2`（终止） ，但不会再收到 `type=0`（问好），因为握手已经完成了。

## 创建 sink

### Listener（Observer）

一个 listener sink 是一个 callbag 函数：

sink 和 source 交流的方式是依赖于一个 talkback（对讲函数）：

```js
function sink(type, data) {
  if (type === 0) {
    // sink 收到了来自 source 的问好
    // 问好的时候确定 source 和 sink 的对讲方式
    const talkback = data;
    // 3s 后，sink 终止和 source 的对讲
    setTimeout(() => talkback(2), 3000);
  }
}
```

当 `type=1` 时，sink 可以处理抽到的数据：

```js
function sink(type, data) {
    if (type === 0) {
        const talkback = data;
        setTimeout(() => talkback(2), 3000);
    }
    if (type === 1) {
        console.log(data);
    }
}
```

当 `type=2` 时，sink 将停止和 source 的对讲，理应清除掉定时器：

```js
let handle;
function sink(type, data) {
    if (type === 0) {
        const talkback = data;
        setTimeout(() => talkback(2), 3000);
    }
    if (type === 1) {
        console.log(data);
    }
    if (type === 2) {
        clearTimeout(handle);
    }
}
```

使用闭包来让代码干净些：

```js
function makeSink() {
  let handle;
  return function sink(type, data) {
    if (type === 0) {
      const talkback = data;
      handle = setTimeout(() => talkback(2), 3000);
    } 
    if (type === 1) {
      console.log(data);
    }
    if (type === 2) {
      clearTimeout(handle);
    }
  }
}
```

### Puller

一个 puller sink 也可以向 source 发送数据：

```js
let handle;
function sink(type, data) {
    if (type === 0) {
        const talkback = data;
        setInterval(() => talkback(1), 1000);
    }
    if (type === 1) {
        console.log(data);
    }
    if (type === 2) {
        clearTimeout(handle);
    }
}
```

## 创建 source

### Listenable（Observable）

无论来自 sink 的请求是否是 `type=1` ，listenable source 都会向 sink 发送数据。创建一个 listenable source 的方式使用 `setInterval` ，例如下面的代码中，source 将每秒向 sink 发送一个 `null`。

```js
function source(type, data) {
  if (type === 0) {
    // 如果 source 收到 sink 的问好，
    // 则第二参数就是 sink 自己
    // 则开始发送数据
    const sink = data;
    setInterval(() => {
      sink(1, null);
    }, 1000);
  }
}
```

哦，对了，忘记让 source 也和 sink 问好了：

```js
function source(type, data) {
    if (type === 0) {
        const sink = data;
        setInterval(() => {
            sink(1, null);
        }, 1000);
    }
    sink(0, /* talkback callbag here */)
}
```

我们需要让 talkback 识别 `type=2`，来让 sink 能终止从 source 处接收数据，也让 source 停止派发数据。同时，我们要注意到，每当有 sink 向 source 问好时，`setInterval` 就会被调用多次，当其中一个 sink 发送了 `type=2`，显然只应当停止该 sink 和 source 的对讲，而不应该影响其他 sink。

```js
function source(type, data) {
    if (type === 0) {
        const sink = data;
        let handle = setInterval(() => {
            sink(1, null);
        }, 1000);
    }
    const talkback = (type, data) => {
        if (type === 2) {
            clearInterval(handle);
        } 
    }
    sink(0, talkback);
}
```

listenable 的 source 不需要处理 `type=1` 和 `type=2`，并且其接收的 payload 是 sink。因此，我们可以优化代码：

```js
function source(start, sink) {
  if (start !== 0) return;
  let handle = setInterval(() => {
    sink(1, null);
  }, 1000);
  const talkback = (t, d) => {
    if (t === 2) clearInterval(handle);
  };
  sink(0, talkback);
}
```

### Pullable（Iterable）

pullable source 则是会等着 sink 送出一个 `type=1` 的对讲后，才会送出一个 `type=1` 的响应。下面的例子只会在需要时发送数字 10 到 20：

```js
function source(start, sink) {
    if (start !== 0) retrun;
    let i = 10;
    const talkback = (t, d) => {
        if (t == 1) {
            if (i <= 20) sink(1, i++);
            else sink(2);
        }
    }
    sink(1, talkback)
}
```

## 创建 operator

借助于 operator，能够不断的构建新的 source，operator 的一般范式为：

```js
const myOperator = args => inputSource => outputSource
```

借助于管道技术，我们能一步步的声明新的 source：

```js
pipe(
  source,
  myOperator(args),
  iterate(x => console.log(x))
)
// same as...
pipe(
  source,
  inputSource => outputSource,
  iterate(x => console.log(x))
)
```

下面我们创建了一个乘法 operator：

```js
const multiplyBy = factor => inputSource => {
  return function outputSource(start, outputSink) {
    if (start !== 0) return;
    inputSource(0, (t, d) => {
      if (t === 1) outputSink(1, d * factor);
      else outputSink(t, d);
    });
  };
}
```

使用：
```js
function source(start, sink) {
    if (start !== 0) return;
    let i = 0;
    const handle = setInterval(() => sink(1, i++), 3000);
    const talkback = (type, data) => {
        if (type === 2) {
            clearInterval(handle);
        }
    }
    sink(0, talkback);
}

let timeout;
function sink(type, data) {
    if (type === 0) {
        const talkback = data;
        timetout = setTimeout(() => talback(2), 9000);
    }
    if (type === 1) {
        console.log('data is', data);
    }
    if (type === 2) {
        clearTimeout(handle);
    }
}

const newSource = multiplyBy(3)(source);
newSource(0, sink);
```