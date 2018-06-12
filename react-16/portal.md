# Portal

## 不使用 Portal 的 Modal

在 React 16 以前，我们通常这样创建一个 Modal 组件：

```jsx
class Modal extends React.Component {
  render() {
    return this.props.visible ? (
       <div>
        {this.props.children}
      </div>
    ) : null
  }
}
```

并需要将它放到某个特定的父组件中使用，这些父组件往往具有 `overflow:hidden` 或者 `z-index` 样式：

```jsx
class App extends React.Component {
  constructor(props) {
    super(props)
    this.state = {showModal: false}
  }

  handleShow = () => {
    this.setState({showModal: true})
  }
  
  handleHide = () => {
    this.setState({showModal: false})
  }

  render() {
    return (
      <div className="app">
        This div has overflow: hidden.
        <button onClick={this.handleShow}>Show modal</button>
        <Modal visible={this.state.showModal}>
          <div className="modal">
            <div>
              With a portal, we can render content into a different
              part of the DOM, as if it were any other React child.
            </div>
            This is being rendered inside the #modal-container div.
            <button onClick={this.handleHide}>Hide modal</button>
          </div>
        </Modal>
      </div>
    )
  }
}
```

考虑将这个 Modal 组件提供给项目协作者使用，他就一定要保证富组件的样式满足一定条件，才能正确展示 Modal。

## 使用 Portal 的 Modal

React 16 引入了 `ReactDOM.createPortal(child, container)` 这个 API，允许我们将任何可被 render 的 React child 置入任意的 dom container 中。

基于这个 API，我们可以这么创建 Modal：

```jsx
const modalRoot = document.getElementById('modal-root')

class Modal extends React.Component {
  constructor(props) {
    super(props)
 		// 创建一个 div 来容纳 modal
    this.el = document.createElement('div')
  }
  
  componentDidMount() {
    // 挂载时，将 modal 渲染到 DOM 中
    modalRoot.appendChild(this.el)
  }
  
  componentWillUnmount() {
    // 组件卸载时，将 modal 移出 DOM
    modalRoot.removeChild(this.el)
  }
  
  render() {
    // 将 modal 中的内容渲染到 modal 容器中
    return ReactDOM.createPortal(
    	this.props.children,
      this.el
    )
  }
}
```

使用的时候，我们不再关心 modal 的父组件：

```jsx
class App extends React.Component {
  constructor(props) {
    super(props)
    this.state = {showModal: false}
  }

  handleShow = () => {
    this.setState({showModal: true})
  }
  
  handleHide = () => {
    this.setState({showModal: false})
  }

  render() {
    const modal = this.state.showModal ? (
      <Modal>
        <div className="modal">
          <div>
            With a portal, we can render content into a different
            part of the DOM, as if it were any other React child.
          </div>
          This is being rendered inside the #modal-container div.
          <button onClick={this.handleHide}>Hide modal</button>
        </div>
      </Modal>
    ) : null
    
    return (
    	<div className="app">
      	This div has overflow: hidden.
        <button onClick={this.handleShow}>Show Modal</button>
        {modal}
      </div>
    )
  }
}
```

## 事件冒泡

React 16 中，portal 可以被放在 DOM 树中的任意位置，但 portal 在 React tree 中的位置仍然和代码中其所在的位置一样。因此，事件冒泡仍然能和代码中反映的一致：

```jsx
const modalRoot = document.getElementById('modal-root')

class Modal extends React.Component {
  constructor(props) {
    super(props)
 		// 创建一个 div 来容纳 modal
    this.el = document.createElement('div')
  }
  
  componentDidMount() {
    // 挂载时，将 modal 渲染到 DOM 中
    modalRoot.appendChild(this.el)
  }
  
  componentWillUnmount() {
    // 组件卸载时，将 modal 移出 DOM
    modalRoot.removeChild(this.el)
  }
  
  render() {
    // 将 modal 中的内容渲染到 modal 容器中
    return ReactDOM.createPortal(
    	this.props.children,
      this.el
    )
  }
}

class Parent extends React.Component {
  constructor(props) {
    super(props)
    this.state = {clicks: 0}
    this.handleClick = this.handleClick.bind(this)
  }

  handleClick() {
    // 当 Child 中的 button 被点击时，该方法将被调用。
    // 即便按钮在真实 DOM 树中并不是 Parent 的子孙。
    this.setState(prevState => ({
      clicks: prevState.clicks + 1
    }))
  }

  render() {
    return (
      <div onClick={this.handleClick}>
        <p>Number of clicks: {this.state.clicks}</p>
        <p>
          Open up the browser DevTools
          to observe that the button
          is not a child of the div
          with the onClick handler.
        </p>
        <Modal>
          <Child />
        </Modal>
      </div>
    )
  }
}

function Child() {
  // 按钮的点击事件将被冒泡的父组件，因为这里没有显式定义 `onClick`
  return (
    <div className="modal">
      <button>Click</button>
    </div>
  );
}

```



