# Fragments

## 动机

在 React 16 之前，我们想要返回同级的组件，就需要对这些组件包裹一层 `<div></div>` 或者其他 HTML TAg：

```jsx
class App extends React.Component {
  render() {
    return (
    	<div>
      	<ChildA />
        <ChildB />
        <ChildC />
      </div>
    )
  }
}
```

这样做的话，导致页面多渲染了一个 DOM 节点，有可能引起样式出错。

## `React.Fragment`

React 16 考虑到了这个方面，现在我们可以使用 `React.Fragment` 包裹组件，避免多余 DOM 节点的渲染：

```jsx
class App extends React.Component {
  render() {
    return (
    	<React.Fragment>
      	<ChildA />
        <ChildB />
        <ChildC />
      </React.Fragment>
    )
  }
}
```

也有一个简写语法：

```jsx
class App extends React.Component {
  render() {
    return (
    	<>
      	<ChildA />
        <ChildB />
        <ChildC />
      </>
    )
  }
}
```

## 参考资料

- [Fragments](https://reactjs.org/docs/fragments.html)
- [React Fiber v16 Essentials](https://www.udemy.com/react-fiber-v16-essentials/learn/v4/overview)