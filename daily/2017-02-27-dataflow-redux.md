# How to you understand Redux

## Overview

数据流算是这两年前端圈内的热点词汇，从SPA普及以来，交互模式的日益升级与增强，不可避免的需要管理和规整更多的数据源，这些数据可能来自后台的业务数据，可能是交互中的临时变量，可能是组件的状态量...，可见把这一染缸的数据统一管理并不是什么容易的事，随之而来的是 `Reflux`、`Redux` 这类数据框架大行其道。最终，Redux凭借 `副作用隔离` 和 `单一可回溯的数据源` 等特点得到了市场的高度认可，当然Redux的 `Middleware机制` 也是它的一大卖点，推动起大量社区中间件和函数式热，也是它地位不可动摇的一大原因。不可否认，Redux带来了前端数据流热，`无副作用`、`可回溯可追踪`、`不可变数据` 这些关键词成为了议论的焦点。随后，`Mobx`、`RxJs`、`dob` 这些以 `Observable` 为代表的框架开始冒出来，很大程度上确实是因为Redux上手成本高所导致的。当然，他们各有千秋，需要用户择善而取。

> 作者打算做一个关于市面上常见数据流框架的剖析：涵盖 `Redux`、`Mobx` 和 `RxJs`，主要以使用切入到实现以及在使用过程中的一些思考，包括周边生态，有兴趣的小伙伴可以一起探讨。

## Redux Usage

先来看看redux代码的基本用法：

```js
  // store的初始数据
  let initalState = { textList: [] }
  // action(action creator)
  // action是一个js对象包含着触发后续动作信息，也是redux驱动的必要条件
  function actionCreator(message) {
    return { type: 'ADD', text: message }
  }

  // reducer是修改redux数据的唯一函数，传入store和action两个参数，注意它是个纯函数，不能做异步的事
  function reducer(state = initalState, action) {
    let newTextList = [].concat(state.textList)
    switch (action.type) {
      case 'ADD': {
        newTextList.push(action.text)
        break;
      }
    }
    return { ...state, textList: newTextList }
  }
  // 其中combineReducers用于将不同作用的reducer进行合并，内部实现我们稍后再谈论
  let reducers = combineReducers({ reducer })

  // store可以认为就是将action和reducer连在一起的对象，存储着全局数据源state
  let store = createStore(reducers, initalState) // 接受reducers和initalState返回store

  // 好了，现在我们就能利用redux开始最基本的数据驱动了
  store.dispatch(actionCreator('add one list message'))
  // 看下现在的store
  console.log(store.getState()) // { textList: ['add one list message'] }
```
其实明眼人应该一眼就能看出来，这里只是用到了最最基础发布订阅模式，这可能并没什么，但是这恰恰是redux最核心思想的驱动方式

`Action`：Redux中的action实质只是一个JavaScript的 `Plain Object`，store中所有数据的改变，都是以驱动一个action为基础的。redux约定我们用action对象中的一个 `type` 字段来作为特定标识，相比普通对象而言，`action creators` 的形式更被Redux所提倡：
  - 方便使用 `bindActionCreators` 绑定 `dispatch` 避免向下级组件传递，也就是说下级组件做到了与Redux的解耦
  - 在 `redux-thunk` 中实际上action creator被作为一个 `thunk`，即返回函数的函数，使得 `控制得到反转(我们能得到dispacth和getState函数)`，组件调用和异步逻辑可以完全分离，直接看代码：

  ```js
    // 使用方法：
    // success 和 error都是普通的action creator
    function success(message) { return { type: 'success', payload: message } }
    function error(error) { return { type: 'error', payload: error } }

    function thunk(param) { // 这就是一个thunk

      return function (dispatch， getState) { // 通过回调(这里也可称为控制反转)出来的两个关键性函数
        //函数里面就可以开始写异步逻辑了 -> TODO
        return ajax(param).then(
          message => dispatch(success(message)),
          error => dispatch(error(error))
        );
      };
    }
    thunk(id) 
    // 内部实现 -> 注释代码涉及到Middleware的内容可以先不看，主要是if里面的语句

    // function createThunkMiddleware(extraArgument) {
    //  return ({ dispatch, getState }) => next => action => {
          if (typeof action === 'function') {
            return action(dispatch, getState, extraArgument);
          }
    //    return next(action);
    //  };
    // }
  ```
`Reducer`：Redux中另一个重要的概念是reducer，用来修改全局store这个单一数据源，所以它只涉及到数据的修改，应尽量保持它的纯净，借用函数式的概念，它必须是一个 `纯函数`，且有以下几个特点：
  - 传递同样的参数，得到相同的结果，所以在函数内部不能执行有副作用的动作，比如 `API请求` 和 `路由跳转` 等
  - 既然是个纯函数也不能调用如 `Math.random()` 和 `Date.now()` 这样的非纯函数动作

同时，redux还希望我们在每个reducer返回的结果遵循：`不要直接修改State的引用，在每次修改部分值时，通过Object.assign返回一个新的，当没有改变值或匹配不到相应当action时，返回旧的State`，同时这两个特点在源码中也是有所体现的：
  ```js
    // 先来看看combineReducers的实现部分，截取了它的函数返回。同时删除了一些非关键性代码
     
    return function combination(state = {}, action) {
      let hasChanged = false
      const nextState = {}
      for (let i = 0; i < finalReducerKeys.length; i++) {

        // redux默认用户会将这个key作为state数据部分，即和reducer的函数签名相同
        const key = finalReducerKeys[i]
        
        const reducer = finalReducers[key]
        const previousStateForKey = state[key]
        const nextStateForKey = reducer(previousStateForKey, action)
        if (typeof nextStateForKey === 'undefined') {
          const errorMessage = getUndefinedStateErrorMessage(key, action)
          throw new Error(errorMessage)
        }

        // 这个位置就是redux按自己对reducer的命名规范([state[key]]: function() {})，做了一层比较，如果全部state[key]比较下来都相同就返回旧的state
        nextState[key] = nextStateForKey
        hasChanged = hasChanged || nextStateForKey !== previousStateForKey
      }
      return hasChanged ? nextState : state
    }
    // ----------------------------------------------
    // 再来看看当匹配到无效action type的情况，在combineReducers.js中的assertReducerShape函数中有所反映

    Object.keys(reducers).forEach(key => {
      const reducer = reducers[key]
      const initialState = reducer(undefined, { type: ActionTypes.INIT })

      if (typeof initialState === 'undefined') {
        throw new Error(
          `Reducer "${key}" returned undefined during initialization. ` +
          `If the state passed to the reducer is undefined, you must ` +
          `explicitly return the initial state. The initial state may ` +
          `not be undefined. If you don't want to set a value for this reducer, ` +
          `you can use null instead of undefined.`
        )
      }
    })
  ```
  **而且可以看出combineReducers的返回函数其实就是一个 `闭包`，整合了 `所有的合法reducers的一个集合`（在combineReducers.js做了一系列的验证来保证reducer的规范），并且在每次 `dispatch` 调用时触发**
  ```js
    // 这是在源码dispatch的调用位置
    // currentReducer其实就是combineReducers的返回
    try {
      isDispatching = true
      currentState = currentReducer(currentState, action)
    } finally {
      isDispatching = false
    }
  ```
`Store`：redux中连接action和reducer的枢纽，同时也是存放唯一数据源的实例。提供了一系列方法
  - `getState()`：获取数据源state
  - `dispatch()`：触发 `reducers` 和 `listeners`，改变state且响应第三方模块的监听函数
  - `subscribe()`：注册监听函数，一般用于对redux扩展的第三方模块
  - `replaceReducer(nextReducer)`： 高级用法，替换当前reducers，用于实现redux热加载 `module.hot.accept()`

整个 `createStore.js` 也差不多就是围绕这几个api来设计的，所以看懂这几个函数基本也就完成了对store的理解
  ```js
    // 这是createStore的核心代码
    function createStore (reducer, preloadedState, enhancer) {
      if (typeof enhancer !== 'undefined') {
        if (typeof enhancer !== 'function') {
          throw new Error('Expected the enhancer to be a function.')
        }
        // 增强器部分先跳过，后续详细说明
        return enhancer(createStore)(reducer, preloadedState)
      }
      // 定义一些私有属性
      let currentReducer = reducer // 经过combineReducers包装后的reducers的集合
      let currentState = preloadedState // 单一store
      let currentListeners = [] 
      let nextListeners = currentListeners
      let isDispatching = false // 是否处于触发动作状态，类似loading
      // ---------------------
      function getState() {
        // getState，顾名思义，返回currentState即可
        if (!isDiapatching) {
          return currentState
        }
      }
      // ---------------------
      function dispatch(action) {
        // action必须是个普通Js对象
        if (!isPlainObject(action)) {
          throw new Error('....')
        }
        // action.type不能为空
        if (typeof action.type === 'undefined') {
          throw new Error('...')
        }
        // 理论上同步Js不会走到这步，但是做一个防范，redux虽然是同步，但是第三个插件涵盖异步的可能
        if (isDispatching) {
          throw new Error('...')
        }
        // dispatch核心部分，其实在上面已经有阐述
        try {
          isDispatching = true
          currentState = currentReducer(currentState, action) // 触发所有reducers
        } finally {
          isDispatching = false
        }
        // 触发第三方模块注册的监听函数
        // 区分currentListeners和nextListeners是更符合redux不可变原则，更加安全
        const listeners = (currentListeners = nextListeners)
        for (let i = 0; i < listeners.length; i++) {
          const listener = listeners[i]
          listener()
        }
        // 返回action对象的设置其实在我看完Middleware后，发现dispatch函数本身真的是一个很好的函数模版，dispatch(action) => action
        // 使得这种隐式的约定在Middleware中得到了很好的体现，当然这归功于applyMiddleware做了一个很好的解耦
        return action
      }
      // ---------------------
      function subscribe(listener) {
        // 必须是函数
        if (typeof listener !== 'function') {
          throw new Error('...')
        }
        // 不能在执行过程中添加监听函数，保证dispatch执行的安全性
        if (isDispatching) {
          throw new Error('...')
        }

        let isSubscribed = true
        // 比较currentListeners和nextListeners的索引，保证每次dispatch的listeners都是一个新对象
        ensureCanMutateNextListeners()
        nextListeners.push(listener)
        // unsubscribe()
        // 闭包：返回解绑函数，且不需要传递具体监听器
        return function unsubscribe() {
          // 确定当前需要移除的监听器已经被绑定
          if (!isSubscribed) {
            return
          }

          if (isDispatching) {
            throw new Error('...')
          }

          isSubscribed = false

          ensureCanMutateNextListeners()
          const index = nextListeners.indexOf(listener)
          // 移除相关监听器
          nextListeners.splice(index, 1)
        }
      }
      // ---------------------
      function replaceReducer(nextReducer) {
        // nextReducer是经过combineReducers处理的reducers集合
        if (typeof nextReducer !== 'function') {
          throw new Error('...')
        }

        currentReducer = nextReducer
        // 热加载实现的关键就在于每次替换新的nextReducer后需要触发一次所有reducers和listeners的调用
        dispatch({ type: ActionTypes.REPLACE })
      }
    }
  ```
`createStore.js` 的源码其实也基本就在里面了，其中还有一个 `observable` 函数，以 `symbol值` 为函数签名，猜测是为了迎合 [ECMAScript标准][1]，这里对它的实现也不作展开了，后期再开坑，有兴趣的同学可以去翻看下redux的源码。

总体来说，redux实现数据变更的流程就是如下图所示，唯一能改变state的方式就是通过dispatch的方式来发起一个action，然后通过reducer这个纯函数来唯一改变state的值，整个流程下来，数据流的清晰就得到了很好的保证。搭配React单向数据流的思想，简直完美！在这个过程中，我们也看到了redux本身是同步的、无副作用的，但异步这个前端最不稳定的因素redux是没有提供方案的，但是它提供了很好的Middleware机制，来让第三方社区帮助其实现。当然，Middleware的作用不仅仅于此，可以说redux是一个 `锁定了核心数据驱动方式，但是又给开发者提供了无限扩展可能的框架`。

```js
  //   ---------                              -------------
  //  |         |   store.dispatch(action)   |             |
  //  |  store  | → → → → → → → → → → → → →  | Middleware1 |
  //  |         |              ↑             |      ↓      |
  //   ---------               ↑             |      ↓      |
  //       ↑              ----------         | Middleware2 |
  //       ↑             |          |        |     ...     |
  //       ↑             |  action  |        |      ↓      |
  //       ↑             |  creator |        |      ↓      |
  //       ↑             |          |        |   reducer   |
  //       ↑              ----------         |             |
  //       ↑                                  -------------
  //       ↑                                        ↓
  //       ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ← ←
```

## Redux Publish-Subscribe

从redux的使用介绍中我们可以看出，从动作参数的输入到store的改变，`store` 充当了 `数据中心` 和 `调度者` 的身份，我们将源码化繁为简，对这种模式的核心部分进行抽象

```js
  function controlCenter(initalData) {
    // data: 外界无法直接访问的私有变量，产生任何变化都将触发反馈
    let data = initalData;
    // listeners：可以泛指监听函数，当data改变时触发调用，reducers和currentListeners都是如此
    let listeners = []
    // dispatch: 触发改变的行为让listener得到反馈
    function dispatch(condition) {
      // TODO: condition -> change data 这部分在redux中由reducer充当
      // data的改变进一步反馈到监听器
      for (var i = 0; i <= listeners.length; i++) {
        if (typeof listeners[i] === 'function') {
          listeners[i](data)
        }
      }
    }
    // register：在得到监听前，需要将监听函数提前注册挂载
    function register(listener) {
      // 防止重复注册
      if (listeners.indexOf(listener) > -1) {
        listeners.push(listener)
        // cancalRegister：注销监听器不是必须的
        return function cancalRegister() {
          let index = listeners.indexOf(listener)
          listeners.splice(index, 1)
        }
      }
    }
    // 暴露给外面获取数据的方式
    function getData() {
      return data
    }
    return {
      getData,
      dispatch,
      register
    }
  }
```
抽象出来的单元也被称为 `发布/订阅模式`，这是一种非常常见的设计模式，是很多框架的底层依赖。一些新起API（ `Obserable/Observer` ）大多都基于它的模式思想上做了进一步的优化。

## Redux Middleware

正如我们前面所说的那样，redux的store部分主要规范了数据的驱动方式，起到了调度者的作用。如果仅仅是这样，redux也不会这么出名，它并没有解决前端最繁琐的副作用问题。显然它需要让redux保留扩展的活性，让开发者来解决这些脏活累活，而它只应该在store数据更改的关卡上进行把关，约束store state的改变方式。`Middleware` 赋予了redux新的生命力。

现假设我们需要在用户发起一个action的过程中记录下对象信息并且在store改变后打印出新的state，记录下日志便于排查。
  ```js
    function wrapperDispatch(action) {
      console.log(action)
      store.dispatch(action)
      console.log(store.getData())
    }
  ```
我们在dispatch触发前后输出了相关日志，封装成了函数，但是破坏了redux的写法，这非常的不直观。对于redux而言，所谓的中间件是指在发起action到reducer改变数据过程中的扩展，这个过程中我们能改变的只有dispatch，但你我们既需要保持dispatch现有的写法，又要让Middleware的实现对用户是无感知的。`猴子补丁` 是一个选择方案。
  ```js
    // 因为返回的是一个闭包，我们可以在createStore的时候先将它注册到redux上
    function logger(store) {
      let next = store.dispatch
      return function newDispatch(action) {
        console.log(action)
        next(action)
        console.log(store.getData())
      }
    }
    // 这册这个logger插件
    function registerMiddleware(store, middleware) {
      let dispatch = logger(store)
      return { ...store, dispatch }
    }
  ```
猴子补丁解决了破坏redux写法的问题，但是上面的代码存在两个关键性问题
  - 1.猴子补丁实际上是对函数的重载，即便是重新调用了破坏前的原始函数，也和低耦合的思想相违背
  - 2.在多个中间件相继注册的过程中，代码尚未提供多个被包装的dispatch之间的链接（中间件之间没有通信）

来看看redux是怎么巧妙的解决这两个问题的：
  ```js
    // 使用过程中先注册需要的中间件,这里就以thunk和logger示例
    applyMiddleware(logger, thunk)
    // redux applyMiddleware相当于链接注册了各个中间件的作用
    function applyMiddleware(...middlewares) {
      return createStore => (...args) => {
        // 生成原始store
        const store = createStore(...args)
        let dispatch = () => {
          throw new Error(
            `Dispatching while constructing your middleware is not allowed. ` +
              `Other middleware would not be applied to this dispatch.`
          )
        }
        let chain = []

        const middlewareAPI = {
          getState: store.getState,
          dispatch: (...args) => dispatch(...args)
        }
        chain = middlewares.map(middleware => middleware(middlewareAPI))
        dispatch = compose(...chain)(store.dispatch)

        return {
          ...store,
          dispatch
        }
      }
    }
  ```
单独看applyMiddleware会让人摸不着头脑，那我们配合thunk的源码来看
  ```js
    // thunk
    ({ dispatch, getState }) => next => action => {
      if (typeof action === 'function') {
        return action(dispatch, getState, extraArgument);
      }

      return next(action);
    };
  ```
`applyMiddleware` 首先调用之前的createStore方法，返回redux最初的 `store对象`，将 `store.getState` 和 `新的dispatch` 注入到每个中间件中
  ```js
    const middlewareAPI = {
      getState: store.getState,
      dispatch: (...args) => dispatch(...args)
    }
    chain = middlewares.map(middleware => middleware(middlewareAPI))
    // ↓ ↓ ↓ ↓ ↓
    ({ dispatch, getState }) => () => () => {}
  ```
- 注意：这里的dispatch由于是引用的值传递，所以永远指向新的dispatch

通过注入的方式每个Middleware已经得到store的高阶API了，但是此时dispatch并未发生实质性变化，也未做猴子补丁。随后redux调用了 `compose(...chain)(store.dispatch)` 返回新的dispatch方法并覆盖，代码完结
  ```js
    dispatch = compose(...chain)(store.dispatch)

    return {
      ...store,
      dispatch
    }
  ```
是不是很神奇（第一次我也觉得amazing，哈哈，原谅没怎么接触过函数式的我）！redux真的只用的一行代码就优雅的解决了Middleware的痛点。奥妙在于 `compose`
 ```js
  function compose(...funcs) {
    if (funcs.length === 0) {
      return arg => arg
    }

    if (funcs.length === 1) {
      return funcs[0]
    }

    return funcs.reduce((a, b) => (...args) => a(b(...args)))
  }
 ```
在函数式编程中compose是个非常基础的概念，其目的是 `将一串相互独立的函数进行串联，返回一个新的函数，调用后，各函数按一定顺序，依次执行（从左到右），后一个函数的返回将作为前一个函数的入参`。compose将各个独立的函数块进行了拼接，像搭积木一样保证了功能之间松散型。

compose的理念恰好与我们解决问题的思路不谋而合：

- 1.首先compose的特性可以保证中间件的执行顺序问题。`因为每个中间件的next正好是下一个需要执行的中间件函数，而是否要继续往下调用，取决于当前中间件`。
  ```js
    // 以logger和thunk为例
    let logger = (middlewareAPI) => next => action => {
      // TODO
      return next(action)
    }
    let thunk = (middlewareAPI) => next => action => {
      // TODO
      return next(action)
    }
    applyMiddleware(logger, thunk)
    let composed = compose([
      next => action => { // ... } // logger
      next => action => { // ... } // thunk
    ])
    // 当compose启动
    compose(store.action)
    // 可以看出：logger的 action => {} 将作为thunk的 next
    // 依次类推所有中间的拼接...
  ```
  所有中间件经过这样拼接后，最终返回的dispatch才是现在redux中最终的dispatch，所以当 `store.dispatch(action)` 执行时，所有的中间件会 `按着注册的顺序依次进行`。

- 2.通过 `柯里化` 替换了猴子补丁，动作和目标解耦
  ```js
    let logger = (middlewareAPI) => next => action => {
      // next和action都以参数的形式传入函数体
      return next(action)
    }
  ```
中间件的执行顺序需要特别注意：`compose(...chain)(store.dispatch)` 中间件是 `从右往左传递next`，但是返回的dispatch一旦执行却是 `从左往右执行中间件`，而且是否继续调用后面的中间件取决于当前中间件。

对于中间件，`express`等服务端框架运用的更早，各中实现其实都是差不多的。redux的作用依旧是调度，只有符合了 `(middlewareAPI) => next => action => { next(action) }` 形式的Middlewares才能受到redux的管理，这也是约束。

## Redux Enhancer

仅仅是Middleware，redux认为或许还不够满足用户，redux Middleware能包装的只有dispatch，如果用户还想修饰store的其他方法，可能有些捉襟见肘了。`store enhancer` 就是用来提供给用户更宽泛的扩展可能性。
```js
  // 先看下redux怎么注册store enhancer
  // 这是createStore的函数签名，初始化数据和enhancer是可选字段
  createStore(reducer, [preloadedState], [enhancer])

  // store enhancer(增强器)的一般形式
  function createEnhancer(...someThingExtra) {
    return createStore => (...args) => {
      // TODO
      let store = createStore(...args)
      // convert store funcs...
    }
  }
  // 使用
  createStore(
    combineReducer(reducers),
    initalState,
    createEnhancer()
  )
```
从上面的签名中可以注意到：`applyMiddleware函数本身就是一个store enhancer的生成器`。它串联所有中间件，返回包装后的dispatch重载原始方法，称为enhancer很贴切。

当出现多个store enhancer的时候，我们需要先将enhancers组合生成一个新的enhancer作为createStore的入参
```js
  let enhancerA = (...extra) => createStore => (...args) => {
    let store = createStore(...args)
    // convert store funcs...
    return { ...store, ...[converts] }
  }
  let enhancerB = (...extra) => createStore => (...args) => {
    let store = createStore(...args)
    // convert store funcs...
    return { ...store, ...[converts] }
  }
  let enhancer = compose(enhancerA(), enhancerB())
  // usage
  createStore(
    combineReducer(reducers),
    initalState,
    enhancer
  )
```
这段代码是不是有点似曾相识的感觉，关键是又见到了 `compose` 函数，意味着 `enhancerA和enhancerB像中间件一样进行了组合`，结合createStore中enhancer的部分，我们再来仔细看看：
```js
  function createStore(reducer, preloadedState, enhancer) {
    if (typeof preloadedState === 'function' && typeof enhancer === 'undefined') {
      enhancer = preloadedState
      preloadedState = undefined
    }

    if (typeof enhancer !== 'undefined') {
      if (typeof enhancer !== 'function') {
        throw new Error('Expected the enhancer to be a function.')
      }
      return enhancer(createStore)(reducer, preloadedState)
    }
    // ....
  }
```
关键性代码是 `enhancer(createStore)(reducer, preloadedState)`，如果像dispatch一样，我们把所有的createStore理解为 `next`，一切就好理解多了
```js
  let enhancerA = (...extra) => next => (...args) => {
    let store = next(...args)
    // convert store funcs...
    return { ...store, ...[converts] }
  }
  let enhancerB = (...extra) => next => (...args) => {
    let store = next(...args)
    // convert store funcs...
    return { ...store, ...[converts] }
  }
  compose(enhancerA(), enhancerB())(createStore)(reducer, preloadedState)
```
store enhancers的串联模式和redux middlewares是完全一样的，`每个enhancer的入参next是下一个enhancer的增强函数`，差别只存在于：`中间件是按注册顺序依次调用，而增强器是逆序增强（即：前一个增强始终是基于后一个增强器返回的store进一步增强）`。
```js
  let funcA = (...extra) => next => (...args) => {
    console.log('funcA before')
    let result = next(...args)
    console.log('funcA after')
    return result
  }
  let funcB = (...extra) => next => (...args) => {
    console.log('funcB before')
    let result = next(...args)
    console.log('funcB after')
    return result
  }
  let funC = (...extra) => next => (...args) => {
    console.log('funcC before')
    let result = next(...args)
    console.log('funcC after')
    return result
  }
  compose(funcA(...extra), funcB(...extra), funC(...extra))(initNext)(...args)
  // console order...
  // funcA before
  // funcB before
  // funcC before
  // funcC after
  // funcB after
  // funcA after
```
从上面的打印顺序能很好的看出middleware和enhancer的调用顺序差别，可以看出 `enhancer的执行顺序非常的关键`。只有清楚完整的调用过程，才能避免破坏性enhancer出现。比如：
```js
  function breakingEnhancer(...extra) {
    return createStore => (...args) => {
      const store = createStore(reducer, initialState, enhancer)
      function dispatch(action) {
        // TODO...
        return res;
      }
      return { ...store, dispatch }
    }
  }
```
这个增强器重载了redux dispatch但没有将原始的dispatch保留下来，破坏了redux的工作流。所以拓展自己的store enhancer非常关键的一点就是：`store enhancer是store的扩展，但是不应破坏redux原有的工作流，否则将导致redux不可用`

[redux][2]做的所有任务基本就是这些，小而精美，不干涉数据更改以外的其他行为，提供了插拔式的中间件和增强器的拓展形式。保持约束的同时也没有强行限制关键性API的更改，一切都给用户留有余地。

## React-Redux
- [ ] react-redux源码分析 -> 思考启示

[1]:https://github.com/tc39/proposal-observable/blob/master/src/Observable.js
[2]:https://github.com/reactjs/redux/blob/master/README.md