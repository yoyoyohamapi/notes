# React, Inline Functions, and Performance

> [原文地址](https://cdb.reacttraining.com/react-inline-functions-and-performance-bdff784f5578)

## 3 个使用 inline function 的场景

### DOM component event handler

```jsx
<button
	onClick={(() => this.setState()}
>click</button>
```

这种写法让 Dom component 及其 event handler 高度内聚，也不用担心 inline function 的性能问题，因为 `button` component 不会是一个 `PureComponent`，因此不用担心 `shouldComponentUpdate` 中的引用比较问题。

### 自定义 event handler

```jsx
<Sidebar onToggle={(isOpen) => {
  this.setState({ sidebarIsOpen: isOpen })
}}/>
```

如果 `Sidebar` 是一个 `PureComponent`，那么，这段代码的 inline function 会造成 `Sidebar` 的重新渲染。

事实上，我们将某个 prop 放入 `shouldComponentUpdate` 中，只会出于下面的两个原因：

1. 使用这个 prop 进行 render
2. 使用这个 prop 在 `componentWillReceiveProps`、`componentDidUpdate`、`componentWillUpdate` 等生命周期中执行副作用。

绝大多数的自定义 event handler 不满足这两个条件，因此使用 `PureComponent` 就显得有点 over。

### render prop

```jsx
<Route
  path=”/topic/:id”
  render={({ match }) => (
    <div>
      <h1>{match.params.id}</h1>}
    </div>
  )
/>
```

render prop 通过一个 render 函数来完成组件间的属性或者状态共享。

```jsx

const App = (props) => (
  <div>
    <h1>Welcome, {props.name}</h1>
    <Route path=”/” render={() => (
      <div>
        <h1>Hey, {props.name}, let’s get started!</h1>
      </div>
    )}/>
  </div>
)
```

由于 render prop 是一个 inline function，所以使用了 render prop 的组件也没必要使用 `PureComponent`，以避免 diff 造成的重新渲染。



