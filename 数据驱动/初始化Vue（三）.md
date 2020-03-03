### new Vue的过程

 入口代码：src/core/instance/index.js
```js
function Vue (options) {
    if (process.env.NODE_ENV !== 'production' && !(this instanceof Vue)) {
        warn('Vue is a constructor and should be called with the `new` keyword')
    }
    this._init(options) // 调用_init函数，进行初始化Vue
} 
```
从这里我们可以看到，这里vue实际上是封装的一个类，需要通过new关键字来实例化。然后调用了this._init私有方法进行Vue的初始化。
> 代码位置：src/core/instance/init.js

```typeScript
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
  // a uid
  vm._uid = uid++

  let startTag, endTag
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    startTag = `vue-perf-start:${vm._uid}`
    endTag = `vue-perf-end:${vm._uid}`
    mark(startTag)
  }

  // a flag to avoid this being observed
  vm._isVue = true
  // merge options
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    // 合并配置
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  /* istanbul ignore else */
  if (process.env.NODE_ENV !== 'production') {
  // 初始化Proxy
    initProxy(vm)
  } else {
    vm._renderProxy = vm
  }
  // expose real self
  vm._self = vm
  initLifecycle(vm) // 初始化生命周期
  initEvents(vm) // 初始化事件中心
  initRender(vm) // 初始化渲染
  callHook(vm, 'beforeCreate')
  initInjections(vm) // resolve injections before data/props
  initState(vm)
  initProvide(vm) // resolve provide after data/props
  callHook(vm, 'created')

  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    vm._name = formatComponentName(vm, false)
    mark(endTag)
    measure(`vue ${vm._name} init`, startTag, endTag)
  }
    
  if (vm.$options.el) { // 如果存在dom，就将dom挂载在VM，将挂载目标就是把模板渲染成最终的DOM
    vm.$mount(vm.$options.el)
  }
}
```
这里可以看到_init()，方法是定义在Vue的原型上的。Vue的初始化主要干了几件事，合并配置、初始生命周期、事件中心、渲染、data、props、computed、watcher等
