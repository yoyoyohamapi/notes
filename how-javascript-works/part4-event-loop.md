#  Event Loop 和 5 个在使用 async/await 时的建议

JavaScript 引擎需要工作在**宿主环境**下，如浏览器或者 Node.js。但无论在哪个环境，都绕不开 **Event Loop**。

![](https://cdn-images-1.medium.com/max/1600/1*FA9NGxNB6-v1oI2qGEtlRQ.png)

例如，我们创建了一个 Ajax 请求去送服务器上拿取数据，你也通过一个回调设置了处理响应的方式。这时，JavaScript 引擎会告诉宿主环境：

”我将推迟这个回调函数的执行，如果你完成了网络请求，并且获得了响应数据，就请调用这个回调函数“。

之后，浏览器开始监听网络请求，一旦有响应，其调度机制是，将回调函数插入到 Event Loop 中。

Event Loop 的职责比较简单 —— 监听调用栈（Call Stack）和回调队列（Callback Queue）。当调用栈空了，Event Loop 将从回调队列中取出第一个事件放入调用栈，并且执行它。

在 Event Loop 中，这样一场过程被称作是一次 **tick**，宛如手表指针转动了一下。每一个 Event Loop 中的事件都不过是一个回调函数。

有趣的是，ES6 说明了事件循环应当如何工作，这意味着 Event Loop 被认为是 JavaScript 引擎的指责了，而不再是宿主环境的一员。改变的主要原因是 ES6 引入了原生的 Promise，Promise 要求能在事件循环队列上访问一个直接的、细粒度的任务管理。

### Event Loop 的工作过程

1. 一开始，一切都是空的：

   ![](https://cdn-images-1.medium.com/max/1600/1*9fbOuFXJHwhqa6ToCc_v2A.png)

2. `console.log('Hi')` 被加入了调用栈：

   ![](https://cdn-images-1.medium.com/max/1600/1*dvrghQCVQIZOfNC27Jrtlw.png)

   3. `console.log('Hi')` 执行了：

      ![](https://cdn-images-1.medium.com/max/1600/1*yn9Y4PXNP8XTz6mtCAzDZQ.png)

   4. 执行完成，`console.log('Hi')` 被移出了调用栈：

      ![](https://cdn-images-1.medium.com/max/1600/1*iBedryNbqtixYTKviPC1tA.png)

   5. `setTimeout(function cb1() { ... })` 也被放入了调用栈：

      ![](https://cdn-images-1.medium.com/max/1600/1*HIn-BxIP38X6mF_65snMKg.png)

   6. `setTimeout(function cb1() { ... })` 执行了。浏览器创建了一个计时器作为 Web API 的一部分。该计时器将负责倒计时：

      ![](https://cdn-images-1.medium.com/max/1600/1*vd3X2O_qRfqaEpW4AfZM4w.png)

   7. `setTimeout(function cb1() { ... })` 执行完毕被移出调用栈：

      ![](https://cdn-images-1.medium.com/max/1600/1*_nYLhoZPKD_HPhpJtQeErA.png)

   8. `console.log('Bye')` 也被放入调用栈：

      ![](https://cdn-images-1.medium.com/max/1600/1*1NAeDnEv6DWFewX_C-L8mg.png)

   9. `console.log('Bye')` 被执行：

      ![](https://cdn-images-1.medium.com/max/1600/1*UwtM7DmK1BmlBOUUYEopGQ.png)

   10. `console.log('Bye')` 被移出调用栈：

       ![](https://cdn-images-1.medium.com/max/1600/1*-vHNuJsJVXvqq5dLHPt7cQ.png)

   11. 至少 5000 ms 以后，计时器倒计时结束，并将回调函数 `cd1` 放入了回调队列：

       ![](https://cdn-images-1.medium.com/max/1600/1*eOj6NVwGI2N78onh6CuCbA.png)

   12. Event Loop 开始转动了，取出了 `cb1` 并将其放入回调队列：

       ![](https://cdn-images-1.medium.com/max/1600/1*jQMQ9BEKPycs2wFC233aNg.png)

   13. `cb1` 执行了，其包含的内容 `console.log('cb1')` 因此被放入了调用栈：

       ![](https://cdn-images-1.medium.com/max/1600/1*hpyVeL1zsaeHaqS7mU4Qfw.png)

   14. `console.log('cb1')` 被执行：

       ![](https://cdn-images-1.medium.com/max/1600/1*lvOtCg75ObmUTOxIS6anEQ.png)

   15. `console.log('cb1')` 从调用栈移出：

       ![](https://cdn-images-1.medium.com/max/1600/1*Jyyot22aRkKMF3LN1bgE-w.png)

   16. `cb1` 也别移出调用栈：

       ![](https://cdn-images-1.medium.com/max/1600/1*t2Btfb_tBbBxTvyVgKX0Qg.png)

   整个过程如下：

   ![](https://cdn-images-1.medium.com/max/1600/1*TozSrkk92l8ho6d8JxqF_w.gif)

## setTimeout(…) 怎么工作的

```js
setTimeout(myCallback, 1000);
```

当我们知道了 Event Loop 工作过程，再来看上面这段代码就会知道，`myCallback` 并不是在 1000 ms 后执行，而是在 1000 ms 后，它被送入了回调队列。此时，回调队列如果由其他事件，它还必须排队等待。

## 任务队列（Job Queue）

ES6 引入了 ”任务队列（Job Queue）“ 的概念，这是一个先进先出的队列，可以有多个任务队列，每个任务队列都是有名字的。V8 引擎可以在不嵌入宿主环境的条件下，使用任务队列来处理 Promise 异步任务。

Promise 任务会被认为是 microtask（微型任务），而 `setTimeout` 则是 macrotask（大型）任务。当存在 microtask 时，会顺序取出 microtask 上的所有任务，放入调用栈执行。以下面的这个取自[知乎问题](https://www.zhihu.com/question/36972010/answer/71338002)的代码为例，我们分析其流程：

```js
setTimeout(function(){console.log(4)},0);
new Promise(function(resolve){
    console.log(1)
    for( var i=0 ; i<10000 ; i++ ){
        i==9999 && resolve()
    }
    console.log(2)
}).then(function(){
    console.log(5)
});
console.log(3);
```

1. `setTimetout` 被放入调用栈执行，浏览器为其回调创建一个计时器。
2. `new Promise()` 被放入调用栈执行。
3. 因此 `console.log(1)` 被放入调用栈执行，输出 `1`。
4. for` 循环被放入调用栈执行，创建的 promise 被放入 microtask 队列。
5. `console.log(2)` 被放入调用栈执行，输出 `2`。
6. `new Promise()` 执行完毕，`console.log(3)` 被放入调用栈执行，输出 `3 `。
7. 此时调用栈为空，优先执行 microtask 上的任务。因此，`console.log(5)` 被放入调用栈执行，输出 `5`。
8. 此间，如果第一步中创建的计时器到时，`setTimeout` 的回调会被放入 macrotask 队列。现在，microtask 队列没有任务，调用栈为空，因此从 macrotask 队列中取出回调送入调用栈执行，输出 `4`。

因此，最终的输出为 `1 2 3 5 4`。

## 5 个使用 async/await 时的建议

1. **Clean Code**：可以尽可能少的使用 `.then` 了。

   ```js
   // `rp` is a request-promise function.
   rp(‘https://api.example.com/endpoint1').then(function(data) {
    // …
   });
   ```

   vs

   ```js
   // `rp` is a request-promise function.
   var response = await rp(‘https://api.example.com/endpoint1');
   ```

2. **错误处理**：

   ```js
   function loadData() {
       try { // Catches synchronous errors.
           getJSON().then(function(response) {
               var parsed = JSON.parse(response);
               console.log(parsed);
           }).catch(function(e) { // Catches asynchronous errors
               console.log(e); 
           });
       } catch(e) {
           console.log(e);
       }
   }
   ```

   vs

   ```js
   async function loadData() {
       try {
           var data = JSON.parse(await getJSON());
           console.log(data);
       } catch(e) {
           console.log(e);
       }
   }
   ```

3. **条件判断**：

   ```js
   function loadData() {
     return getJSON()
       .then(function(response) {
         if (response.needsAnotherRequest) {
           return makeAnotherRequest(response)
             .then(function(anotherResponse) {
               console.log(anotherResponse)
               return anotherResponse
             })
         } else {
           console.log(response)
           return response
         }
       })
   }
   ```

   vs

   ```js
   async function loadData() {
     var response = await getJSON();
     if (response.needsAnotherRequest) {
       var anotherResponse = await makeAnotherRequest(response);
       console.log(anotherResponse)
       return anotherResponse
     } else {
       console.log(response);
       return response;    
     }
   }
   ```

4. **错误堆栈信息**：Promise 链中你捕获到的错误并不知道是哪一步的，而 async/await 就没这个烦恼

   ```js
   function loadData() {
     return callAPromise()
       .then(callback1)
       .then(callback2)
       .then(callback3)
       .then(() => {
         throw new Error("boom");
       })
   }
   loadData()
     .catch(function(e) {
       console.log(err);
   // Error: boom at callAPromise.then.then.then.then (index.js:8:13)
   });
   ```

   vs

   ```js
   async function loadData() {
     await callAPromise1()
     await callAPromise2()
     await callAPromise3()
     await callAPromise4()
     await callAPromise5()
     throw new Error("boom");
   }
   loadData()
     .catch(function(e) {
       console.log(err);
       // output
       // Error: boom at loadData (index.js:7:9)
   });
   ```

5. **调试**：Promise 中进行断点这样的调试比较蛋疼，而在 async/await 中，调试与 JavaScript 的调试方式并无两样。

