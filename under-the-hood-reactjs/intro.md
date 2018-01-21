# Intro

一个 Demo 组件，用于分析 React 执行流程：

```js
import React from 'react'
import ReactDOM from 'react-dom'

class ChildCmp extends React.Component {
  render() {
    return <div>{this.props.childMessage}</div>
  }
}

class ExampleApplication extends React.Component {
  constructor(props) {
    super(props)
    this.state = {message: 'no message'}
  }

  componentWillMount() {

  }

  componentDidMount() {

  }

  shouldComponentUpdate(nextProps, nextState, nextContext) {

  }

  componentDidUpdate(prevProps, prevState, prevContext) {

  }

  componentWillReceiveProps(nextProps) {

  }

  componentWillUnmount() {

  }

  onClickHandler() {

  }

  render() {
    return (
      <div>
        <button onClick={this.onClickHandler.bind(this)}>set state button</button>
        <ChildCmp childMessage={this.state.message} />
        And some text as well!
      </div>
    )
  }
}

ReactDOM.render(
  <ExampleApplication hello={'world'} />,
  document.getElementById('container'),
  function () {}
)
```

我们可以在 `node_modules/react` 及 `node_modules/react-dom` 中设置断点或者 `console.log` 来观测 React 执行流程。 