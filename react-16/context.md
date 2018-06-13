# Context

## 动机

React 的核心理念是单向数据流，组件属性都是由上自下地进行传递的。通常，我们的应用会有一些全局属性的概念，或者说全局配置，例如主题、domain 信息、国际化 locale 等。仅仅依靠单向数据流传递这些配置或者对象，当组件层级较深时，就会非常冗长，传递途中如果某天少了几个组件，传递也可能中断。

```jsx
class App extends React.Component {
  render() {
    return <Toolbar theme="dark" />
  }
}

function Toolbar(props) {
  // “朴素” 的单向数据流传递 theme 信息
  return (
    <div>
      <ThemedButton theme={props.theme} />
    </div>
  )
}

function ThemedButton(props) {
  return <Button theme={props.theme} />
}
```

## Context 模式

对于全局共享信息，React 使用了 Context 模式，即为**应用维护一个应用上下文**。所谓上下文，也是指的应用存在期间的周边环境，为此，还需要定义两个概念：

- 上下文提供者 **Provider**：将共享的上下文暴露给需求方
- 上下文消费者 **Consumer**：共享上下文的需求方，按需消费。消费方式并不是将 context 注入到 props 中，而是通过 render props 进行消费。

```jsx
// 创建一个主题上下文
const ThemeContext = React.createContext('light');

class App extends React.Component {
  render() {
    // 使用 Provider 包裹组件树，那么该主题配置能够被组件树中的所有组件共享
    // 此时，我们还修改了当前的主题为 dark
    return (
      <ThemeContext.Provider value="dark">
        <Toolbar />
      </ThemeContext.Provider>
    );
  }
}

// Toolbar 没有被声明为消费者，那么 theme context 被继续向下传递
function Toolbar(props) {
  return (
    <div>
      <ThemedButton />
    </div>
  );
}

function ThemedButton(props) {
  // ThemedButton 被声明为了 Consumer，通过 render props，拿到了 theme context
  return (
    <ThemeContext.Consumer>
      {theme => <Button {...props} theme={theme} />}
    </ThemeContext.Consumer>
  );
}
```

## 动态上下文

有时候，我们还会修改到全局配置（上下文），比如我们有两幅主题：

**theme-context.js**

```jsx
export const themes = {
  light: {
    foreground: '#000000',
    background: '#eeeeee',
  },
  dark: {
    foreground: '#ffffff',
    background: '#222222',
  },
}

export const ThemeContext = React.createContext(
  themes.dark // default value
)
```

**themed-button**

```jsx
import {ThemeContext} from './theme-context'

function ThemedButton(props) {
  return (
    <ThemeContext.Consumer>
      {theme => (
        <button
          {...props}
          style={{backgroundColor: theme.background}}
        />
      )}
    </ThemeContext.Consumer>
  )
}

export default ThemedButton
```

此时，我们再工具栏添加了切换主题的能力：

**app.js**

```jsx
import {ThemeContext, themes} from './theme-context'
import ThemedButton from './themed-button'

function Toolbar(props) {
  return (
    <ThemedButton onClick={props.changeTheme}>
      Change Theme
    </ThemedButton>
  )
}

class App extends React.Component {
  constructor(props) {
    super(props)
    this.state = {
      theme: themes.light,
    }

    this.toggleTheme = () => {
      this.setState(state => ({
        theme:
          state.theme === themes.dark
            ? themes.light
            : themes.dark,
      }))
    }
  }

  render() {
    // 这里我们将设定了主题配置 Provider 的主题来源为 App 的 theme state
    return (
      <Page>
        <ThemeContext.Provider value={this.state.theme}>
          <Toolbar changeTheme={this.toggleTheme} />
        </ThemeContext.Provider>
        <Section>
          <ThemedButton />
        </Section>
      </Page>
    )
  }
}

ReactDOM.render(<App />, document.root)
```

## 在嵌套组件中更新上下文

有时候，我们也有需求在层级很深的组件中，也想要修改上下文，此时，我们创建上下文时也暴露一个修改接口：

**theme-context.js**

```jsx
export const ThemeContext = React.createContext({
  theme: themes.dark,
  toggleTheme: () => {},
})
```

此时，主题的上下文不仅有主题，还有对主题的修改方式。

**theme-toggler-button.js**

```jsx
import {ThemeContext} from './theme-context'

function ThemeTogglerButton() {
  // 作为主题配置的消费者，ThemeTogglerButtong 现在不仅
  // 接收到主题配置，还收到了主题配置的修改方式
  return (
    <ThemeContext.Consumer>
      {({theme, toggleTheme}) => (
        <button
          onClick={toggleTheme}
          style={{backgroundColor: theme.background}}>
          Toggle Theme
        </button>
      )}
    </ThemeContext.Consumer>
  )
}

export default ThemeTogglerButton
```

**app.js**

```jsx
import {ThemeContext, themes} from './theme-context';
import ThemeTogglerButton from './theme-toggler-button';

class App extends React.Component {
  constructor(props) {
    super(props);

    this.toggleTheme = () => {
      this.setState(state => ({
        theme:
          state.theme === themes.dark
            ? themes.light
            : themes.dark,
      }));
    };

    // 此时 App 的 state 中容纳的主题配置页包含了一个 toggleTheme 方法
    this.state = {
      theme: themes.light,
      toggleTheme: this.toggleTheme,
    };
  }

  render() {
    // 传入主题、主题修改方式
    return (
      <ThemeContext.Provider value={this.state}>
        <Content />
      </ThemeContext.Provider>
    );
  }
}

function Content() {
  return (
    <div>
      <ThemeTogglerButton />
    </div>
  );
}

ReactDOM.render(<App />, document.root);
```

## 消费多个上下文

牢记每个上下文都有 Provider 和 Consumer 的两个角色

```jsx
// 主题上下文
const ThemeContext = React.createContext('light');

// 登录用户的上下文
const UserContext = React.createContext({
  name: 'Guest',
});

class App extends React.Component {
  render() {
    const {signedInUser, theme} = this.props;

    // 多个上下文提供者
    return (
      <ThemeContext.Provider value={theme}>
        <UserContext.Provider value={signedInUser}>
          <Layout />
        </UserContext.Provider>
      </ThemeContext.Provider>
    );
  }
}

function Layout() {
  return (
    <div>
      <Sidebar />
      <Content />
    </div>
  );
}

// ProfilePage 需要消费多个上下文
function Content() {
  return (
    <ThemeContext.Consumer>
      {theme => (
        <UserContext.Consumer>
          {user => (
            <ProfilePage user={user} theme={theme} />
          )}
        </UserContext.Consumer>
      )}
    </ThemeContext.Consumer>
  );
}
```

## 在生命期内访问上下文

React 并没有将上下文注入到生命周期钩子中，上下文消费者拿到上下文后，可以作为属性传递给组件，从而在组件的生命周期中访问。

```jsx
class Button extends React.Component {
  componentDidMount() {
    // ThemeContext 可以通过 this.props.theme 取到
  }

  componentDidUpdate(prevProps, prevState) {
    // 上一次主题上下文是 prevProps.theme
    // 下一次主题上下文是 this.props.theme
  }

  render() {
    const {theme, children} = this.props
    return (
      <button className={theme ? 'dark' : 'light'}>
        {children}
      </button>
    )
  }
}

export default props => (
  <ThemeContext.Consumer>
    {theme => <Button {...props} theme={theme} />}
  </ThemeContext.Consumer>
)
```

## 使用 HOC 来更容易地消费上下文

上下文的核心就在于被多个组件消费，如果每次需要消耗到上下文时，我们都需要撰写 `<Context.Consumer />` 的话，就太啰嗦了，我们可以使用 HOC 来降低复杂度：

```jsx
const ThemeContext = React.createContext('light');

// 创建一个 withTheme HOC 
export function withTheme(Component) {
  // 返回一个能够消费到主题配置的组件
  return function ThemedComponent(props) {
    // 将 context 作为属性传给需要的组件
    return (
      <ThemeContext.Consumer>
        {theme => <Component {...props} theme={theme} />}
      </ThemeContext.Consumer>
    )
  }
}
```

接着，使用就方便很多了：

```jsx
function Button({theme, ...rest}) {
  return <button className={theme} {...rest} />
}

const ThemedButton = withTheme(Button)
```

## Caveats

由于上下文直接比较的引用来决定是否要重新渲染，因此要留意一下可能引起的不必要的渲染，比如下面这个例子中，每次 App 渲染，都会传给 Provider 一个新的引用，导致 Provider 下的 Consumer 都重新渲染：

```jsx
class App extends React.Component {
  render() {
    return (
      <Provider value={{something: 'something'}}>
        <Toolbar />
      </Provider>
    )
  }
}
```

因此，我们应当将传入的配置提升到父组件状态进行维护：

```jsx
class App extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      value: {something: 'something'}
    }
  }

  render() {
    return (
      <Provider value={this.state.value}>
        <Toolbar />
      </Provider>
    )
  }
}
```

