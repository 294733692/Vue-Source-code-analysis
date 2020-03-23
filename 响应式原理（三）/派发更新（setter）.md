上一节分析了`getter`响应式数据依赖收集的过程，收集的目的就是为了当我们修改数据的时候，可以对相关依赖派发更新。

```typescript
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  // 拿到obj上的属性描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)
  // configurable 属性描述符为true，才能修改数据对象
  // 如果configurable为false，那么不能修改数据对象，直接return
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  // 保存property对象的get和set函数  
  const getter = property && property.get
  const setter = property && property.set
  // 当只传递了两个参数的时候，说明没有传递第三个参数val
  if ((!getter || setter) && arguments.length === 2) {
    // 那么这个时候需要根据key主动去对象获取相依的值  
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  // 重新定义getter和setter方法，覆盖原来的set和get方法。
  // 上面将原有方法get就set缓存，并在重新定义的get和set方法中调用缓存的get和set函数
  // 这样做到了不影响原有属性的读写
 Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // ...
    set: function reactiveSetter (newVal) {
      // 判断是否存在getter，存在返回值，不存在返回val  
      const value = getter ? getter.call(obj) : val
      /* eslint-disable no-self-compare */
      // 新旧值判断 || 
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      // customSetter函数判断（作用：用来打印赋值属性）  
      if (process.env.NODE_ENV !== 'production' && customSetter) {
        customSetter()
      }
      if (setter) {
        // 设置正确属性  
        setter.call(obj, newVal)
      } else {
        val = newVal
      }
      childOb = !shallow && observe(newVal)
      // 深度观测，依赖收集  
      dep.notify()
    }
  })
}
```

`setter`有两个关键点

- `childOb = !shallow && observe(newVal)`：如果`shallow`为false，会把新设置的值变成一个响应式对象。
- `dep.notity`：通知所有的订阅者

### 过程分析：

当我们在组件中对响应的数据做了修改，就会触发`setter`的逻辑，最后调用`dep.notity()`方法。它是`Dep`实例方法，定义在

> ​	src/core/observer/dep.js

```typescript
class Dep {
  // ...
  notify () {
  // stabilize the subscriber list first
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}
```

这里过程其实很简单，就是遍历`subs`，也就是`Watcher`的实例数组，然后调用每个`Watcher`的`update()`方法。`update`方法定义在

> src/core/observer/watcher.js

```typescript
class Watcher {
  // ...
  update () {
    /* istanbul ignore else */
    if (this.computed) {
      // A computed property watcher has two modes: lazy and activated.
      // It initializes as lazy by default, and only becomes activated when
      // it is depended on by at least one subscriber, which is typically
      // another computed property or a component's render function.
      if (this.dep.subs.length === 0) {
        // In lazy mode, we don't want to perform computations until necessary,
        // so we simply mark the watcher as dirty. The actual computation is
        // performed just-in-time in this.evaluate() when the computed property
        // is accessed.
        this.dirty = true
      } else {
        // In activated mode, we want to proactively perform the computation
        // but only notify our subscribers when the value has indeed changed.
        this.getAndInvoke(() => {
          this.dep.notify()
        })
      }
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }
}  
```

对于`wathcer`的不同状态，会执行不同的逻辑，在一般组件数据更新的场景，会走到最后一个`queueWatcher(this)`逻辑，该方法定义在

> ​	src/core/observer/scheduler.js

```typescript
const queue: Array<Watcher> = []
// 保存Watcher,用于判断watcher是否存在
// 用于保证同一个`watcher`只能被添加一次
let has: { [key: number]: ?true } = {}
let waiting = false
let flushing = false
/**
 * Push a watcher into the watcher queue.
 * Jobs with duplicate IDs will be skipped unless it's
 * pushed when the queue is being flushed.
 */
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
   // 如果之前watcher不存在，就进去  
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // queue the flush
    // 保证该逻辑只能执行一次  
    if (!waiting) {
      waiting = true
      nextTick(flushSchedulerQueue)
    }
  }
}
```

这里使用了队列的方法，这也是Vue在做派发更新的一个优化点，它并不会每次数据改变都触发`Watcher`的回调，而是把这些`Watcher`先添加到一个队列里，然后在`nextTick`后执行`flushSchedulerQueue`。`flushSchedulerQueue`定义在

> ​	src/core/boserver/scheduler.js

```typescript
let flushing = false
let index = 0
/**
 * Flush both queues and run the watchers.
 */
function flushSchedulerQueue () {
  flushing = true
  let watcher, id

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort((a, b) => a.id - b.id)

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index]
    if (watcher.before) {
      watcher.before()
    }
    id = watcher.id
    has[id] = null
    watcher.run()
    // in dev build, check and stop circular updates.
    if (process.env.NODE_ENV !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? `in watcher with expression "${watcher.expression}"`
              : `in a component render function.`
          ),
          watcher.vm
        )
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  const activatedQueue = activatedChildren.slice()
  const updatedQueue = queue.slice()

  resetSchedulerState()

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue)
  callUpdatedHooks(updatedQueue)

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush')
  }
}
```

这里有几个地方的逻辑

#### 队列排序

`queue.sort((a, b) => a.id - b.id)`对队列做了从小到大的排序，这么做主要有以下几点确保

- 组件的更新有父到子：因为父组件的创建过程要先于子组件的，所以`Watcher`的创建过程也是先父后子，执行顺序也是先父后子
- 用户的自定义`watcher`要优先渲染`Watcher`执行：因为用户自定义`watcher`是在渲染`Watcher`之前创建的
- 如果一个组件在父组件的`watcher`执行期间被销毁，那么它对应的`watcher`执行都可以被跳过，所以父组件的`watcher`应该先执行。

#### 队列遍历

在对`queue`排序后，接着就是要对它做遍历，拿到对应的`watcher`，执行`watcher.run()`。这里需要到，在遍历的时候每次都会`queue.length`求值，因为在`watcher.run()`的时候，很可能用户会再次添加到新的`watcher`，这样会再次执行到`queueWatcher`，如下：

```typescript
export function queueWatcher (watcher: Watcher) {
  const id = watcher.id
  if (has[id] == null) {
    has[id] = true
    if (!flushing) {
      queue.push(watcher)
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      let i = queue.length - 1
      while (i > index && queue[i].id > watcher.id) {
        i--
      }
      queue.splice(i + 1, 0, watcher)
    }
    // ...
  }
}
```

这里可以看到，这个时候`flushing`为true，就会执行到else的逻辑，然后就会从后向前找，找到第一个待插入`watcher`的id比当前中`watcher`的**id**大的位置，把`watcher`按照**id**的插入到队列中，因此`queue`的长度发生了变化。

#### 状态恢复

这个过程就是执行`resetchedulerState`函数。它定义在:

> ​	src/core/observer/scheduler.js

```typescript
const queue: Array<Watcher> = []
let has: { [key: number]: ?true } = {}
let circular: { [key: number]: number } = {}
let waiting = false
let flushing = false
let index = 0
/**
 * Reset the scheduler's state.
 */
function resetSchedulerState () {
  index = queue.length = activatedChildren.length = 0
  has = {}
  if (process.env.NODE_ENV !== 'production') {
    circular = {}
  }
  waiting = flushing = false
}
```

这块代码的逻辑非常简单，就是把这些控制流程状态的一些变量恢复成`初始值`，把`watcher`队列清空。

接下来看看`watcher.run()`的逻辑，它的定义

> ​	src/core/obserber/watcher.js

```typescript
class Watcher {
  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    if (this.active) {
      this.getAndInvoke(this.cb)
    }
  }

  getAndInvoke (cb: Function) {
    const value = this.get()
    if (
      value !== this.value ||
      // Deep watchers and watchers on Object/Arrays should fire even
      // when the value is the same, because the value may
      // have mutated.
      isObject(value) ||
      this.deep
    ) {
      // set new value
      const oldValue = this.value
      this.value = value
      this.dirty = false
      if (this.user) {
        try {
          cb.call(this.vm, value, oldValue)
        } catch (e) {
          handleError(e, this.vm, `callback for watcher "${this.expression}"`)
        }
      } else {
        cb.call(this.vm, value, oldValue)
      }
    }
  }
}
```

这里的`watcher.run()`方法实际就是执行了`getAndInvoke`方法，并传入了`watcher`的回调函数，`getAndInvoke`函数，

先是通过`this.get()`得到它当前的值。然后做判断，如果新旧值不相等、新值是对象、`deep`模式下任何一个条件，则执行`watcher`的回调。**注意**回调函数执行的时候会把第一个和第二个参数传入新值`value`和旧值`oldValue`，这就是我么在添加自定义`watcher`的时候能在回调函数的参数中，拿到新旧值的原因。

那么对于渲染`watcher`，它在执行`this.get()`方法求值的时候，会执行`getter`方法

```typescript
updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

所以这就是当我们去修改组件相关的响应式数据的时候，会触发组件重新渲染的原因。接着就会重新执行`patch`的过程，但是和首次渲染有点区别。