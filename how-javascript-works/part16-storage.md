# 存储引擎及选择

## 数据模型

- **结构化的数据**：数据按照 table 进行组织，有预定义的 field，例如一些基于 SQL 的数据库管理系统，浏览器中，主要的结构化存储手段是 IndexDB。
- **Key/Value**: value 被唯一的 key 进行索引，也就是一张哈希表，浏览器中的典型应用是 Cache API。
- **字节流**：主要用于文件流或者其他 blob 数据。

## 持久化

- **Session 持久化**：数据只会在同一个应用会话或者浏览器标签中保存。这个机制在浏览器的典型应用是 [Session Storage API](https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage)。
- **设备持久化**：如果数据在某个指定的设备上，想要跨 session 或者在各个 tab 页都有效。这个机制由 [Cache API](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage)。
- **全局持久化**：数据能跨 session、tab、窗口、设备被持久化。这个需求意味着数据不能被存储到对应的设备，只能存储在中央服务上，比如说存储在服务端。

## 浏览器中的数据持久化

在浏览器中进行数据的持久化，需要考虑：

- **浏览器支持**：标准化和流行的 API 更应该被考虑。这些 API 会得到社区更好的支持，有更多的文档和资料。
- **事务性**：持久化存储可能会失败，如果有事务进行回滚和重试就更加健壮。
- **同步/异步**：使用异步的存储 API 不会阻塞主线程（UI 线程），用户体验不会受到太大影响。

| API             | 数据模型   | 持久化  | 浏览器支持 | 事务 | 同步/异步 |
| --------------- | ---------- | ------- | ---------- | ---- | --------- |
| File System     | 字节流     | 设备    | 52%        | No   | 异步      |
| Local Storage   | key/value  | 设备    | 93%        | No   | 同步      |
| Session Storage | key/value  | session | 93%        | No   | 同步      |
| Cookies         | 结构化数据 | 设备    | 100%       | No   | 同步      |
| Cache           | key/value  | 设备    | 60%        | No   | 异步      |
| IndexedDB       | hybrid     | 设备    | 83%        | Yes  | 异步      |

## File System API

这个 API 分为：

- 文件的读取：`File/Blob`、`FileList`、`FileReader`
- 创建和写入：`Blob()`、`FileWriter`
- 目录和文件系统的访问：`DirectoryReader`、`FileEntry/DirectoryEntry`、`LocalFileSystem`

File System API 是一个非标准的 API，各个浏览器的实现也不同，不稳定。使用时可能要慎重考虑。

这个接口当然也不会让你访问用户的文件系统，而是在浏览器 sandbox 中，你有一个 “虚拟设备”。如果你想用获得用户的文件系统，就是撰写 Chrome 扩展的事儿了。

### Requesting a file system

```js
window.requestFileSystem = window.requestFileSystem || window.webkitRequestFileSystem;
window.requestFileSystem(type, size, successCallback, optErrorCallback)
```

调用 `requestFileSystem` 后，浏览器就会创建一个在这个应用的 sandbox 中创建一个 storage。这也意味着应用无法访问其他应用的文件。

File System API 的使用场景有：

- **持久化的文件上传**：能够将上传的文件先存储在本地，然后分块上传。
- **视频游戏，音乐等**：这些应用大量依赖多媒体资源。
- **需要离线存储来做性能优化的音视频编辑器**：数据 blob 会非常大，并且读写频繁。
- **离线视频查看器**：需要预先下载文件存储在本地，边下边播。
- **离线邮箱**：能够将附件存储到本地。

![](https://cdn-images-1.medium.com/max/1600/0*ndU4N8xQF6QEQmSY)

## Local Storage

`localStorage` 允许开发者访问 Document origin 下的 [Storage](https://developer.mozilla.org/en-US/docs/Web/API/Storage) 对象。local storage 存储的对象是跨 session 的，其类似于 session storage，只是没有过期时间。当页面 session 结束（页面关闭），session storage 中的数据将会被清理。

session storage 和 local storage 都是和 origin （协议 + host + port） 相关的。

![](https://cdn-images-1.medium.com/max/1600/0*hxC_NUPNycUBhj-L)

## Session Storage

session storage 会一直存在于页面 session 有效的周期中，在一个新的 tab 页或者窗口中打开页面会造成一个新的会话初始化。

![](https://cdn-images-1.medium.com/max/1600/0*PTDs1BkbMgekizit)

## Cookies

Cookie 是一段用户服务端发向用户浏览器器的数据。浏览器将其存储在本地，并在下一次请求时将它回送给服务端。cookie 常被用来让服务端识别是否两个请求来自于同一个浏览器，它为无状态的 HTTP 协议附上了有状态的信息。

Cookie 的使用场景有：

- **会话管理**：用户的登录信息，购物车，游戏分数等。
- **个性化信息**：用户的配置，主题等。
- **跟踪**：记录分析用户行为。

Cookie 有下面两种：

- **session cookies**: 当客户端关闭后，cookie 会被销毁。但是浏览器会使用 session 恢复策略，这让大多数 session cookie 都持久化了，就好像浏览器从没有被关闭过。
- **持久 cookie**：持久化的 cookie 会要求一个过期时间（Expires）或者最大生命期（Max-Age）参数，来决定 cookie 保存多久。

需要注意的是，敏感和隐私信息不要通过 cookie 在 HTTP 中传输。

## Cache

Cache 接口为 **[Request/Response](http://fetch.spec.whatwg.org/#request)** 对象提供了缓存机制。类似于 woker，它被暴露在了 window 作用域下。

一个 origin 可以有多个具名的 Cache 对象。开发者需要自行决定 Cache 的更新。首先通过 `CacheStorage.open()` 这个 API 打开对应名字花奴陈宁。

同时，开发者还需要负责缓存的清理。各个浏览器对于每个 origin，都有一个 cache 上限。Cache 的配额可以通过 [StorageEstimate](https://developer.mozilla.org/en-US/docs/Web/API/StorageEstimate) 获知。

**CacheStorage**：

- 提供了对具名缓存的管理，供 ServiceWorker 等访问。
- 维护一个 Cache 映射表。

使用 [CacheStorage.open()](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage/open) 来获得 Cache 实例。

```js
caches.open(cacheName).then(cache => {
})
```

使用 [CacheStorage.match()](https://developer.mozilla.org/en-US/docs/Web/API/CacheStorage/match) 来判断给定的 Request 是否命中缓存。

```js
const cachedResponse = caches.match(event.request)
	.catch(() => {
    // 没有命中，就执行请求
    return fetch(event.request)
  }).then(response => {
    // 命中，就去取得缓存
    caches.open('v1').then(cache => {
      // 更新缓存
      cache.put(event.request, response)
    })
    // 这里需要返回新的 response，是因为 cache.put 消费掉了老的 response
    return response.clone()
  }).catch(() => {
    // fallback response
    return caches.match('/sw-test/gallery/myLittleVader.jpg')
  })
```

## IndexedDB

IndexedDB 不仅支持到了数据的离线化存储，还带来了一个数据查询的能力，它可以同时工作在 online 和 offline 下。因此，IndexedDB 非常适合有大量数据需要存储的应用和离线化应用。

IndexedDB 仍然遵循**同源策略**，因此不能跨域访问 IndexedDB。

IndexDB 提供了可在多数上下文（甚至包括 WebWorker）中使用的异步 API。IndexedDB 具有下面这些特点：

- **IndexedDB 存储了键值对**：value 可以是复杂的结构化对象，key 则可以是这些对象的属性。你可以用对象的任何属性创建索引，以加快查询效率。key 也可以是二进制对象。
- **IndexedDB 构建在事务性的数据模型**：你无法再事务之外执行命令或者打开 cursor，事务是自动提交的，也不是手动提交的。
- **IndexedDB API 多数都是异步的**：你无法 “store” 值到数据库，或者从数据库中 “retrieve” 值。取而代之的是，你是在 “request” 相关的数据库操作，并且通过事件机制来判断操作完成与否。
- **IndexedDB 大量使用了 request**：request 对象包含 `onsuccess` 和 `onerror` 两个属性，以及 `readyState` 、`result`、`errorCode` 等属性来描述当前 request 对象的状态。
- **IndexedDB 是面向对象的**：IndexedDB 不是一个使用表来描述行列的关系型数据库。这会让我们在使用编程模型上作出考虑。
- **IndexedDB 没有使用 SQL**：其使用的基于索引的查询，这涉及到了cursor，以及结果集的迭代。
- **遵循同源策略**

### IndexedDB 的限制

- **国际化排序**：数据库在某种语言下可能无法按照特定的国际化字符顺序进行排序。这就需要读取顺序后，手动进行一个排序。
- **同步化**：API 无法像 server 端那样有同步的 API。
- **全字匹配**：IndexedDB 并没有提供诸如 SQL 中 LIKE 这样的操作子，

另外，浏览器会在下面这些场景下清除 IndexedDB 的数据：

- **用户手动清除**：用户通过浏览器的清空浏览数据清理。
- **浏览器以隐私模式工作**
- **磁盘或者配额达到上限**
- **数据被破坏**

![](https://cdn-images-1.medium.com/max/1600/0*kGDQYE70_z58D7na)

## 如何选择

- 对于离线应用，使用 Cache API。
- 对于应用状态，用户数据的存储，使用 IndexedDB。这让用户在不支持 Cache API 的浏览器下，也能获得离线应用的能力。