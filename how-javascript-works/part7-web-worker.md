# Web Worker

## 任务异步化

```js
function average(numbers) {
    var len = numbers.length,
        sum = 0,
        i;
    if (len === 0) {
        return 0;
    }
    
    for (i = 0; i < len; i++) {
        sum += numbers[i]
    }
    
    return sum / len;
}
```

异步化：

```js
function averageAsync(numbers. callback) {
    var len = numbers.length,
        sum = 0;
    
    if (len === 0) {
        return 0;
    }
    
    function calculateSumAsync(i) {
        if (i < len) {
            setTimeout(function() {
                sum += numbers[i];
                calculateSumAsync(i + 1);
            }, 0)
        } else {
            callback(sum / n)
        }
    }
}
```

借助于 `setTimeout` ，我们将每一步求和操作都放到了任务队列，让出了线程占用。

## Web Worker

Web Worker 是浏览器的内置的线程技术，能够在不阻塞 event loop 的情况下，执行 JavaScript。

Web Worker 并没有打破 JavaScript 单线程的范式，它不过是浏览器提供的，能够通过 JavaScript 访问的特性。

Web Worker 有如下三种类型：

- **Dedicated Workers**：由主进程实例化，且只能和主线程通信。
- **Shared Workers**：可以被同源下的所有进程访问，即便这些进程运行在不同的标签页或者 iframe 中。
- **Service Workers**：它是一个事件驱动的线程，根据源（origin）和路径（path）注册。它可以拦截、修改导航以及资源请求，以更细的粒度缓存资源，帮助你的应用在诸如离线环境这样的场景时也能有好的表现。

## 初始化 Worker

Web Worker 是以一个 `.js` 文件实现的，该文件通过异步的 HTTP 请求进行加载。Web Woker 运行在浏览器中相互隔离的线程，因此，Web Worker 执行的代码需要包含在**分离的文件**。

```js
var worker = new Worker('task.js');
```

使用 `postMessage` 来启动 worker：

```js
worker.postMessage();
```

## `postMessage()`

较新的浏览器支持传递 `JSON` 作为 `postMessage()` 的第一个参数，老版本的浏览器只支持 `string`。

看一个更复杂的例子，这是页面：

```html
<button onclick="startComputation()">
    Start computation
</button>

<script>
    function startComputation() {
        worker.postMessage({cmd: 'average', data: [1, 2, 3]});
    }
    
    var woker = new Worker("doWork.js");
    
    worker.addEventListener('message', function(e) {
        console.log(e.data);
    }, false);
</script>
```

下面则是 worker 的内容：

```js
self.addEventListener('message', function(e) {
  var data = e.data;
  switch (data.cmd) {
    case 'average':
      var result = calculateAverage(data); 
      self.postMessage(result);
      break;
    default:
      self.postMessage('Unknown command');
  }
}, false);
```

在 Web Worker 中，`self` 和 `this` 都指向了 worker 所在的全局作用域。

停止 worker 的方法有两种：

1. 在主页面执行 `worker.terminate()`
2. 在 worker 中调用 `self.close()`

## 广播

[Broadcast Channel](https://developer.mozilla.org/en-US/docs/Web/API/BroadcastChannel) API 能够广播消息到同源下的所有上下文。包括所有同源的 tab 页和 iframe 等。

```js
var bc = new BroadChannel('test_channel');

bc.postMessage('This is a test message');

bc.onmessage = function (e) {
    console.log(e.data);
}

bc.close();
```

![](https://cdn-images-1.medium.com/max/1600/1*NVT6WbNrH_mQL64--b-l1Q.png)

## 向 Web Worker 发送消息

- **复制消息**：消息在一端进行序列化，复制，并发送到另一端，在另一端进行反序列化。页面和 worker 不会共享相同的消息实例，接收端使用的是消息的拷贝。多数浏览器是通过 JSON 的自动编解码来实现消息传输的。因此，如果消息体越大，耗时越久。
- **转移消息**：这意味着消息的发送方一旦发送了消息，就不能再使用它。消息的转移近乎是瞬时的。唯一的限制是只能使用 [ArrayBuffer](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer) 来转移消息。

## Web Worker 能访问的浏览器特性

1. `navigator` 对象
2. `location` 对象（只读）
3. `XMLHttpRequest`
4. `setTimeout()/clearTimeout()` 和 `setInterval()/clearInterval()`
5. [Application Cache](https://www.html5rocks.com/tutorials/appcache/beginner/)
6. 通过  `importScripts()` 引入额外的脚本
7. 创建其他的 Web Worker

## Web Worker 限制

1. DOM
2. `window`
3. `document`
4. `parent`

一言蔽之，Web Worker 是不能操纵 DOM 的。

## 错误处理

当 Worker 执行时，如果发生错误，则会发出一个 `ErrorEvent`，其包含了 3 个信息：

- **filename**：发生错误的 worker 文件
- **lineno**：发生错误的行号
- **message**：错误描述

```js
function onError(e) {
    console.log('Line: ' + e.lineno);
    console.log('In: ' + e.filename);
    console.log('Message: ' + e.message);
}

var worker = new Worker('workerWithError.js');
worker.addEventListener('error', onError, false);
worker.postMessage();
```

在 workerWithError.js 中：

```js
self.addEventListener('message', function(e) {
  postMessage(x * 2); // Intentional error. 'x' is not defined.
};
```

## Web Worker 使用场景

1. **图像处理**：将图像进行划分，并使用多个 worker（也就能利用多个 CPU） 进行处理或者渲染。例子：https://nerget.com/rayjs-mt/rayjs.html
2. **加密技术**
3. **数据预加载**
4. **PWA（渐进式 Web 应用）**：即便在网络条件比较差的时候，渐进式 Web 应用也需要快速加载。这意味着数据要被保存在本地。如果你使用了  [IndexDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API) ，就可以使用异步的存取 API，但如果使用同步的 API，则需要 worker 来提高性能。
5. **拼写检查**：拼写检查的工作流程为：程序读取一个包含了正确单词拼写列表的字典文件。字典被解析为了搜索树。当检查器接收到了一个单词，它就在搜索树上查找这个单词。如果没有找到，就会认为用户拼写错误。所以，将繁重的搜索和建议任务交给 Worker，用户在 UI 上的输入才不会出现卡顿。