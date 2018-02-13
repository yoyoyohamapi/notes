## Service Worker

![](https://cdn-images-1.medium.com/max/1600/1*oOcY2Gn-LVt1h-e9xOv5oA.jpeg)

Service Worker 有如下几个特点：

- 运行在自己的全局上下文中
- 无法访问 DOM
- 不耦合某个特定页面

Service Worker 最大好处是让我们的 web 应用支持离线体验。

## 生命期

Service Worker 的生命期是与页面隔离的，分为：

- 下载（Download）
- 安装（Installation）
- 激活（Activation）

### 下载

即加载包含有 ServiceWorker 的 `.js` 文件。

## 安装

注册 Service Worker：

```js
// 检查当前环境是否支持 Service Worker API
if ('serviceWorker' in navigator) {
  window.addEventListener('load', function() {
    navigator.serviceWorker.register('/sw.js').then(function(registration) {
      // 注册成功
      console.log('ServiceWorker registration successful');
    }, function(err) {
      // 注册失败
      console.log('ServiceWorker registration failed: ', err);
    });
  });
}
```

一个需要关注的细节是，`register()` 函数接收的 Service Worker 文件的位置。在这个例子中，worker 文件存储在域的根目录下，这就意味着 worker 的作用域将是整个源。换言之，worker 能够收到每个该域名下的 `fetch` 事件。如果 worker 文件的地址是 `/example/sw.js`，那么 worker 的就只能看到 `/example` 下的 `fetch` 事件。

在安装阶段，最好能对静态资源进行加载和缓存。一旦静态资源被成功缓存，Service Worker 的安转就完成了，否则 Service Worker 会进行重试。

你可能会问，为什么注册要放在加载之后？

首先，我们的应用一开始是没有任何 service worker 的，浏览器也无法提前知道将来是否有 service worker 需要被加载。如果需要安转 service worker，则浏览器需要为这些线程耗费额外的 CPU 和 内存，否则，浏览器只需要进行页面渲染。显然，我们应当尽快让页面渲染完成，完成之后再花时间安转 service worker。

## 激活

接下来，就是 worker 的激活了，一旦 worker 完成了激活，它就能控制所有属于它作用域下的一切：

1. 当网络请求或者页面发出消息是，worker 将操纵 fetch 和 message 事件
2. 它也会被终止以节约内存

![](https://cdn-images-1.medium.com/max/1600/1*mVOrpKC9pFTMg4EXPozoog.png)

## 在 Service Worker 中监听安装事件

在安转期间，worker 需要完成

1. 开启缓存
2. 缓存文件
3. 确定所有的资源文件都被缓存

```js
var CACHE_NAME = 'my-web-app-cache';
var urlsToCache = [
  '/',
  '/styles/main.css',
  '/scripts/app.js',
  '/scripts/lib.js'
];

self.addEventListener('install', function(event) {
  event.waitUntil(
    caches.open(CACHE_NAME)
      .then(function(cache) {
        console.log('Opened cache');
        return cache.addAll(urlsToCache);
      })
  );
});
```

## 在运行时缓存请求

当 Service Worker 安转后，如若用户跳转到其他页面，或者在当前页面进行刷新工作，Service Worker 将受到 fetch 事件。下面的代码展示了如何返回缓存的资源或者执行完新的请求后缓存新的内容：

```js
self.addEventListener('fetch', function(event) {
    event.respondWith(
    	caches.match(event.request)
        .then(function(response) {
            // 如果命中缓存，则直接返回缓存
            if (response) {
                return response;
            }
            
            // clone request
            // 一个请求是一个流对象，并且只能被消费一次，
            // 由于缓存判定时已经消费了一次，我们的 fetch 就需要一份新的拷贝
            var fetchRequest  = event.request.clone();
            
            // 执行 fetch，响应成功时进行缓存
            return fetch(fetchRequest).then(
            	function(response) {
                    if (!respone || response.status !== 200 || response.type !== basic) {
                        return response;
                    }
                    
                    // 出于同样的原因，我们 clone 一份响应
                    var responseToCache = response.clone();
                    
                    caches.open(CACHE_NAME)
                        .then(function(cache) {
                            cache.put(event.request, responseToCache);
                        })
                    
                    return response;
                }
            )
            
        })
    )
})
```

## Service Worker 的更新
如果一个用户访问了你的网站，浏览器会重新下载包好了 service worker 代码的 `.js` 文件。

如果新下载的文件与之前的文件存在差异，浏览器会启动新的 service worker。新 worker 的 install 事件将被触发，但是，控制页面的 worker 仍然是老的 worker，新 worker 的状态为 `wating`。

如果当前应用的页面都关闭了，老的 worker 就会被 kill，而把控制权转让给新的 worker，新 worker 的 activate 事件会被触发。

之所以要这样做，是为了避免不同标签页同时存在不同版本的 web 应用，产生 bug。

## 删除缓存中的数据

```js
self.addEventListener('activate', function(event) {
    var cacheWhiteList = ['page-1', 'page-2'];
    
    event.waitUntil(
        caches.keys().thne(function(cacheName) {
            return Promise.all(
                cacheNames.map(function(cacheName) {
                    if (cacheWhiteList.indexOf(cacheName) === -1) {
                        return caches.delete(cacheName);
                    }
                })
            )
        })
    )
})
```

## 使用 HTTPS

由于 Service Worker 可以拦截请求，伪造响应，因此，使用 HTTPS 能防止 [**MITM（man-in-the-middle attack）**](https://www.wikiwand.com/en/Man-in-the-middle_attack)：

![](https://upload.wikimedia.org/wikipedia/commons/thumb/e/e7/Man_in_the_middle_attack.svg/520px-Man_in_the_middle_attack.svg.png)