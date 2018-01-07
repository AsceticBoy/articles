# How to achieve a Promise

## Environment
浏览器是一个复杂的环境，`JavaScript` 这个单独的线程依托于这个宿主平台，帮助用户获得更好的用户体验，但是往往 `JavaScriot` 的执行还常常伴随着其他操作，比如我需要依托它 `修改DOM`、`加载CSS` 等等。

这就会出现几个问题：
- 1.在执行其他任务时，`JavaScript` 因为等待反馈结果，只能干等着，这不大大的浪费了CPU的资源吗
- 2.如果所有的工作都依赖于浏览器的其他线程执行结束，这样的 `流式处理` 其实是让用户及其痛苦的，或者可能还没等页面出来，用户已经关闭了浏览器，真是非常糟糕的用户体验啊

不过所幸的是，工程师们很早就想到了这样的问题，所以才有了 `事件模型` 的概念，无论从浏览器端还是现在端 `nodeJs` 都基于这个基础此
```js
var event = domNode.addEventListener('load', function() {
  // TODO
}, false)
```

示例中我们在 `node` 加载完成后，执行后续的操作，这样确实就避免了 `阻塞` 的问题，但是试想两种情况：
- 1.我们需要在 *加载完某个节点 -> 修改某段 `CSS` -> 再发一个 `ajax`...* 一个长长的 `事件流` 甚至更长，或许我们会这么写
  ```js
  var event = domNode.addEventListener('load', function(e) {
    changeStyle(e.target, function() {
      ajax(url, function() {
        // TODO....
      })
    })
  }, false)
  ```
  代码会可读性将变得非常差，而且也非常容易出错，这就是 `callback回调地狱`

- 2.设想如果我们在为一个 `image node` 注册一个加载完成后的回调事件前，这个节点已经加载完成，那在这次加载过程中我们也就捕捉不到后续操作，除非我们在后面又重新加载了它，此时代码上就需要增加特殊的判断来避免这种情况
  ```js
  var image = docement.getElementById('image')
  function imageOnload() {
    // TODO
  }
  // 这就是特殊判断的代码
  if (image.completed) {
    imageOnload()
  }
  image.addEventListener('load', imageOnload)
  ```

  这只是一个简单的图片加载的情况，实际场景可能我们需要加载更多的东西，这样的 `特殊情况` 会越来越频繁

那我们的核心究竟需要的是什么 ？
- `事件模型` 其实已经从根本上解决了 `异步` 的问题，我们需要的其实是在事件发生后能够被告知，能够被执行后续操作
- `事件什么时候执行完成` 并不是我们关心的重点，我们只是想知道事件执行的结果，如果有一个 `状态机` 能记录下来，那该多好

## Why Promise ?

`Promise` 英文翻译为 `承诺` 顾名思义，这中间包含着 `稳定` 从中也能看出 `Promise` 中的两个特点：
- 1.状态量单一，`Promise` 只存在三种可能：`pending(进行中)`、`fulfilled(成功)`、`rejected(失败)`
- 2.状态量一旦稳定，就被一直记录下来，不会发生篡改，也不可逆。状态量信息将被一直记录在 `Promise` 中
- 3.通过 `then` 可以很好的针对结果做不同处理，同时返回的 `Promise` 让我们能 `链式调用` 后续操作，摆脱了 `回调地狱` 的控制

`Promise` 是 `ECMAScript` 标准中的一部分，现已经被广泛到浏览器所支持，同时现代浏览器在许多新的 `JS API` 中得到了广泛的使用，比如 `ServiceWorker`、`Notification` etc...

`Promise` 在 `ECMAScript` 标准中核心方法只包含 `Promise` 本身和其原型链上的 `then` 方法，在后续的使用过程中得到了不断的扩充，也将 `all` `race` `catch` 等方法也不断的纳入其中

## How To Achieve

首先我们知道它是一个状态机，在启动时( `new` )就立即执行，且状态只会从 `pending -> fulfilled` 或 `pengding -> rejected`，同时它会储存下状态对应的 `状态量`，因此我们初步写出：

  ```js
  // core.js
  function Promise(exec) {
    // ...这里会有一些判断，比如[exec必须是个function] [使用new的方式来新建Promise]...

    this._status = 'pengding' // 默认为penging
    this._val = null // 状态量

    let done = false // 确保执行一次
    try { // 所有异常不再向外抛，所以需要内部捕获
      exec(
        (result) => {
          done = true
          doResolve(result)
        },
        (reason) => {
          done = true
          doReject(reason)
        }
      ) // 立即执行
    } catch(e) {
      done = true
      doReject(e)
    }
  }
  ```

## Promise Shortcoming

- `Promise` 有自己的错误捕获机制，内部的错误不会向外抛出，即 `try.catch` 无法捕获，除非设置回调函数带出错误
  ```js
  // 这里有个实现上的不同点：
  // chrome原生支持点Promise在reject之后但未做下一步操作，即then或者catch时，不仅会抛出异常且会记录下Promise的value为error
  // 各版本的实现中只做了状态的存储，在下一步操作时(then或者catch)再抛出异常
  // 但无论哪种情况，外层的try.catch都是无法捕获异常的
  try {
    // 用回调函数将错误带出
    function cb(err) {
      throw err
    }
    new Promsie(function (resolve, reject) {
      reject(new Error('this error can not catch'))
    }).catch(function (err) {
      // 此处才可以捕获异常
      cb(err)
    })
  } catch(e) {
    console.log(e) // 此处无法捕捉到异常
  }
  ```
  根本原因：`异步产生的错误无法靠同步的try.catch捕获`

- `Promise` 一旦执行后中途无法取消
  ```js
  // 当一个错误被捕获到后却仍旧会执行，不管在catch中return还是throw，都仍旧会往下面执行
  new Promise(function(resolve, reject) {
    resolve(42)
  })
  .then(function(value) {
    // BIG ERROR
    throw new Error('shortcoming')
  })
  .catch((err) => console.log('error')) // .catch((err) => (new Promise(() => {})))
  .then(() => console.log('then1'))
  .then(() => console.log('then2'))
  .catch()
  .then()
  ```
  从中也可以看出 `Promise` 是个状态机，一旦确定状态就会按后面到流程处理，直到结束。当然也有一些 `hack` 的手段就是让 `catch` 处理完异常以后，返回一个 `pending` 状态的 `Promise`

- `Promise` 发生后，无法感知其具体进度，如是刚刚开始，还是将要结束