# Transducer

## Reduce

```js
function sum(result, item) {
    return result + item
}

function mult(result, item) {
    return result * item
}

const sumed = [2, 3, 4].reduce(sum, 1)
const multed = [2, 3, 4].reduce(mult, 1)
```

reduce 概括：

- 始于一个初始值
- 通过 reducing function 进行迭代：
  -  第一步中使用初始值
  - 后续步骤中迭代的对象为上一次迭代结果
- 使用最后的计算结果作为输出

reduce 实际上就是一个 transformation（转化）的过程

## Transformer

因此，我们可以使用一个对象来描述 reduce：

```js
const transformer = function(reducingFunction) {
    return {
        init() {
            return 1
        },
        step: reducingFunction,
        result(result) {
            return result
        }
    }
}
```

用例：

```js
const input = [2, 3, 4]
const xf1 = transformer(sum)
const summed = input.reduce(xf.step, xf.init())
const xf2 = transformer(multi)
const multed = input.reduce(xf.step, xf.init())
```

再将 transfomer 从 input 和 output 中解耦：

```js
function reduce (xf, init, input) {
    const result = input.reduce(xf.step, init)
	return xf.result(result)
}
```

现在，新的用例为：

```js
const input = [2, 3, 4]
const xf1 = transformer(sum)
const summed = reduce(xf1, xf.init(), input)
const xf2 = transformer(mult)
// 也可以手动指定初始值
const multed = reduce(xf2, 2, input)
```

为了让 `reduce()` 也能接收一个普通的 reducing function 而不是 transfomer 作为第一个参数，我们创建一个 wrapper 将 reducing function 转换为 transfomer：

```js
function reduce(xf, init, input) {
    if (typeof xf === 'function') {
        xf = wrap(xf);
    }
    const result = input.reduce(xf.step, init)
    return xf.result(result)
}

function wrap(xf) {
    return {
        // 初始值已经接收了，此时不再需要
        init() {
            throw new Error('init nor supported')
        },
        step: xf,
        result(result) {
            return result
        }
    }
}
```

现在，新的用例为：

```js
const input = [2, 3, 4]
const summed = reduce(sum, 2, input)
```

除了算数级的 reduce，我们也可以完成数组级的 reduce：

```js
function append(result, item) {
    result.push(item)
    return result
}

const input = [2, 3, 4]
const appened = reduce(append, [1], input)
```

## Transducer

如果我们想对序列中的每个元素进行累加，首先定义一个累加函数：

```js
function plus1(item) {
    return item + 1
}
```

反映这个过程的 transformer 为：

```js
const xfplus1 = {
    init() {
        throw new Error('init not needed')
    },
    step(result, item) {
        const plused = plus1(item)
        return append(result, plused)
    },
    result(result) {
        return result
    }
}
```

用例为：

```js
const xf = xfplus1
const init = []
const input = [2, 3, 4]
const output = reduce(xfplus1, [], input)
```

如果此时我们还想对 `output` 中的元素求和，就需要:

```js
const summed = reduce(sum, 0, output)
```

在这个求和实现中，我们不得不创建一个中间数组 `output`，这不是最优实践。在函数式编程中，我们应当优先定义好逻辑，最后接受数据进行计算。因此，我们引入 transducer 来预定义 transform 过程，它接受一个 tansformer 并返回一个新的 transfomer：

```js
function transducerPlus1(xf) {
    return {
        init() {
            return xf.init()
        },
        step(result, item) {
            const plused = plus1(item)
            return xf.step(result, plused)
        },
        result(result) {
            return xf.result(result)
        }
    }
}
```

新的用例即为：

```js
const input = [2, 3, 4]
const stepper = wrap(append)
const transducer = transducerPlus1
const xf = transducer(stepper)
const output = reduce(xf, [], input) // => [3, 4, 5]
```

这里，`stepper` 声明了当元素进行了累加操作后的后续步骤。如果我们累加后是想要求和，则替换我们的 `stepper` 即可：

```js
const input = [2, 3, 4]
const stepper = wrap(sum)
const transducer = transducerPlus1
const xf = transducer(stepper)
const output = reduce(xf, 0, input) // => 12
```

## 优化

假如需要的是累加 2，而不是累加 1，再手写一个 `transducerPlus2` 是很呆的办法，为此，我们创建一个 `map` 方法用于创建只完成映射的 transducer：

```js
function map(f) {
    return function transducer(xf) {
        return {
            init() {
                return xf.init()
            },
            step(result, item) {
                const mapped = f(item)
                return xf.step(result, item)
            },
            result(result) {
                return result
            }
        }
    }
}
```
用例：

```js
function plus2(item) {
    return item + 2
}
const input = [2, 3, 4]
const transducePlus2 = map(plus2)
const stepper = wrap(append)
const transducer = transducerPlus2
const xf = transducer(stepper)
const output = reduce(xf, [], input)
```

## Tranduce

我们梳理一下上例中对输入数据进行转换的过程：

1. 声明对数据序列中各个元素操作的 transducer:

   ```js
   const transducer = map(plus1)
   ```

2. 声明元素操作后的累加步骤 stepper：

   ```js
   const stepper = wrap(append)
   ```

3. 使用 transducer 包裹累加步骤 stepper，完成对数据处理的定义：

   ```js
   const xf = transducer(stepper)
   ```

整个过程非常类似于最开始定义的 `reduce` 过程，我们将该过程定义为 transduce：

```js
function transduce(transducer, stepper, init, input) {
    if (typeof stepper === 'function') {
        stepper = wrap(stepper)
    }
    
    const xf = transducer(stepper)
    
    return reduce(xf, init, input)
}
```

那么，用例就可以简化为：

```js
const transducer = transducerPlus1
const stepper = append
const input = [2, 3, 4]
const output = transduce(transducer, stepper, [], input) // => [3, 4, 5]
```

## 组合

### 组合元素处理过程

如果我们了解函数式编程中的组合，我们可以轻易的组合对元素的逐个处理过程：

```js
const transducerPlus3 = map(compose(plus1, plus2))
const transducer = transducerPlus3
const input = [2, 3, 4]
const output = transduce(transducer, stepper, [], input) // => [4, 5, 6]
```

其中，组合函数如下：

```js
const compose = (...funcs) => x => funcs.reduceRight((input, func) => func(input), x)
```

### 组合 Transducer

```js
const transducerPlus1 = map(plus1)
const transducerPlus2 = map(plus2)
const transducerPlus3 = compose(transducerPlus1, transducerPlus2)
const stepper = append
const input = [2, 3, 4]
const output = transduce(transducer, stepper, [], input)
```

## ramda 中的 transducer

在 ramda 中的 transduce 方法，其接收的 transducer 操作的是整个序列，而不是序列上单个元素：

```js
const numbers = [1, 2, 3, 4]
const transducer = R.compose(R.map(R.add(1)), R.take(2))
R.transduce(transducer, R.flip(R.append), [], numbers) //=> [2, 3]
```

## 参考资料

- [Transducers Explained: Part 1](http://simplectic.com/blog/2014/transducers-explained-1/)
- [ramda-transduce](http://ramdajs.com/docs/#transduce)