nextTick

### 在之间我们需要了解**JS运行机制**

我们都知道`javaScript`是单线程的，它是基于**事件循环**。**事件循环**大致分为以下几个步骤

<img src="https://upload-images.jianshu.io/upload_images/4820992-82913323252fde95.png?imageMogr2/auto-orient/strip|imageView2/2/w/863/format/webp" alt="img" style="zoom: 80%;" />

- 整体的script（作为第一个宏任务）开始执行的时候，会把所有的代码分为两部分`同步任务`、`异步任务`
- 同步任务会直接进入主线程一次执行
- 异步任务会分为`宏任务`、`微任务`
- `宏任务(macro task)`进入到`Event Table`中，并在里面注册回调函数，每当指定的事件完成时，`Event Table`会将这个函数移到`Event Queue`中
- `微任务(micro task)`也会进入到另一个`Event Table`中，并在里面注册回调函数，每当指定的事件完成时，`Event Table`会将这个函数移到`Event Queue`中
- 当主线程内的任务执行完毕，主线程为空时，会检查`微任务`的`Event Queue`，如果有任务，就全部执行，如果没有及执行下一个`宏任务(macro task)`
- 上述过程不断重复，就是`Event Loop`事件循环

这里需要说明一下；

- 宏任务(macro task)：常见的有
  - `setTimeout`
  - `MessageChannel`
  - `postMessage`
  - `setImmediate`
- 微任务(micro task)：常见的有
  - `MutationObserver`
  - `Promise.then`

### Vue的实现

Vue在实现`nextTick`单独使用一个js文件来维护它，定义在

> ​	src/core/util/next-tick.js

```typescript
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIOS, isNative } from './env'

// 保存添加任务的数组
const callbacks = []
// 队列任务状态
let pending = false

// 事件循环中任务队列的回调函数
function flushCallbacks () {
  // 将状态置为等带任务添加的状态 
  pending = false
  // 保存一个副本
  // 原因：
  // 如果我们在nextTick的回调函数中使用了nextTick方法
  // 那么这个时候添加的方法会放在下一个任务队列中执行，而不是在当前队列中执行  
  const copies = callbacks.slice(0)
  // 清空任务队列
  callbacks.length = 0
  // 执行回调函数(任务)  
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

// Here we have async deferring wrappers using both microtasks and (macro) tasks.
// In < 2.4 we used microtasks everywhere, but there are some scenarios where
// microtasks have too high a priority and fire in between supposedly
// sequential events (e.g. #4521, #6690) or even between bubbling of the same
// event (#6566). However, using (macro) tasks everywhere also has subtle problems
// when state is changed right before repaint (e.g. #6813, out-in transitions).
// Here we use microtask by default, but expose a way to force (macro) task when
// needed (e.g. in event handlers attached by v-on).
// 保存宏观任务
let microTimerFunc
// 保存微观任务
let macroTimerFunc
// 是否使用`macrotask`的标志
let useMacroTask = false

// Determine (macro) task defer implementation.
// Technically setImmediate should be the ideal choice, but it's only available
// in IE. The only polyfill that consistently queues the callback after all DOM
// events triggered in the same loop is by using MessageChannel.
/* istanbul ignore if */
// 判断浏览器是否支持setImmediate方法，如果支持，直接通过这个方法将我们的任务队列添加到事件循环的`macrotask queue`中
// setImmediate方法：该方法用来把一些需要长时间运行的操作放在一个回调函数中，在浏览器完成后面的其他语句后
// 就立即执行这个回调函数，需要注意的是目前只有最新版本的`IE`和`Node.js 0.10+`实现了该方法
if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  macroTimerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else if (typeof MessageChannel !== 'undefined' && (
  isNative(MessageChannel) ||
  // PhantomJS
  MessageChannel.toString() === '[object MessageChannelConstructor]'
)) {
  // 判断浏览器是否支持`MessageChannel`方法，将我们的任务队列添加到`macrotask queue`中
  // 该方法可以参考[记录：window.MessageChannel那些事](https://zhuanlan.zhihu.com/p/37589777) 
  const channel = new MessageChannel()
  const port = channel.port2
  channel.port1.onmessage = flushCallbacks
  macroTimerFunc = () => {
    port.postMessage(1)
  }
} else {
  /* istanbul ignore next */
  // 如果上面两个条件都不满足的时候，直接降级为`setTimeout`，将我们的任务队列添加到`macrotask queue`中执行
  macroTimerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

// Determine microtask defer implementation.
/* istanbul ignore next, $flow-disable-line */

// 	使用哪种方式将我们的任务队列添加到事件循环的`microtask`事件队列中

// Promise可以将我们的回调添加到事件循环的微任务队列中
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  microTimerFunc = () => {
    p.then(flushCallbacks)
    // in problematic UIWebViews, Promise.then doesn't completely break, but
    // it can get stuck in a weird state where callbacks are pushed into the
    // microtask queue but the queue isn't being flushed, until the browser
    // needs to do some other work, e.g. handle a timer. Therefore we can
    // "force" the microtask queue to be flushed by adding an empty timer.
    
    // 这个是给hack手段
    // 因为在IOS的webView中，promise.then的回调函数会被推入到`microtask`的任务队列中
    // 但是队列却不会刷新。直到浏览器做其他工作的时候才会刷新这个队列，
    // 所以，我们可以通过添加一个空的定时器来`强制`微任务队列被刷新  
    if (isIOS) setTimeout(noop)
  }
} else {
  // fallback to macro
  // 如果不支持Promise，就直接抛弃`microtask`，直接采用宏任务`macrotask`  
  microTimerFunc = macroTimerFunc
}

/**
 * Wrap a function so that if any code inside triggers state change,
 * the changes are queued using a (macro) task instead of a microtask.
 */
// 用来绑定事件，我们在`vue`中的绑定的原生事件的回调函数所造成的状态的变化
// 都会被强制执行的在`macrotask`队列中被执行
export function withMacroTask (fn: Function): Function {
  return fn._withTask || (fn._withTask = function () {
    useMacroTask = true
    const res = fn.apply(null, arguments)
    useMacroTask = false
    return res
  })
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  // 将所有添加的任务保存到callbacks数组中
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  // 如果不是pendding（在执行状态），运行任务  
  if (!pending) {
    pending = true
     // 判断使用哪种任务（宏任务，微任务）运行添加的队列任务 
    if (useMacroTask) {
      macroTimerFunc()
    } else {
      microTimerFunc()
    }
  }
  // $flow-disable-line
  // 如果我们在调用nextTick的时候没有传递回调函数，这个方法会返回一个promise
  // 这就是这个方法的第二种用法  
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```

一开始，文件内申明了`microTimerFunc`和`macroTimerFunc`两个变量，他们分别对应的就是`micro task`的函数和`macro task`的函数。

- 对于`macro task`的实现，优先检测是否支持原生的`setImmediate`，这个Api只有高版本的IE和Edge和node.js 0.10.0+才支持。如果不支持的话在去检测`MessageChannel`，如果也不支持的话就降级为`settimeout 0`；
- 对于`micro task`的实现，则检测浏览器是否支持`Promise`，不支持的话直接指向`macro task`的实现



文件内部对外暴露了两个函数，一个是`nextTick`，另外一个是`withMacroTask`。这里先看`nextTick`函数。

#### nextTick

在派发更新的时候，我们执行了`nextTick(flushSchedulerQueue)`。函数实现很简单，把传入的回调函数`cb`压入`callbakcs`数组中，最后一次性根据`useMacroTask`条件执行`macroTimerFunc`或者是`microTimeFunc`，而他们都会在下一个**tick**执行`flushCallbacks`，`flushCallbacks`逻辑很简单，就是对`callbacks`遍历。然后执行相应的回调函数。

需要注意的是，这里使用`callbacks`而不是直接在`nextTick`中执行回调函数的原因是保证在同一个**tick**内多次执行`nextTick`，不会开启多个异步任务，而把这些异步任务都压成一个同步任务，在下一个**tick**执行完毕。

这里说的只是`nextTick`传参数的一种用法，`nextTick`还有另外一种不传参数的用法。

```typescript
// $flow-disable-line
  // 如果我们在调用nextTick的时候没有传递回调函数，这个方法会返回一个promise
  // 这就是这个方法的第二种用法  
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
```

第二种用法是当不传`nextTick`不传`cb`回调的时候，`nextTick`会提供一个`Promise`化的调用。比如：

```js
nextTick().then(() => {})
```

当`_resolve`函数执行，就会调到`then`的逻辑。

#### withMacroTask

它是对函数做了一层包装，确保函数执行的过程中对数据的任意修改，触发变化执行`nextTick`的时候强制走`macroTimeFunc`。比如对于一些DOM交互时间，如`v-on`绑定的时间回调函数的处理。会强制走`macro task`