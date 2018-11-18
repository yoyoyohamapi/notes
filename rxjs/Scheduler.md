# Scheduler

在 RxJS 中，很多 creator 是同步吐出数据的，这意味着，一旦数据源被订阅，那么直到其吐完所有数据之前，后面的程序都无法运行：

```js
const source$ = of(1, 2, 3)

source$.subscribe(v => console.log('source', v))

console.log('completed!')
```

现实世界中，我们还希望数据能够有节奏的发出，即数据能够以一定速率被吐出。RxJS 为这样的需求提供了 Scheduler，顾名思义，即任务的调度器。再了解 RxJs 的任务调度之前，先可以回顾下 JavaScript 中的任务调度。

## JavaScript 中的任务调度

在 JavaScript 中，我们常用到的任务调度（这里考虑的是浏览器环境）有：

- Sync Task

- Animation Frame
- Macro Task：setTimeout, Events
- Micro Task: Promise

```js
requestAnimationFrame(() => {
  console.log('animation frame')
})

setTimeout(() => {
  console.log('macro task')
}, 0)

Promise.resolve().then(() => {
  console.log('micro task')
})

console.log('sync task')

// sync task
// micro task
// macro task
// animation frame
```

获得如上的输出，是因为，浏览器的 Event Loop 大致是：

```js
while (true) {
  queue = getNextQueue()
  task = queue.pop()
  execute(task)
  
  while (microtaskQueue.hasTasks())
    doMicroTask()
  
  if (isRepaintTime()) {
    animationTasks = animationQueue.copyTasks()
    for (task in animationTasks) {
      doAnimationTask(task)
    }
    
    repaint()
  }
}
```

## RxJs 中的任务调度

基于 JavaScript 的四种任务调度，RxJs 的 Scheduler 也提供了四种基本的任务调度：

- Async Scheduler，类似于 setTimeout，macro task
- Asap Scheduler，As soon as possible，类似于 Promise，micro task
- Queue Scheduler，同步任务调度
- AnimationFrame Scheduler，及 requestAnimationFrame

通过 `observeOn` operator，我们可以传入对应的 scheduler 控制数据源**发出值**的节奏：

```js
console.log('start')

const default$ = source$.subscribe(v => console.log('default', v))

const async$ = source$
  .pipe(observeOn(asyncScheduler))
  .subscribe(v => console.log("async", v))

const asap$ = source$
  .pipe(observeOn(asapScheduler))
  .subscribe(v => console.log("asap", v))

const queue$ = source$
  .pipe(observeOn(queueScheduler))
  .subscribe(v => console.log("queue", v))

const animation$ = source$
  .pipe(observeOn(animationFrameScheduler))
  .subscribe(v => console.log("animation", v))

console.log('end')

// start 
// queue 1
// default 1
// end 
// asap 1
// async 1
// animation 1
```

## Scheduler 职责

RxJs 中，Scheduler 将负责：

- 执行任务
- 控制任务的执行顺序
- 及时地调度任务

在这当中，涉及到三个概念：

- **执行上下文（Execution Context）**：任务执行时的上下文
- **执行策略（Execution Policy）**：约定了任务的调度顺序
- **时钟（Clock）**：如果某个 scheduler 需要在特定时间调度某个任务，它就需要一个时钟

Scheduler 的类被定义为：

```ts
class Scheduler {
  constructor(private SchedulerAction: typof Action);
  
  public actions:Array<SchedulerAction<any>> = [];

	public flush(action: SchedulerAction<any>): void;
}
```

如果 Scheduler 需要时钟，它的构造函数就将是：

```ts
class Scheduler {
  constructor(now: () => number) {
    this.now = now
  }

	private now(): number;
}
```

## Scheduler 的构成

scheduler 可以被表述为: 在一个受控的**执行上下文**中，在一个特定的时间，在各个 **action** 上调度/执行某个 **work**，并且返回一个可以被 **unsubscribe** 的 subscripition。

**Work**

即需要被某个 action 所调度或者说执行的任务，例如：

```ts
const work = state => console.log('state: ', state)
```

**Action**

Action 定义了 work 的执行上下文，并且暴露了一个 schedule 方法：

```ts
const sub = scheduler.schedule(work, delay, state)
sub.unsubscribe()
```

如你所见，schedule 完成：

- 在某个执行上下文中，执行 work
- 返回一个 subscription 对象，该对象可以被 unsubscribe

## 使用场景

**Async Scheduler**:

- 在未来某个时间执行任务

**Asap Scheduler**：

- 在当前的同步任务执行完成后，立即执行

**Animation Frame Scheduler**：

- 当浏览器空闲的时候执行任务

**Queue Scheduler**：

- 顺序计算，尾递归等

比较容易疑惑的是 Queue Scheduler 和不声明 scheduler 的差别，我们看到下面的例子，假定我们的应用会派发若干个 action，action 还会引起其他 action：

```ts
const action$ = new Subject()

const a$ = action$.pipe(filter(({ type }) => type === "first"))
const b$ = a$.pipe(mergeMap(() => of({ type: "second" }, { type: "third" })))
const c$ = action$.pipe(
  filter(({ type }) => type === "second"),
  mergeMap(() => of({ type: "forth" }))
)

action$.subscribe(action => console.log(action.type))
b$.subscribe(action => action$.next(action))
c$.subscribe(action => action$.next(action))

action$.next({ type: "1" })
```

这里，我们期望的 action 顺序是：

```ts
first second third forth
```

但实际上输出的确是：

```
first second forth third
```

我们分析其执行过程：

- 发出 first action
- 调度 first action，输出 first， 派生出发出 second action 及 third action 的 observable
- 调度 second action，输出 second，派生出发出 forth action 的 observable
- 调度 forth action，输出 forth
- 调度 third action，输出 third

如果我们使用 Queue Scheduler 控制 action 发出值的节奏：

```ts
const action$ = new Subject().pipe(observeOn(queueScheduler))
```

就将输出：

```
first second third forth
```

我们分析其执行过程：

- 发出 first action
- 调度 first action，入队
- 此时没有 action，first action 出队，输出 first，派生出发出 second action 及 third action 的 observable
- second action 入队，third action 入队
- 此时没有等待的 action，则 second action 出队，输出 second，派生出发出 forth action 的 observable
- forth action 入队
- 此时没有等待的 action，队首元素 third action 出队，输出 third
- forth action 出队，输出 forth

## VirtualTime Scheduler

除了上面基本的四种 scheduler，RxJs 还提供了两种特殊的 scheduler：**VirtualTime Scheduler** 以及 **Test Scheduler**。

VirtualTime Scheduler 有如下特点：

- 立即开始调度任务
- 将所有的 action 存储先存储到一个数组
- 知道调用 `.flush()` 方法，才开始执行整个流

因此，其使用场景是：立即获得一个流作业的结果。

## Test Scheduler

顾名思义，Test Scheduler 负责的是 Observable 的测试，其继承自 VirtualTime Scheduler：

- 将 marble diagram 转换为 observable
- 调用 `.flush()` 方法时，开始执行

```ts
const obs$ = cold('-a-b-c--------|')
const result$ = obs$.pipe(
  debounceTime(5)
)
const expectedObs$ = '-----c--------|'
expectObservable(result$).toBe(expectedObs$)
```

## 参考资料

- [RxJS schedulers in depth | Michael Hladky | AngularConnect 2018](https://www.youtube.com/watch?v=S1eDh7MonbI&t=343s)
- [PRIMER ON RXJS SCHEDULERS](https://staltz.com/primer-on-rxjs-schedulers.html)



