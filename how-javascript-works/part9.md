# Web Push

## 架构

```
 +-------+           +--------------+       +-------------+
 |  UA   |           | Push Service |       | Application |
 +-------+           +--------------+       |   Server    |
     |                      |               +-------------+
     |      Subscribe       |                      |
     |--------------------->|                      |
     |       Monitor        |                      |
     |<====================>|                      |
     |                      |                      |
     |          Distribute Push Resource           |
     |-------------------------------------------->|
     |                      |                      |
     :                      :                      :
     |                      |     Push Message     |
     |    Push Message      |<---------------------|
     |<---------------------|                      |
     |                      |                      |
```

通过浏览器提供的推送服务进行推送，服务器发送消息到推送中心，推送中心派发消息到订阅了推送的 UA（user agent）。

## 过程

1. 检测浏览器是否支持 web push：

   ```js
   function supportPush() {
     return ('serviceWorker' in navigator) && ('PushManager' in window)
   }
   ```

2. 注册 service worker，以响应推送事件：

   ```js
   const register = await navigator.serviceWorker.register('./serviceWorker.js')
   ```

   service worker 会在 `push` 事件到来时，展示消息：

   ```js
   self.addEventListener('push', event => {
     const data = event.data.json()
     console.log('This push event has data: ', data)

     const { title, body, icon, badge } = data
     const options = {
       body,
       icon,
       badge
     }
     const notificationPromise = self.registration.showNotification(title, options)
     event.waitUntil(notificationPromise)
   })
   ```

3. 问询用户是否允许进行消息推送：

   ```js
   // 是否允许推送服务
   try {
       await askPermission()
   } catch {
       return
   }
   ```
   
   这里会用到 `Notification.requestPermission()` API：

   ```js
   function askPermission() {
     return new Promise(function(resolve, reject) {
       const permissionResult = Notification.requestPermission(function(result) {
         resolve(result);
       });

       if (permissionResult) {
         permissionResult.then(resolve, reject);
       }
     })
     .then(function(permissionResult) {
       if (permissionResult !== 'granted') {
         throw new Error('We weren\'t granted permission.');
       }
     });
   }
   ```

4. 为当前用户订阅推送：

   ```js
   // 获得当前的推送订阅
   let subscription = await register.pushManager.getSubscription()
   // 如果没有订阅，则为当前用户进行订阅
   if (!subscription) {
       subscription = await subscribeUser(register)
   }
   ```

   ```js
   async function subscribeUser(register) {
     // 获得通信公钥
     const publicKey = await services.getPublicKey()
     // 为 service worker 订阅推送服务
     const subscription = await register.pushManager.subscribe({
       userVisibleOnly: true,
       applicationServerKey: urlB64ToUint8Array(publicKey)
     })
     return subscription
   }
   ```

   `applicationServerKey` 用于推送中心对应用服务的认证。

5. 向应用服务器注册这个订阅，以便后续的消息派发：

   ```js
   const subscribeResponse = await services.register(subscription)
   ```

   这里的 `register` 实际上是我们封装的请求方法：

   ```js
   const services = {
     send(options) {
       return window.fetch('/api/webpush/send', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json'
         },
         body: JSON.stringify(options)
       })
     },

     getPublicKey() {
       return window.fetch('/api/webpush/publicKey').then(resp => resp.text())
     },

     register(subscription) {
       return window.fetch('/api/webpush', {
         method: 'POST',
         headers: {
           'Content-Type': 'application/json',
         },
         body: JSON.stringify(subscription)
       }).then(resp => resp.json())
     }
   }
   ```

   在应用服务器中，我们保存这份订阅：

   ```js
   router.post('/api/webpush', async (ctx, next) => {
     subscriptions.push(ctx.request.body)
     ctx.body = {
       message: '订阅成功',
       applicationServerKey: vapidKeys.publicKey
     }
   }) 
   ```

6. 当应用服务器发送消息时，会将其保存的所有订阅发给推送中心，这里我们使用了 web-push 的 [Node.js 实现](https://github.com/web-push-libs/web-push)：

   ```js
   router.post('/api/webpush/send', async (ctx, next) => {
     // ...
     const payload = {
       body,
       title,
       icon: 'icon',
       badge: 'badge'
     }
     const subscriptionsPromise = subscriptions.map(subscription => {
       return webpush.sendNotification(subscription, JSON.stringify(payload))
     })
     await Promise.all(subscriptionsPromise)
     // ...
   })

   ```


## VPID

VAPID 是公钥和私钥对，私钥保存在应用服务器，公钥则可以被各个客户端共享。使用公钥来和推送中心通信，服务器则使用私钥来和推送中心通信：

![](https://developers.google.com/web/fundamentals/push-notifications/images/svgs/application-server-key-subscribe.svg)

![](https://developers.google.com/web/fundamentals/push-notifications/images/svgs/application-server-key-send.svg)

## Demo

[Demo](https://github.com/yoyoyohamapi/web-push-demo)