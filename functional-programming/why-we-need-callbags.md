# Why we need callbags

## JavaScript 中的 Push 与 Pull

![](https://i.stack.imgur.com/HwvQv.png)

- **Pull（`f(): B`）**：返回一个值。
- **Push（`f(x: A): void`）**：响应式的，当有值产生时，会发出一个事件，并携带上这个值。订阅了该事件的观察者（Observer）将获得反馈。

JavaScript 中的 `Math.random()`、`UUID()`、`window.outerHeight()`  以及网络请求都是 pull 风格的。

## Observable 与 Observer

在响应式编程中，对 Observable 和 Observer 的定义大致是：

```
interface Observer {
  next(x): void;
  error(e): void;
  complete(): void;
}

interface Observable {
  subscribe(observer): Subscription;
  unsubsribe(): void;
}
```

这也可以换一种表述：

```javascript
function observable(msgType, msgPayload) {
}
```

当：

- `msgType == 0`：payload 应当是 observer 函数。
- `msgType == 1`：意味着 unsubsribe。

```js
function observer(msgType, msgPayload) {
}
```

当：

- `msgType == 1`：对应于 `observer.next()`，payload 携带了数据。
- `msgType == 2` 且 payload 为 `undefined`：对应于 `observer.complete()`。
- `msgType == 2` 且 payload 含有值：对应于 `observer.error`

再概括一下就是：

**Observer**:

- ```
  observer(1, data): void
  ```

  - “next”

- ```
  observer(2, err): void
  ```

  - “error”

- ```
  observer(2): void
  ```

  - “complete”

**Observable**:

- ```
  observable(0, observer): void
  ```

  - “subscribe”

- ```
  observable(2): void
  ```

  - “unsubscribe”

转换为对应的可迭代对象程序（iterable programming）描述就是：

**Consumer**：

- ```
  consumer(0, producer): void
  ```

  - **开始时，将 producer 传给 consumer，即 payload 为 producer**

- ```
  consumer(1, data): void
  ```

  - “next”

- ```
  consumer(2, err): void
  ```

  - “error”

- ```
  consumer(2): void
  ```

  - “complete”

**Producer**：

- ```
  producer(0, consumer): void
  ```

  - “subscribe”

- ```
  producer(1, data): void
  ```

  - **数据传输时， 从 consumer 处接收到数据**

- ```
  producer(2): void
  ```

  - “unsubscribe” ，接收自 consumer。

现在，一个 consumer 可以是一个 **Listener（“observer”）**，或是一个 **Puller**，这取决于其是否会 pull producer。而一个 producer 可以是一个 **Listenable（“Observable”）**，或者是一个 **Pullable（“Iterable”）**，这取决于其是主动发出数据还是按需发送数据。如你所见，consumer 和 producer 是同类型（函数 signature 相同）的简单函数。