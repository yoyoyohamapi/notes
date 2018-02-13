# callbag 规范

**`(type: number, payload?: any) => void`**

### 定义

- Callbag：一个函数，其签名为： `(type: 0 | 1 | 2, payload?: any) => void`
- Greet：如果一个 callbag 以 `0` 为第一个参数进行调用，我们就说 `该 callbag 被问好了`。此时函数执行的操作是： “向这个 callbag 问好”。
- Deliver：如果一个 callbag 以 `1` 为第一个参数进行调用，我们就说 “这个 callbag 正被交付数据”。此时函数执行的操作是：“交付数据给这个 callbag”。
- Terminate：如果一个 callbag 以 `2` 为第一个参数进行调用，我们就说 “这个 callbag 被终止了”。此时函数执行的操作是：“终止这个 callbag”。
- Source：一个负责交付数据的 callbag。
- Sink：一个负责接受数据的 callbag。

### 协议

**问好（Greets）**: `(type: 0, cb: Callbag) => void`

当第一个参数是 `0`，而第二个参数是另外一个 callbag（即一个函数）的时候，这个 callbag 就被问好了。

**握手**

当一个 source 被问好，并被作为 payload 传递给了某个 sink，sink **必须**使用一个 callbag payload 进行问好，这个 callbag 可以是他自己，也可以是另外的 callbag。换言之，问好是相互的。相互间的问好被称为**握手**。

**终止（Termination）**: `(type: 2, err?: any) => void`

当第一个参数是 `0`，而第二个参数要么是 undefined（由于成功引起的终止），要么是任何的真实值（由于失败引起的终止），这个 callbag 就被终止了。

在握手之后，source **可能**终止掉 sink，sink 也**可能**会终止掉 source。如果 source 终止了 sink，则 sink **不应当**终止 source，反之亦然。换言之，终止行为**不应该**是相互的。

**数据交付（Data delivery）** `(type: 1, data: any) => void`

交付数目：

- 一个 callbag（source 或者 sink）**可能**会被一次或多次交付数据

有效交付的窗口：

- 一个 callbag **一定不能**在被问好之前被交付数据
- 一个 callbag **一定不能**在终止后被交付数据
- 一个 sink **一定不能**在其终止了它的 source 后被交付数据

**保留码**

一个 callbag 不应该使用下面这些数字作为第一个参数： `3`, `4`, `5`, `6`, `7`, `8`, `9`，这些被称作**保留码（reserved codes）**。某个 callbag **可能**使用其他 `[0-9]` 区间以外的保留码进行调用，但是这份规范不会考虑这些场景。