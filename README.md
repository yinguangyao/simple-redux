# simple-redux
一个简单的 redux 实现，仅供参考其原理实现。

## 原理
## createStore
redux源码还是比想象中的要精炼很多的，除去注释和打印错误信息，核心代码大概在两三百行左右，我们就先从createStore这个入口函数一探究竟。

![redux入口.png-41.8kB][1]

从返回参数来看，createStore函数主要返回了这四个我们常用的函数。
![createStore.png-251.1kB][2]

进入到create函数里面看，第一个判断应该是当用户没有给初始化的数据，直接将enhancer函数（一般是applyMiddleware）当第二个参数传进来的时候做的一些默认处理，如果enhancer是一个函数，那么就会把reducer和preloadedState传给enhancer，我们知道enhancer一般是一些中间件函数，这里的reducer一般是combineReducers这个函数的返回值，我们再来看看combineReducers和applyMiddleware这两个函数。
## combineReducers
删除掉多余的注释和打印信息后，完整的combineReducers函数是这样的：
![combineReducers.png-309.2kB][3]

参数reducers就是我们最后传给combineReducers的那个对象，一般是我们reducer函数的对象集合。

这里代码也很清晰了，将reducer函数以键值对的形式赋给finalReducers对象，并且返回一个combination函数，这个函数可以拿到当前的状态和要执行的action，这两个参数肯定是在createStore里面调用的时候传入的，我们先不用管。

这里使用for循环来遍历每一个reducer函数，这也就意味着，我们每次触发一个action，redux都会遍历并执行一遍所有的reducer函数，直到找到匹配的那个action.type（这里我不得不说react-imvc应该是做了一些优化的，它以action.type作为reducer函数名，这样就不需要去遍历查找了，可以减少很多不必要的工作量）。

并且将执行后的结果nextStateForKey和前一个状态做比较，最后根据判断是return新值还是老值，这也是为什么在reducer函数里面最后一定要return出来一个新的对象or数组才会刷新store，而不能简单的修改一下当前的state，并将其直接返回，因为这里比较的是引用。
## applyMiddleware
Middleware这个概念是Redux从其他框架借鉴过来的，本意如下：

> middleware是指可以被嵌入在框架接收请求到产生响应过程之中的代码。例如，Express 或者 Koa 的 middleware
> 可以完成添加 CORS headers、记录日志、内容压缩等工作。

而在Redux中：

> middleware被用于解决不同的问题，但其中的概念是类似的。它提供的是位于 action 被发起之后，到达 reducer
> 之前的扩展点。 你可以利用 Redux middleware 来进行日志记录、创建崩溃报告、调用异步接口或者路由等等。

看完了combineReducers函数，我们继续分析applyMiddleware函数：

![applyMi.png-172kB][4]

applyMiddleware函数就更加简练了一些，我们一般会把redux-thunk、redux-logger这种中间件当做参数传给applyMiddleware。

redux的中间件是在action触发后执行的，所以中间件内部必须拿到完整的state、dispatch和action，这里使用compose包裹了中间件方法，最终返回了一个新的dispatch，可以理解为这个dispatch是经过中间件加强后的dispatch。
## compose
![compose.png-80.9kB][5]

这里是compose的源码，我们可以明显看出来这里是将dispatch再次作为参数放进去的，最后得到一个强化的dispatch，结合redux-logger来理解，大概是传入dispatch后对其加了打印的功能，之后再返回出来。
## dispatch
再回头来看我们的createStore函数，我们关键来看一下对应的几个函数：
![dispatch.png-152.4kB][6]
首先是我们的dispatch函数，dispatch函数主要做了两件事，一个是执行reducer函数拿到最新的state，另一个是执行subscribe的事件。

dispatch接收了一个action当参数，通过isDispatching来判断是否执行reducer，这也就不可能出现多个dispatch同时执行的情况了，因为这样会干扰store的值。这里看到会把currentState传到reducer里面，更新后得到了新的currentState，之后还执行了一下listener函数，这个函数是从nextListeners里面拿到的。
## subscribe
这里我们看一下subscribe函数：
![subscribe.png-128kB][7]

subscribe会传入一个回调函数，这个函数一般是监听redux中状态变化后执行的，nextListeners里面保存着所有需要执行的回调，如果subscribe函数执行两次，那就是卸载当前加载上的listener。

这样的话，其实还是有一个问题，如果我们用subscribe监听了ReactDOM.render，这样我们每次发送dispatch，即使最后state没有变化，页面也是会重新render。
## 重写
这里是自己重写的简练版redux：
```
/// 这里需要对参数为0或1的情况进行判断
const compose = (...funcs) => {
    if (!funcs) {
        return args => args
    }
    if (funcs.length === 1) {
        return funcs[0]
    }
    return funcs.reduce((f1, f2) => (...args) => f1(f2(...args)))
}

const bindActionCreator = (action, dispatch) => {
    return (...args) => dispatch(action(...args))
}

const createStore = (reducer, initState, enhancer) => {
    if (!enhancer && typeof initState === "function") {
        enhancer = initState
        initState = null
    }
    if (enhancer && typeof enhancer === "function") {
        return enhancer(createStore)(reducer, initState)
    }
    let store = initState, 
        listeners = [],
        isDispatch = false;
    const getState = () => store
    const dispatch = (action) => {
        if (isDispatch) return action
        // dispatch必须一个个来
        isDispatch = true
        store = reducer(store, action)
        isDispatch = false
        listeners.forEach(listener => listener())
        return action
    }
    const subscribe = (listener) => {
        if (typeof listener === "function") {
            listeners.push(listener)
        }
        return () => unsubscribe(listener)
    }
    const unsubscribe = (listener) => {
        const index = listeners.indexOf(listener)
        listeners.splice(index, 1)
    }
    return {
        getState,
        dispatch,
        subscribe,
        unsubscribe
    }
}

const applyMiddleware = (...middlewares) => {
    return (createStore) => (reducer, initState, enhancer) => {
        const store = createStore(reducer, initState, enhancer)
        let chain = middlewares.map(middleware => middleware(store))
        store.dispatch = compose(...chain)(store.dispatch)
        return {
          ...store
        }
      }
}

const combineReducers = reducers => {
    const finalReducers = {},
        nativeKeys = Object.keys
    nativeKeys(reducers).forEach(reducerKey => {
        if(typeof reducers[reducerKey] === "function") {
            finalReducers[reducerKey] = reducers[reducerKey]
        }
    })
    return (state, action) => {
        const store = {}
        nativeKeys(finalReducers).forEach(key => {
            const reducer = finalReducers[key]
            const nextState = reducer(state[key], action)
            store[key] = nextState
        })
        return store
    }
}
```
## 总结
redux源码实现很精简，比想象中的还要简单，react-redux在redux基础中多了Provider和connect两个方法，通过context将store传给Provider包裹的组件，之后会再开一篇文章分析react-redux的源码。

  [1]: http://static.zybuluo.com/gyyin/1jibi2tofs7mazu6jfsmxs7s/redux%E5%85%A5%E5%8F%A3.png
  [2]: http://static.zybuluo.com/gyyin/an7e9bhcfayz00zh4etg20w1/createStore.png
  [3]: http://static.zybuluo.com/gyyin/kum9agu106cscpkcp1wbsarb/combineReducers.png
  [4]: http://static.zybuluo.com/gyyin/0jwy1qmx1uqwsnzgjm1qas2f/applyMi.png
  [5]: http://static.zybuluo.com/gyyin/tw7co5q0p8dkn1kdyqyi72en/compose.png
  [6]: http://static.zybuluo.com/gyyin/y8s7f4h2s6dmzqxb2zpa5kte/dispatch.png
  [7]: http://static.zybuluo.com/gyyin/fmtq4qz206dnet1s2h4x4zz1/subscribe.png