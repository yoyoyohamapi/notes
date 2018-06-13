# String Return Type

在 React 中，`render` 方法支持直接返回字符串：

```jsx
class App extends React.Component {
  render() {
    return 'Hello World'
  }
}
```

返回的内容不会被包裹上任何的 `<span>` 或者 `<div>`

- [React Fiber v16 Essentials](https://www.udemy.com/react-fiber-v16-essentials/learn/v4/overview)