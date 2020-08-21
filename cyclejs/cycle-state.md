# Cycle.js 状态管理模型

## 分形（fractal）

当今前端领域，最流行的状态管理模型毫无疑问是 redux，但遗憾的是，redux 并不是一个分形架构。什么是分形架构：

> 如果子组件能够以同样的结构，作为一个应用使用，这样的结构就是分形架构。

在分形架构下，每个应用都组成为更大的应用使用，而在非分形架构下，应用往往依赖于一个统揽全局的**协调器（orchestrators）**，各个组件并不能以同样的结构当做应用使用，而是统一接收这个协调器协调。例如，redux 只是聚焦于状态管理，而不涉及组件的视图实现，无法构成一个完整的应用闭环，因此 redux 不是一个分形架构，在 redux 中，协调器就是全局 `Store` 。

![Redux diagram](https://staltz.com/img/redux-unidir-ui-arch.jpg)

我们再看下 redux 灵感来源 —— Elm：

![Model-View-Update diagram](https://staltz.com/img/mvu-unidir-ui-arch.jpg)

在 Elm 架构下，每个组件都有一个完整的应用闭环：

- 一个 Model 类型
- 一个 Model 的初始实例
- 一个 View 函数
- 一个 Action type 以及对应的更新函数

因此，Elm 就是分形架构的，每个 Elm 组件也是一个 Elm 应用。

## Cycle.js

分形架构的好处显而易见，就是复用容易，组合方便，Cycle.js 推崇的也是分形架构。其将应用抽象为了一个纯函数 `main(sources)`，该函数接收一个 `sources` 参数，用来从外部环境获得诸如 DOM、HTTP 这样的副作用，再输出对应的 `sinks` 去影响外部环境。

![img](https://cycle.js.org/img/cycle-nested-frontpage.svg)

基于这种简单而直接的抽象，Cycle.js 容易做到分形，即每个 Cycle.js 应用（每个 `main` 函数）可以组合为更大的 Cycle.js 应用：

![nested components](https://cycle.js.org/img/nested-components.svg)在分形体系下，通过 `run` API，能驱动任何 Cycle.js 应用运行，无论它是一个简单的 Cycle.js 应用，还是一个嵌套复合的 Cycle.js 应用。

```js
import {run} from '@cycle/run'
import {div, label, input, hr, h1, makeDOMDriver} from '@cycle/dom'

function main(sources) {
  const input$ = sources.DOM.select('.field').events('input')

  const name$ = input$.map(ev => ev.target.value).startWith('')

  const vdom$ = name$.map(name =>
    div([
      label('Name:'),
      input('.field', {attrs: {type: 'text'}}),
      hr(),
      h1('Hello ' + name),
    ])
  )

  return { DOM: vdom$ }
}

run(main, { DOM: makeDOMDriver('#app-container') })
```

## Cycle.js 的状态管理

### 响应式

上面我们提到，Cycle.js 推崇的是分形应用结构，因此，redux 这样的状态管理器就不是 Cycle.js 愿意使用的，它会让全局只有一个 redux 应用，而不是多个可拆卸的 Cycle.js 分形应用。基于此，若要引入状态管理模型，其设计应当不改变 Cycle.js 应用的基本结构：从外部世界接收 `sources`，输出 `sinks` 到外部世界。

另外，由于 Cycle.js 是一个响应式前端框架，那么状态管理仍然保持是响应式的，即以 stream/observable 为基础。如果你熟悉响应式编程，基于 Elm 的理念，以 RxJs 为例，我们可以很轻松的实现一个状态管理模型：

```js
const action$ = new Subject()

const incrReducer$ = action$.pipe(
	filter(({type}) => type === 'INCR'),
  mapTo(function incrReducer(state) {
    return state + 1
  })
)

const decrReducer$ = action$.pipe(
	filter(({type}) => type === 'DECR'),
  mapTo(function decrReducer(state) {
    return state - 1
  })
)

const reducer$ = merge(incrReducer$, decrReducer$)

const state$ = reducer$.pipe(
  scan((state, reducer) => reducer(state)),
	startWith(initState),
  shareReplay(1)
)
```

基于上述的前提，Cycle.sj 状态管理模型的基础设计也跃然纸上：

- 将状态源 `state$` 放入 `sources ` 中，输入给 Cycle.js 应用
- Cycle.js 应用则将 **reducer$** 放入 `sinks` 中，输出到外部世界

> 参看 [`@cycle/state` 的 `withState` 的源码](https://github.com/cyclejs/cyclejs/blob/master/state/src/withState.ts)，其响应式状态管理模型实现亦大致如上。

在实际实现中，Cycle.js 通过 `@cycle/state` 暴露的 `withState`  来为 Cycle.js 注入状态管理模型：

```js
import {withState} from '@cycle/state'

function main(sources) {
  const state$ = sources.state.stream
  const vdom$ = state$.map(state => /*render virtual DOM*/)
  
  const reducer$ = xs.periodic(1000)
  .mapTo(function reducer(prevState) {
    // return new state
  })
  
  const sinks = {
    DOM: vdom$,
    state: reducer$
  }
  return sinks
}

const wrappedMain = withState(main)

run(wrappedMain, drivers)
```

在思考了如何让 Cycle.js 引入状态管理模型后仍然保持分形后，我们还要再状态管理模型中解决下面这些问题：

- 如何声明应用初始状态
- 应用如何读取以及修改某个状态

### 初始化状态

为了遵循响应式，我们可以声明一个 `initReducer$`，其默认发出一个 `initReducer`，在这个 reducer 中，直接返回组件的初始状态：

```js
const initReducer$ = xs.of(function initReducer(prevState) {
  return { count:0 }
})

const reducer$ = xs.merge(initReducer$, someOtherReducer$);

const sinks = {
  state: reducer$,
};
```

### 使用洋葱模型传递状态

实际项目中，应用总是由多个组件组成，并且组件间还会存在层级关系，因此，还需要思考：

1. 怎么传递状态到组件
2. 怎么传递 reducer 到外部

假定我们的状态树是：

```js
const state = {
  visitors: {
    count: 300
  }
}
```

假定我们的组件需要 `count` 状态，就有两种设计思路：

（1）在组件中直接声明要摘取的状态，如何处理子状态变动：

```js
function main(sources) {
  const count$ = sources.state.visitors.count
  const reducer$ = incrAction$.mapTo(function incr(prevState) {
    return prevState + 1
  })
  
  return {
    state: {
      visitors: {
        count: reducer$
      }
    }
  }
}
```

（2）保持组件的纯净，其获得的 `state$` ，输出的 `reducer$` 不用考虑当前状态树形态，二者都只相对于组件自己：

```js
function main(sources) {
  const state$ = sources.state
  const reducer$ = incrAction$.mapTo(function incr(prevState) {
    return prevState + 1
  })
  
  return {
    state: reducer$
  }
}
```

两种方式各有好处，第一种方式更加灵活，适合层级嵌套较深的场景。第二种则让组件逻辑更加内聚，拥有更高的组件自治能力，在简单场景下可能表现得更加直接。这里我们首先探讨第二种传递状态方式。

在第二种状态传递方式下，我们要将 `count` 传递给对应的组件，就需要**从外到内**逐层的剥开状态，直到拿到组件需要的状态：

```js
stateA$ // Emits object `{visitors: {count: 300}}}`
stateB$ // Emits object `{count: 300}`
stateC$ // Emits object `300`
```

而组件输出 reducer 时，则需要**由内到外**进行 reduce：

```js
reducerC$ // Emits function `count => count + 1`
reducerB$ // Emits function `visitors => ({count: reducerC(visitors.count)})`
reducerA$ // Emits function `appState => ({visitors: reducerB(appState.visitors)})`
```

这形成了一个类似**洋葱**（cycle state 的前身正是 cycle-onionify）的状态管理模型：我们由外部世界开始，层层剥开外衣，拿到状态；在逐层进行 reduce 操作，由内到外进行状态更新：

![Diagram](https://github.com/staltz/cycle-onionify/raw/master/diagram.png)

具体看一个例子，假定父组件获得如下的状态：

```js
{
  foo: string,
  bar: number,
  child: {
    count: number,
  },
}
```

其中，`child` 子状态是其子组件需要的状态，此时，洋葱模型下就要考虑：

- 将 `child` 从状态树中剥离，传递给子组件
- 收集子组件输出的 `reducer$`，合并后继续向外输出

首先，我们需要使用 `@cycle/isolate` 隔离子组件，其暴露了一个 `isolate(component, scope)` 函数，该函数接受两个参数：

- `component`：需要隔离的组件，即一个接受 `sources` 并返回 `sinks` 的函数
- `scope`：组件被隔离到的 scope。scope 决定了 DOM，state 等外部环境如何划分其资源到组件

该函数最终将返回隔离组件输出的 `sinks`。获得了子组件的 `reducer$` 之后，还要与父组件的 `reducer$` 进行合并，继续向外抛出。

例如下面的代码中，`isolate(Child, 'child')(sources)` 将 `Child` 组件隔离到了名为 `child` 的 scope 下，因此， `@cycle/state` 能够知道，要从状态树上选出名为 `child` 的状态子树给 `Child` 组件。

```js
function Parent(sources) {
  const state$ = sources.state.stream; // emits { foo, bar, child }
  const childSinks = isolate(Child, 'child')(sources);
  
  const parentReducer$ = xs.merge(initReducer$, someOtherReducer$);
  const childReducer$ = childSinks.state;
  const reducer$ = xs.merge(parentReducer$, childReducer$);
  
  return {
    state: reducer$
  }
}
```

另外，为了保证父组件不存在时，子组件能够独立运行的能力，需要在子组件中进行识别这种场景（`prevState === undefined`），并返回对应状态：

```js
function Child(sources) {
  const state$ = sources.state.stream; // emits { count } 
  
  const defaultReducer$ = xs.of(function defaultReducer(prevState) {
    if (typeof prevState === 'undefined') {
      return { count: 0}
    } else {
      return prevState
    }
  })
  
  // 这里，reducer 将处理 { count } state
  const reducer$ = xs.merge(defaultReducer$, someOtherReducer$);
  
  return {
    state: reducer$
  }
}
```

好的习惯是，每个组件我们都声明一个 `defaultReducer$`，用来照顾其单独使用时的场景，以及存在父组件时的场景。

> 关于组件隔离的来由，可以参看：[Cycle.js Components 一节](https://cycle.js.org/components.html#components)

### 使用 Lens 机制传递状态

在洋葱模型中，数据通过父组件传递到子组件，这里父组件仅仅能够从自身的状态树摘取一棵子树给子组件，因此，这个模型在灵活性上受到了一些限制：

- **个数上**：只能传递一个子状态
- **规模上**：不能传递整个状态
- **I/O 上**：只能读取，不能修改状态

如果你有下面的需求，这种模式就难以胜任：

- 组件需要多个状态，例如需要获得 `state.foo` 及 `state.status`
- 父子组件需要访问同一部分状态，例如父组件和子组件需要获得 `state.foo` 
- 当子组件的状态变动后，需要联动修改状态树，而不只是通过 `reducer$` 修改其自身状态

为此，就需要考虑使用上文中我们提到的第一种状态共享方式。我们给到的多少有些粗糙，Cycle.js 则是引入了 lens 机制来处理洋葱模型无法照顾到的这些场景，顾名思义，这能让组件拥有**洞察（读取）**并且**更改（写入）**状态的能力。

简单来说，lens 通过 getter/setter 定义了对某个数据的读写。

为了实现通过 lens 来读写状态，Cycle.js 让 `isolate` 在隔离组件实例时，接受组件自定义的 lens 作为 scope selector，以让 `@cycle/state` 组件要如何读取以及修改状态。

```js
const fooLens = {
  get: state => state.foo,
  set: (state, childState) => ({...state, foo: childState})
};

const fooSinks = isolate(Foo, {state: fooLens})(sources);
```

上面代码中，通过自定义 lens，组件 `Foo` 能够获得状态树上的 `foo` 状态，而当 `Foo` 修改了 `foo` 后，将联动修改状态树上的 `foo` 状态。

### 处理动态列表

渲染动态列表是前端最常见的需求之一，在 Cycle.js 引入状态管理之前，这一直是 Cycle.js 做不好的一个点，甚至 André Staltz 还专门开了一篇 [issue](https://github.com/cyclejs/cyclejs/issues/312) 来讨论如何更在 Cycle.js 中更优雅的处理动态列表。

现在，基于上述的状态管理模型，只需要一个 `makeCollection` API，即可在 Cycle.js 中，创建一个动态列表：

```js
function Parent(sources) {
  const array$ = sources.state.stream;

  const List = makeCollection({
    item: Child,
    itemKey: (childState, index) => String(index),
    itemScope: key => key,
    collectSinks: instances => {
      return {
        state: instances.pickMerge('state'),
        DOM: instances.pickCombine('DOM')
        .map(itemVNodes => ul(itemVNodes))
        // ...
      }
    }
  });
  
  const listSinks = List(sources);
  
  const reducer$ = xs.merge(listSinks.state, parentReducer$);
  
  return {
    state: reducer$
  }
}
```

看到上面的代码，基于 `@cylce/state` 创建一个动态列表，我们需要告诉 `@cycle/state`：

- 列表元素是什么
- 每个元素在状态中的位置
- 每个元素的 scope

- 列表的 `reducer$`：`instances.pickMerge('state')`，其约等于:
  -  `xs.merge(instances.map(sink => sink.state))`
- 列表的 `vdom$`：`instances.pickCombine('DOM')`，其约等于：
  - `xs.combine(instances.map(sink => sink.DOM))`

**新增列表元素**只需要在列表容器的 `reducer$` 中，为数组新增一个元素即可：

```js
const reducer$ = xs.periodic(1000).map(i => function reducer(prevArray) {
  return prevArray.concat({count: i})
})
```

**删除元素**则需要子组件在删除行为触发时，将其状态标识为 `undefiend`，Cycle.js 内部会据此从列表数组中删除该状态，进而删除子组件及其输出的 sinks：

```js
function Child(sources) {
  const deleteReducer$ = deleteAction$.mapTo(function deleteReducer(prevState) {
    return undefined;
  })
  
  const reducer$ = xs.merge(deleteReducer$, someOtherReducer$)
  
  return {
    state: reducer$
  }
}

```

## 总结

Cycle.js 相比较于前端三大框架（Angular/React/Vue）来说，算是小众的不能再小众的框架，学习这样的框架并不是为了**标新立异**，考虑到你的团队，你也很难在大型工程中将它作为支持框架。但是，这不妨碍我们从 Cycle.js 的设计中获得启发和灵感，它多少能让你感受到：

- 也许我们的应用就是一个和外部世界打交道的环
- 什么是分形
- 响应式程序设计的魅力
- 什么是 lens 机制？如何在 JavaScript 应用中使用 lens
- ...

另外，Cycle.js 的作者 André Staltz 也是一个颇具个人魅力和表达能力的开发者，推荐你关注他的：

- André Staltz 的博客：https://staltz.com/
- André Staltz 在 egghead.io 上的 RxJs 和 Cycle.js 教程，你不仅能学到 API，还能学到框架设计思路：https://egghead.io/instructors/andre-staltz
-  André Staltz 参加并演讲的一系列会议：https://www.youtube.com/results?search_query=andre+staltz

最后，不要盲目崇拜，只要疯狂学习和探索。

## 参考资料

- [UNIDIRECTIONAL USER INTERFACE ARCHITECTURES](https://staltz.com/unidirectional-user-interface-architectures.html)

- [Cycle State](https://cycle.js.org/api/state.html)

- [An Introduction Into Lenses In JavaScript](https://medium.com/javascript-inside/an-introduction-into-lenses-in-javascript-e494948d1ea5)

