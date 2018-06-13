# Forwarding Refs

Ref Forwarding 特性能够让组件将它接收到 ref 传递给其（重定向到）子组件。

下面代码中，由 `React.createRef()` 创建的 `ref` 被重定向到了 `React.forwardRef` 

```jsx
const FancyButton = React.forwardRef((props, ref) => (
  <button ref={ref} className="FancyButton">
    {props.children}
  </button>
))

// 现在，ref 被重定向 DOM button
const ref = React.createRef();
<FancyButton ref={ref}>Click me!</FancyButton>
```

此时，`ref.current` 指向了 DOM button。

## 在 HOC 中使用

下面这个 HOC 用来帮我们记录属性日志：

```jsx
function logProps(WrappedComponent) {
  class LogProps extends React.Component {
    componentDidUpdate(prevProps) {
      console.log('old props:', prevProps)
      console.log('new props:', this.props)
    }

    render() {
      return <WrappedComponent {...this.props} />
    }
  }

  return LogProps
}
```

我们为 `FancyButton` 赋予属性日志的功能：

```jsx
class FancyButton extends React.Component {
  focus() {
    // ...
  }

  // ...
}

export default logProps(FancyButton)
```

与 `key` 一样，`ref` 不算一个 prop，因此不会被传递。这意味着 HOC 上的 `ref` 不会引用被包裹的组件：

```jsx
import FancyButton from './FancyButton'

const ref = React.createRef();

// 此时，ref 并不能渗透到 FancyButton，而是停在了 logProps 这个 HOC 上
// 这不是我们所预期的
<FancyButton
  label="Click Me"
  handleClick={handleClick}
  ref={ref}
/>
```

此时，引入 Ref Forwarding 就能帮我们重定向 ref 到正确的位置：

```jsx
function logProps(Component) {
  class LogProps extends React.Component {
    componentDidUpdate(prevProps) {
      console.log('old props:', prevProps)
      console.log('new props:', this.props)
    }

    render() {
      const {forwardedRef, ...rest} = this.props

      // 取到要渗透的 ref，让 ref 渗透到正确的位置
      return <Component ref={forwardedRef} {...rest} />
    }
  }

  // 通过 React.forwardRef，我们将 ref 重定向到 LogProps 的 forwardedRef 上
  return React.forwardRef((props, ref) => {
    return <LogProps {...props} forwardedRef={ref} />
  })
}
```

## 在 DevTools 展示自定义名称

`React.forwardRef` 接收一个 render 函数。React DevTools 根据 render 函数的函数名在开发者工具中展示 ref forwarding 组件。下面的组件将展示为 “ForwardRef”：

```jsx
function logProps(Component) {
  class LogProps extends React.Component {
    // ...
  }

  function forwardRef(props, ref) {
    return <LogProps {...props} forwardedRef={ref} />;
  }

  // Give this component a more helpful display name in DevTools.
  // e.g. "ForwardRef(logProps(MyComponent))"
  const name = Component.displayName || Component.name;
  forwardRef.displayName = `logProps(${name})`;

  return React.forwardRef(forwardRef);
}
```

如果具名这个函数为 myFunction，将展示为 “*ForwardRef(myFunction)”：

```jsx
const WrappedComponent = React.forwardRef(
  function myFunction(props, ref) {
    return <LogProps {...props} forwardedRef={ref} />
  }
)
```

更精细地，我们可以自定义 render function 的 `displayName` 属性，以此展示更详细的信息：

```jsx
function logProps(Component) {
  class LogProps extends React.Component {
    // ...
  }

  function forwardRef(props, ref) {
    return <LogProps {...props} forwardedRef={ref} />;
  }

  // e.g. "ForwardRef(logProps(MyComponent))"
  const name = Component.displayName || Component.name;
  forwardRef.displayName = `logProps(${name})`;

  return React.forwardRef(forwardRef);
}
```

