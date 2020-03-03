### Vue实例挂载的实现

compiler版本
> 文件位置：src/platform/web/entry-runtime-with-compiler.js

```typeScript
const mount = Vue.prototype.$mount
Vue.prototype.$mount = function (
  el?: string | Element,
  hydrating?: boolean
): Component {
  // query函数处理传入的el，判断是否是string，还是dom
  // 如果是string，调用querySelector,找到返回dom，没有找到报错，并返回一个div
  // 如果el本身就是dom，直接返回el
  el = el && query(el) 

  /* istanbul ignore if */
  // 如果el是body或者是html的话，会直接报错
  // 所以，我们一般在定义的时候，一般用的div或者是template
  if (el === document.body || el === document.documentElement) {
    process.env.NODE_ENV !== 'production' && warn(
      `Do not mount Vue to <html> or <body> - mount to normal elements instead.`
    )
    return this
  }

  const options = this.$options
  // resolve template/el and convert to render function
  if (!options.render) { // 判断是否定义了render方法
    let template = options.template
    if (template) { // 判断是否写了template
      if (typeof template === 'string') { // 判断template是否是字符串
        if (template.charAt(0) === '#') { // 判断template的第0位是否是#
          template = idToTemplate(template) // 获取template的Id
          /* istanbul ignore if */
          if (process.env.NODE_ENV !== 'production' && !template) { // 如果不是生产环节&&templateId为空，抛出警告
            warn(
              `Template element not found or is empty: ${options.template}`,
              this
            )
          }
        }
      } else if (template.nodeType) { // 如果template是dom，判断dom类型，并获取html内容
        template = template.innerHTML
      } else {
        if (process.env.NODE_ENV !== 'production') {
          warn('invalid template option:' + template, this)
        }
        return this
      }
    } else if (el) { // 如果template为空，el dom存在
      // getOutHtml方法，判断el.outhtml是否存在，存在直接返回el.outhtml  --> outHtml:把当前标签的全部内容包括标签本身也全部取出来了
      // 如果不存在，创建一个div，并向div里面添加el的所有属性和值
      // 注意：tmeplate是字符串
      template = getOuterHTML(el) 
    }
    
    // 编译过程，此处忽略
    if (template) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile')
      }

      const { render, staticRenderFns } = compileToFunctions(template, {
        shouldDecodeNewlines,
        shouldDecodeNewlinesForHref,
        delimiters: options.delimiters,
        comments: options.comments
      }, this)
      options.render = render
      options.staticRenderFns = staticRenderFns

      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
        mark('compile end')
        measure(`vue ${this._name} compile`, 'compile', 'compile end')
      }
    }
  }
  // 调用mount方法挂载
  return mount.call(this, el, hydrating)
}
```
首先缓存了原型行的`$mount`方法，在重新定义了该方法。

- 首先对`el`做了限制，Vue不能挂载在`body`，`html`这样的根节点上
- 如果没有定义`render`方法，则会把`el`或`template`字符串转化为`render`方法，在Vue2.0版本中所有Vue组件都需要通过`render`方法来实现

无论是单文件.vue的方式开发组件，还是写`el`或`template`属性，最终都是转化为`render`方法，这个过程就是Vue的一个在线编译过程，它是调用`compileToFunctions`方法实现的，会调用原先原型上的`$mount`方法挂载。

原型上的`$mount`方法在`src/platform/web/runtion/index.js`中定义的，之所以这样设计是为了复用，因为在`runtime only`版本的Vue直接使用。

```typeScript
// public mount method
Vue.prototype.$mount = function (
  el?: string | Element,  // 挂载的元素，可以为字符串，也可以是Dom对象
  hydrating?: boolean
): Component {
  el = el && inBrowser ? query(el) : undefined  // 对el做一次校正，runtime only版本会直接执行这一步
  return mountComponent(this, el, hydrating)
}
```
`$mount`方法支持传入2个对象参数，
- 第一个`el`，表示挂载的元素，可以是字符串，也可以是Dom对象，如果是字符串在浏览器环境会调用`query`方法转换成DOM对象。
- 第二个参数是和服务端渲染相关，在浏览器环境下，我们不需要传入第二个参数

`$mount`方法实际上回去调用`mountComponent`方法
> 代码路径：`src/core/instance/lifecycle.js`中

```typeScript
export function mountComponent (
  vm: Component,
  el: ?Element,
  hydrating?: boolean
): Component {
  vm.$el = el  // 将el做缓存
  if (!vm.$options.render) { // 如果没有render函数，也就是说，template没有正确转化为render函数
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') ||
        vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
          'compiler is not available. Either pre-compile the templates into ' +
          'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn(
          'Failed to mount component: template or render function not defined.',
          vm
        )
      }
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
  // 执行_update，
    updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  // 渲染Watcher
  new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```
`mountComponent`核心就是先调用`vn_render`方法先生成虚拟Node，在实例化一个渲染`Watcher`，在它的回调函数中调用`updateComponent`方法，最终调用`vm_update`更新DOM

`Watcher`在这里起了两个作用，一个是初始化的时候会执行回调函数，另一个是当`VM`示例中的监测的数据发生变化的时候执行回调函数。

函数最后判断为根节点的时候设置`vm._isMounted`为`true`，表示这个示例挂载了，同时执行`mounted`钩子函数。这里注意`vm.$vnode`表示Vue实例的父虚拟Node，所有它为`null`则表示当前的根Vue的实例
