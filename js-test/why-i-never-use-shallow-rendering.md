> 原文地址：[https://kentcdodds.com/blog/why-i-never-use-shallow-rendering](https://kentcdodds.com/blog/why-i-never-use-shallow-rendering)

## 什么是 Shallow Rendering
假定我们的组件代码是：

```jsx
import React from 'react'
import {CSSTransition} from 'react-transition-group'
function Fade({children, ...props}) {
  return (
    <CSSTransition {...props} timeout={1000} className="fade">
      {children}
    </CSSTransition>
  )
}
class HiddenMessage extends React.Component {
  static defaultProps = {initialShow: false}
  state = {show: this.props.initialShow}
  toggle = () => {
    this.setState(({show}) => ({show: !show}))
  }
  render() {
    return (
      <div>
        <button onClick={this.toggle}>Toggle</button>
        <Fade in={this.state.show}>
          <div>Hello world</div>
        </Fade>
      </div>
    )
  }
}
export {HiddenMessage}
```

使用以 Shallow Rendering 为测试模式的 Enzyme，测试代码就会是：

```jsx
import React from 'react'
import Enzyme, {shallow} from 'enzyme'
import Adapter from 'enzyme-adapter-react-16'
import {HiddenMessage} from '../hidden-message'
Enzyme.configure({adapter: new Adapter()})
test('shallow', () => {
  const wrapper = shallow(<HiddenMessage initialShow={true} />)
  expect(wrapper.find('Fade').props()).toEqual({
    in: true,
    children: <div>Hello world</div>,
  })
  wrapper.find('button').simulate('click')
  expect(wrapper.find('Fade').props()).toEqual({
    in: false,
    children: <div>Hello world</div>,
  })
})
```

如果我们使用 `wrapper.debug()` 进行调试，则能看到 enzyme 为我们的渲染的组件结构如下：

```jsx
<div>
  <button onClick={[Function]}>Toggle</button>
  <Fade in={true}>
    <div>Hello world</div>
  </Fade>
</div>
```

shallow 就体现在此，我们看到，enzyme 只考虑了 <`HiddenMessage />` 的浅层渲染，而没有继续其 children `<Fade />` 的渲染，所以在此，我们看不到 `<Fade />` 需要渲染的 `<CSSTransition />`。

实际上，如果我们打印 `<HiddenMessage />` 的 `render` 调用结果时，能看到其输出的是一个 JavaScript 对象：

```javascript
{
  "type": "div",
  "props": {
    "children": [
      {
        "type": "button",
        "props": {
          "onClick": [Function],
          "children": "Toggle"
        }
      },
      {
        "type": [Function: Fade],
        "props": {
          "in": true,
          "children": {
            "type": "div",
            "props": {
              "children": "Hello world"
            }
          }
        }
      }
    ]
  }
}
```

Enzyme 对这个对象进行了包裹，返回了一个 `wrapper`，这个 `wrapper` 上捆绑了一些工具函数，让我们能去访问这个对象上的一些属性。因此，shallow rendering 并不能让我们运行生命周期函数，也不允许我们与 DOM 进行交互，也无法获得自定义组件所返回的组件元素。

概括说来，shaollow rendering 着重测试的是组件的 **态**：即组件的 props 或者 state，被测试对象也局限在组件自身。

Kent C.Dodds 调研了开发者为什么要使用 Enzyme：

- 为了直接调用 React 组件的方法
- 觉得 render 所有的组件和子组件对于单测开销甚大
- 认为单测就应该聚焦在被测试组件自身，不受额外组件影响

这些问题可以逐一来看：

### 调用 React 组件的方法

```jsx
test('toggle toggles the state of show', () => {
  const wrapper = shallow(<HiddenMessage initialShow={true} />)
  expect(wrapper.state().show).toBe(true) // initialized properly
  wrapper.instance().toggle()
  wrapper.update()
  expect(wrapper.state().show).toBe(false) // toggled
})
```

这样的测试，是否脆弱，我们考虑两个问题：

1. 一些组件撰写错误，能否让测试失败，从而保证错误不被带到线上？
1. 当我们重构了组件后，并且这个重构是向后兼容的，是否我不需要重写用例，也能让用例有效？

但是 Shallow Rendering 无法做到这样的保障：

1. 在组件中，我将 `onClick` 绑定到了 `this.togogle`，而不是 `this.toggle`，即便如此，测试仍然能够通过，因为我们没有渠道测试到 `onClick`，我们只能测试组件实例上的 `toggle` 方法。
1. 我可以重命名 `toggle` 为 `handleButtonClick`，并更新 `onClick` 的绑定对象，但这个重构需要我重写测试，不然用例就无法通过。

让 Shallow Rendering 测试脆弱的原因是，我们在测试一些无关的实现细节（比如实例方法，组件状态等）。用户关注的是组件的行为，而不是实现，例如用户关注的不是 show state 某刻时刻是 `true` 还是 `false` ，而是此时此刻，组件应当做什么样的展示。

### 开销大
测试应当关注质量而不是速度，主流的诸如 Jest 这样的测试框架，也对测试用例的运行有优化。

### 不应该测试其他组件
这是一个误区，如上例中，我们看到自定义的 `<Fade />` 组件也没有被 render，这就导致，如果我们对 `<Fade />` 做出了修改，由于测试只关注 `<HiddenMessage />`，因此他不能发现我们这次修改中可能存在的问题。

测试，终究是要关注其能为我们带来的质量保证。

## react-testing-library

> RTL 属于 Full DOM rendering

相比于 Enzyme，react-testing-library 不能做到：

1. shallow rendering
2. static rendering（如 Enzyme 提供的 `render` 函数）
3. 一些查询 React element 的能力，例如 Enzyme 可以直接用 Component Class 查询元素 `.find(Foo)`
4. 直接获得组件实例，如 `instance()`
5. 获取与设置组件属性，如 `props()`
6. 获取与设置组件状态，如 `state()`

但是这些能力，与 “用户如何使用组件” 并不相关，看到 RTL 的测试代码，你会发现，它更接近用户使用组件的方式：

```jsx
import 'react-testing-library/cleanup-after-each'
import React from 'react'
import {CSSTransition} from 'react-transition-group'
import {render, fireEvent} from 'react-testing-library'
import {HiddenMessage} from '../hidden-message'
// NOTE: you do NOT have to do this in every test.
// Learn more about Jest's __mocks__ directory:
// https://jestjs.io/docs/en/manual-mocks
jest.mock('react-transition-group', () => {
  return {
    CSSTransition: jest.fn(({children, in: show}) => (show ? children : null)),
  }
})
test('you can mock things with jest.mock', () => {
  const {getByText, queryByText} = render(<HiddenMessage initialShow={true} />)
  const toggleButton = getByText('Toggle')
  const context = expect.any(Object)
  const children = expect.any(Object)
  const defaultProps = {children, timeout: 1000, className: 'fade'}
  expect(CSSTransition).toHaveBeenCalledWith(
    {in: true, ...defaultProps},
    context,
  )
  expect(getByText('Hello world')).not.toBeNull()
  CSSTransition.mockClear()
  fireEvent.click(toggleButton)
  expect(queryByText('Hello world')).toBeNull()
  expect(CSSTransition).toHaveBeenCalledWith(
    {in: false, ...defaultProps},
    context,
  )
})
```


