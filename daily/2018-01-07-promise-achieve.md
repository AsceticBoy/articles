# How to achieve a Promise

浏览器是一个复杂的环境，JavaScript这个单独的线程依托于这个宿主平台，帮助用户获得更好的用户体验，但是往往JS的执行还常常伴随着其他操作，比如我需要依托它 `修改DOM`、`加载CSS` 等等。

这就会出现几个问题：
- 1.在执行其他任务时，JS因为等待反馈结果，只能干等着，这不大大的浪费了 `CPU资源` 吗
- 2.如果所有的工作都依赖于浏览器的其他线程执行结束，这样的 `流式处理` 其实是让用户极其痛苦的，或者可能还没等页面出来，用户已经关闭了浏览器，真是非常糟糕的用户体验啊

不过所幸的是，工程师们很早就想到了这样的问题，所以才有了 `事件模型` 的概念，无论从 `浏览器端` 还是 `nodeJs` 都基于这个基础
```js
var event = domNode.addEventListener('load', function() {
  // TODO
}, false)
```

示例中我们在 node加载完成后，执行后续的操作，这样确实就避免了 `阻塞` 的问题，但是试想两种情况：
- 1.我们需要在 `加载完某个节点 -> 修改某段CSS -> 再发一个ajax...`，一个长长的 `事件流`，甚至更长，或许我们会这么写
  ```js
    var event = domNode.addEventListener('load', function(e) {
      changeStyle(e.target, function() {
        ajax(url, function() {
          // TODO....
        })
      })
    }, false)
  ```
  显然层次嵌套会让代码的可读性变得非常的差，而且也非常容易出错，这就是所谓的 `callback回调地狱`

- 2.设想如果我们在加载一个 `image node` 节点前，JS还未来得及注册事件，那么这次加载过程我们将无法捕捉到，自然也就没有了后续操作，除非我们在后面又重新加载了它，此时代码上就需要增加特殊的判断来避免这种情况
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

  这只是一个简单的图片加载的情况，实际场景中我们可能需要加载更多的东西，这样的 `特殊情况` 会出现的越来越频繁

那我们的核心需求究竟是什么 ？
- `事件模型` 其实已经从根本上解决了 `同步执行` 的问题，我们需要的其实是在事件发生后能够被告知，能够被执行后续操作
- `事件什么时候执行完成` 并不是我们关心的重点，我们只是想知道事件执行的结果，如果有一个 `状态机` 能记录下来，那该多好

## Why Promise ?

`Promise` 英文翻译为 `承诺` 顾名思义，这中间包含着 `稳定` 的意思，从中也能看出 `Promise` 中的几个特点：

  - 1.状态量单一，`Promise` 只存在三种可能：`pending(进行中)`、`fulfilled(成功)`、`rejected(失败)`
  - 2.状态量一旦稳定，就会被一直记录下来，不会发生篡改，也不可逆。状态量信息将被一直记录在 `Promise` 中
  - 3.通过 `then` 可以很好的针对结果做不同处理，同时返回新的 `Promise` 让我们能 `链式调用` 后续操作，摆脱了 `回调地狱` 的控制

`Promise` 是 `ECMAScript` 标准中的一部分，现已经被广泛到浏览器所支持，同时现代浏览器在许多新的 `JS API` 中得到了广泛的应用，比如 `ServiceWorker`、`Notification` etc...

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
            doResolve.apply(this, result)
          },
          (reason) => {
            done = true
            doReject.apply(this, reason)
          }
        ) // 立即执行
      } catch(e) {
        done = true
        doReject.apply(this, e)
      }
    }

    function doResolve(result) {
      this._status = 'fulfilled'
      this._val = result
    }
    function doReject(reason) {
      this._status = 'rejected'
      this._val = reason
    }
  ```

OK，看起来好像一个初步的 `Promise` 构造函数已经成型，但是我们好像缺少这几种情况的考虑：

  - 等待执行的 `doResolve` 或者 `doReject` 接受的也是一个 `Promise` 该怎么处理，当发生reject时我们可以认为这是一个异常，将内容直接作为状态量，resolve呢 ？显然 `doResolve` 仍需要改造，这个我们后面讨论
  - 既然 `Promise` 和 `callback` 一样，都是来解决 `异步场景` 下的问题的，那异步在哪里，现在的代码仍然是一个同步的执行过程

这两个疑问我们先暂且搁置，那我们先来实现 `ECMAScript Standard` 中的另一个方法 `then`，从各个版本中可以知道 `Promise.then` 是并不存在的，那么显然这个方法是在 `Promise原型链` 上的

  ```js
    // Promise.prototype.then
    Promise.prototype.then = function (endOfResolve, endOfReject) {
      var self = this
      var newPromise = self.constructor(() => {}) // 需要返回一个新的Promise，此时让它处于pending状态
      // 判断当前的的Promise实例的状态，做不同处理
      if (self._status !== 'pending') {
        var func = self._status === 'fulfilled' ? endOfResolve : endOfReject
        asyncFunc(newPromise, func, self._val)
      } else {
        // Promise还在运行状态该怎么处理未来才知道的结果？需要用一个队列来记录这些待处理事件
        self._onRejectedCallback.push(function (reason) {
          asyncFunc(newPromise, endOfReject, reason)
        })
        self._onResolveCallback.push(function (result) {
          asyncFunc(newPromise, endOfResolve, result)
        })
      }
      return newPromise
    }
    // asyncFunc
    function asyncFunc(self, func, arg) {
      try{
        // 当有状态量时触发回调
        var res = func(arg)
        doResolve.apply(self, res) // 将endOfResolve或者endOfReject的结果作为下一个当前Promise的状态量
      } catch(e) {
        doReject.apply(self, e)
      }
    }
  ```
好了，我们好像已经实现了标准所规定的 `Promise`, 但是显然这里面存在一些特殊的情况，我们仍然没有考虑在内

  - 1.首先依然是 `异步问题`，如果当前我们在 `endOfResolve` 中直接resolve一个常量，那整个过程其实就是同步执行的，和 `Promise` 的理念相违背( `异步的同步调用` )
  - 2.我们在 `self._status === 'pending'` 情况下将回调函数塞入的 `队列`，该何时调用
  - 3.如果 `endOfResolve` 和 `endOfReject` 方法传入的是 `undefined`，该怎么处理，从现有的实现来看，当任一函数为空时，`Promise` 仍旧可以将 `状态量` 传入到下一个回调函数，类似：
    ```js
      new Promise(resolve => resolve(8)).then().then(function(value) {
        console,log(8)
      })
    ```
  - 4.最重要的问题是 `func(arg)` 所返回的内容仍旧是一个 `Promise`，那当前的Promise该怎么处理自身的 `状态量`

带着这些问题，我们先来解决回调函数为空的问题，这种情况其实应该称为 `值穿透`，这个情况还是比较容易解决的
  ```js
    endOfResolve = typeof endOfResolve === 'function' ? endOfResolve : (value) => { return value }
    endOfReject = typeof endOfReject === 'function' ? endOfReject : (reason) => { throw reason }
  ```
当为空时，我们手动创建一个函数，将 `状态量` 往下一个 `Promise` 传递，这样就很好的解决了 `值穿透` 的问题，然后我们再来看刚才的问题，显然我们需要进一步改造 `doResolve` 和 `doReject` 这两个方法，来处理那些较为复杂的 `数据传递` 情况，实际上 `ECMAScript Standard` 中明确指示了这些复杂情况的 [实现标准][1]，为了就是能够让 `不同的Promise实现能够相互通信`，直接看代码：

  ```js
    function safeToExec(exec) {
      let done = false // 确保执行一次
      try { // 所有异常不再向外抛，所以需要内部捕获
        exec(
          (result) => {
            done = true
            doResolve.apply(this, result)
          },
          (reason) => {
            done = true
            doReject.apply(this, reason)
          }
        ) // 立即执行
      } catch(e) {
        done = true
        doReject.apply(this, e)
      }
    }
    function safeTothen(thenable) {
      var then = thenable instanceof Promise
        ? Promise.prototype.then
        : thenable && thenable.then // 标准2.3.3.1 thenable.then有可能是一个getter，同时也防止getter抛出错误
      if ((typeof thenable === 'object' || typeof thenable === 'function') 
        && typeof then === 'function') {
          return function () {
            then.apply(thenable, arguments)
          }
      }
    }
    // doResolve
    function doResolve(result) {
      try {
        if (self === result) throw new TypeError('....') // 对应标准2.3.1
        // 个人觉得：[result in Promise](对应标准2.3.2)和标准2.3.3都是在等待result的最终状态量
        // 从实现上都需要调用then方法，所以就在此处合并处理了,主要体现在safeToThen中
        var then = safeTothen(result) // 获取then，若有错误会被catch捕获
        if (then) {
          // 2.3.3.3
          // 这不至关重要涵盖了2.3.3.3中的所有要领，简单来说就是若状态量为Promise就一直等到最内层的Promise得到状态量后一层一层返还回去
          safeToExec.apply(this, then.bind(result)) 
        } else {
          // 2.3.3.4
          // 已经确定了状态量
          self._status = 'fulfilled'
          self._val = result;
          // pengding时self的状态量未确定，但此时已经明确，所以newPromise的状态量也就能够确定了
          for (let i = 0; i < self._onResolveCallback.length; i++) {
            self._onResolveCallback[i](result);
          }
          self._onResolveCallback = [];
        }
      } catch (e){
        doReject.apply(self, e)
      }
    }
    // doReject
    function doReject(reason) {
      // reject相对简单只负责将reason错误和特定的Promise相关联
      self._status = 'rejected'
      self._val = reason;
      // 同理：状态量此时已经明确，所以newPromise的状态量也就能够确定了
      for (let i = 0; i < self._onRejectedCallback.length; i++) {
        self._onRejectedCallback[i](reason);
      }
      self._onRejectedCallback = [];
    }
  ```
从代码上看整个实现好像变得复杂又模糊，但其实已经解决了我们之前说提到的大部分问题，整体归纳下：

  - 所谓的 `值穿透` 本质是不断的将 `状态量` 赋予到 `新产生的Promise` 中，直到遇到存在相应回调函数的 `Promise`
  - 每个 `Promise` 中所包含的 `_onResolveCallback` 和 `_onRejectedCallback` 队列，都是用来方便记录 `当自身拿到状态量转变后，能及时通知依赖于自己状态量出现的其余Promise执行下一步的操作`
  - 按照标准，当 `doResolve` 的值较为复杂时，其实遵循的原则是 `如果参数是Promise就等待Promise的执行结果，非Promise有then函数就等待then函数的执行结果，其余的就视为状态量本身` 具体实现上需要参考规范，这应该是 `Promise` 里面最复杂的一部分逻辑了，需要细细推敲

Promise标准中的所有的流程都已经实现了，开始测试吧！这里推荐一个官方的Promise的 [测试工具][2]，是用 `Mocha` 实现的，用来测试自己的 `Promise` 是否符合官方标准，经过测试发现Promise总是会出现 `Error: Timeout of 2000ms exceeded. For async tests and hooks, ensure "done()" is called` 类似关于 `timeout` 的错误，显然我们刚才还有一个异步的问题没处理，有可能是 `同步调用` 导致的，那什么时候需要异步调用呢 ？

  - 假设我们正常使用Promise，异步会存在于什么地方，那应该是 `获取到值的地方`，比如一个Ajax，那对于 `Promise` 来说是不是可以理解为 `改变状态量的地方`，以及 `确定状态量后，回调执行的地方`，在原先的代码上增加异步逻辑，这里用 `setTimeout` 代替：

    ```js
      function doResolve() {
        // before...
        setTimeout(function () {
          // 2.3.3.4
          // 已经确定了状态量
          self._status = 'fulfilled'
          self._val = result;
          // pengding时self的状态量未确定，但此时已经明确，所以newPromise的状态量也就能够确定了
          for (let i = 0; i < self._onResolveCallback.length; i++) {
            self._onResolveCallback[i](result);
          }
          self._onResolveCallback = [];
        }, 0)
        // after...
      }

      function doReject(reason) {
        setTimeout(function () {
          // reject相对简单只负责将reason错误和特定的Promise相关联
          self._status = 'rejected'
          self._val = reason;
          // 同理：状态量此时已经明确，所以newPromise的状态量也就能够确定了
          for (let i = 0; i < self._onRejectedCallback.length; i++) {
            self._onRejectedCallback[i](reason);
          }
          self._onRejectedCallback = [];
        }, 0)
      }

      function asyncFunc(self, func, arg) {
        setTimeout(function () {
          try{
            // 当有状态量时触发回调
            var res = func(arg)
            doResolve.apply(self, res) // 将endOfResolve或者endOfReject的结果作为下一个当前Promise的状态量
          } catch(e) {
            doReject.apply(self, e)
          }
        }, 0)
      }
    ```

好了，再次测试，发现已经能够顺利通过，至此 `Promise` 已经完成实现 [传送门][3]

  - 注意实际情况中并不会用 `setTimeout` 来替代异步手段，详见此处 [Promise性能问题][4]，个人理解是使用 `Microtask` 要好过 `Macrotask`，当Promise调用链比较多时 `Micotask` 更节约时间，而且在特定的情形下能防止 `UI阻塞` 而产生卡顿感，详见：[JS事件轮询][5]

## Promise Shortcoming

- `Promise` 有自己的错误捕获机制，内部的错误不会向外抛出，即 `try.catch` 无法捕获
  ```js
    // 这里有个实现上的不同点：
    // chrome原生支持点Promise在reject之后但未做下一步操作，即then或者catch时，不仅会抛出异常且会记录下Promise的value为error
    // 各版本的实现中只做了状态的存储，在下一步操作时(then或者catch)再抛出异常
    // 但无论哪种情况，外层的try.catch都是无法捕获异常的
    try {
      new Promise(function (resolve, reject) {
        reject(new Error('this error can not catch'))
      }).catch(function (err) {
        // 此处才可以捕获异常
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

- `Promise` 发生后，无法感知其具体进度，只是刚刚开始，还是将要结束

[1]:https://promisesaplus.com/#point-46
[2]:https://github.com/promises-aplus/promises-tests
[3]:https://github.com/AsceticBoy/chilli-toolkit/blob/master/lib/ipromise/core.js
[4]:https://github.com/xieranmaya/blog/issues/3
[5]:./2017-11-07-event-loop.md