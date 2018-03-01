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
`createStore.js` 的源码其实也基本就在里面了，其中还有一个 `observable` 函数，以 `symbol值` 为函数签名，猜测是为了迎合 [ECMAScript标准](1)，这里对它的实现也不作展开了，后期再开坑，有兴趣的同学可以去翻看下redux的源码。

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
- [ ] 中间件的使用入手到核心源码分析
- [ ] 思考抽离出redux Middleware的抽象模式

## Redux Enhancer
- [ ] 增强器的使用 -> 源码查看 -> 实质

## React-Redux
- [ ] react-redux源码分析 -> 思考启示

[1]:https://github.com/tc39/proposal-observable/blob/master/src/Observable.js