# Error Boundary（错误边界）

## 引子

看到下面的一个例子，我们的应用维护了一份学生列表，当点击 “Next” 按钮后，将展示下一名学生的信息：

```jsx
const Student = ({student}) => {
    const { name, age } = student
    return (
        <div>
            <h1>name: {name}</h1>
            <h3>age: {age}</h3>
        </div>
    )
}

class App extends React.Component {
    constructor(props) {
        super(props)
        this.students = [{
            name: 'Tom',
            age: 16
        }, {
            name: 'Jerry',
            age: 20
        }]
        this.state = {
            currentIdx: 0,
            student: this.students[0]
        }
    }
    
    handleNextClick = () => {
        this.setState({
            student: this.students[this.state.currentIdx + 1],
            currentIdx: this.state.currentIdx + 1
        })
    }

    render() {
        return (
        	<div>
            	<Student student={this.state.student} />
                <p>
                	<button onClick={this.handleNextClick}>
                    	Next
                    </button>
                </p>
            </div>
        )
    }
}
```

这里存在数组越界的可能，不断点击下一步，控制台将抛出错误，但页面不会崩溃（React 16 则会卸载这个组件树）。

## try-catch

谨慎一点的话，我们进行一个 try-catch ，俘获可能的错误，以避免程序崩溃，并将错误输出：

```jsx
const Student = ({student}) => {
    try {
        const { name, age } = student
        return (
            <div>
                <h1>name: {name}</h1>
                <h3>age: {age}</h3>
            </div>
        )
    } catch(e) {
        return (
        	<div>
            	<h1 style={{color: 'red'}}>Somthing went wrong!</h1>
                <p>{e.toString()}</p>
            </div>
        )
    }
}
```

## `componentDidCatch`

假设我们有若干的子组件，我们并不关心每个组件的异常，只是在任何子组件出错的情况下，父组件不再渲染各个子组件，而是渲染错误消息。在 React 16 以前，我们可能需要**在每个子组件中进行错误捕获（try-catch），并传递给父组件（通常会在 `handleError()`）中调用父组件传给它的 `onError` 方法**。这么做有两点不好：

- 冗长，啰嗦
- 命令式的 `try-catch`  与声明式的 React 相悖

React 16 中，通过 `componentDidCatch` 实现了这个**错误边界**，该函数接收两个参数，这个钩子**能且只能帮助组件 catch 住其所有子组件的异常**：

 - `error` 错误消息
 - `errorInfo` 错误详情

```jsx
const Student = ({student}) => {
    const { name, age } = student
    return (
        <div>
            <h1>name: {name}</h1>
            <h3>age: {age}</h3>
        </div>
    )
}

class App extends React.Component {
    constructor(props) {
        super(props)
        this.students = [{
            name: 'Tom',
            age: 16
        }, {
            name: 'Jerry',
            age: 20
        }]
        this.state = {
            error: null,
            errorInfo: null,
            currentIdx: 0,
            student: this.students[0]
        }
    }
    
    componentDidCatch(error, errorInfo) {
        this.setState({
            error,
            errorInfo
        })
    }
    
    handleNextClick = () => {
        this.setState({
            student: this.students[this.state.currentIdx + 1],
            currentIdx: this.state.currentIdx + 1
        })
    }

    render() {
        if (this.state.error) {
            return (
            	<div>
                	<h1 style={{color: 'red'}}>
                    	{this.state.error.toString()}
                    </h1>
                    <p>
                    	{this.state.errorInfo.componentStack}
                    </p>
                </div>
            )
        } else {
            return (
                <div>
                    <Student student={this.state.student} />
                    <p>
                        <button onClick={this.handleNextClick}>
                            Next
                        </button>
                    </p>
                </div>
            )
        }
    }
}
```

## 边界位置

假设我们应用容器中有详情面板，监控面板，时钟面板，我们创建一个 `ErrorBoundary` 组件：

```jsx
class ErrorBoundary extends React.Component {
    constructor(props) {
        super(props)
        this.state = {
            error: null,
            errorInfo: null
        }
    }
    
  	componentDidCatch(error, errorInfo) {
      	this.setState({
            error: error,
            errorInfo: errorInfo
        })
  	}
    
    render() {
        if (this.state.error) {
            return (
                <div>
                	<h1 style={{color: 'red'}}>
                    	{this.state.error.toString()}
                    </h1>
                    <p>
                    	{this.state.errorInfo.componentStack}
                    </p>
                </div>
            )
        }
    }
}
```

我们想在详情和监控出错时，时钟面板仍能正常交互，就可以这么**放置**我们的错误边界：

```jsx
class App extends React.Component {
    render() {
        return (
        	<div>
            	<ErrorBoundray>
                	<DetailPanel />
                    <MonitorPanel />
                </ErrorBoundray>
                <ClockPanel />
            </div>
        )
    }
}
```

## 未捕获的错误

在 React 16 中，如果你没有对异常进行捕获，那么将造成整个 React 组件树的卸载。而在 React 15 中，即便出现异常，UI 也是最近一次正常的状态。

Facebook 这么做是认为，一旦有错，使得应用崩溃或许更好，这样会阻断用户继续在一个错误的状态上进行操作。

那么，为了让应用不崩溃，也迫使开发者更注重错误捕获，错误日志上报等提升应用健壮性的习惯。

