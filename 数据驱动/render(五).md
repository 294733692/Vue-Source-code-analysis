### render

Vue的`_render`方法是实例的一个私有方法，它用来把实例渲染成一个虚拟Node
> 位置 `src/core/instance/render.js`

```TypeScript
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options  // 获取到render函数和父级的虚拟节点

  // reset _rendered flag on slots for duplicate slot check
  if (process.env.NODE_ENV !== 'production') {
    for (const key in vm.$slots) {
      // $flow-disable-line
      vm.$slots[key]._rendered = false
    }
  }

  if (_parentVnode) {
    vm.$scopedSlots = _parentVnode.data.scopedSlots || emptyObject
  }

  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    // call（1,2）1为上下文，这里的vm._renderProxy，生成环境指的是VM，
    vnode = render.call(vm._renderProxy, vm.$createElement) // 重点是这句话，下面的目前可以暂时可以忽略
  } catch (e) {
    handleError(e, vm, `render`)
    // return error render result,
    // or previous vnode to prevent render error causing blank component
    /* istanbul ignore else */
    if (process.env.NODE_ENV !== 'production') {
      if (vm.$options.renderError) {
        try {
          vnode = vm.$options.renderError.call(vm._renderProxy, vm.$createElement, e)
        } catch (e) {
          handleError(e, vm, `renderError`)
          vnode = vm._vnode
        }
      } else {
        vnode = vm._vnode
      }
    } else {
      vnode = vm._vnode
    }
  }
  // return empty vnode in case the render function errored out
  if (!(vnode instanceof VNode)) {
    if (process.env.NODE_ENV !== 'production' && Array.isArray(vnode)) {
      warn(
        'Multiple root nodes returned from render function. Render function ' +
        'should return a single root node.',
        vm
      )
    }
    vnode = createEmptyVNode()
  }
  // set parent
  vnode.parent = _parentVnode
  return vnode
}
```
这段代码最关键的是`render`方法的调用，平时在开发中手写`render`的时候不多，直接使用vue脚手架搭建的，写`template`模板，在前面的`mounted`方法的实现中，会把`template`编译成`render`方法。

Vue官方文档中介绍了`render`函数的使用方法，第一个参数是`createElement`，结合下面的例子
```Vue
<div id="app">{{message}}</div>
```
相当于我们编译下面的`render`函数
```js
render: function (createElement) {
    return createElement('div', {
        attrs:{
            id: 'app'
        }
    }, this.message)
}
```
在看到`_render`函数中的`render`方法调用：
```js
vnode = render.call(vn._renderProxy, vn.$createElement)
```
这里的`vm.$createElement`方法，就相当于`render`函数中的`createElement`方法

```Ts
export function initRender (vm: Component) {
  // ...
  // bind the createElement fn to this instance
  // so that we get proper render context inside it.
  // args order: tag, data, children, normalizationType, alwaysNormalize
  // internal version is used by render functions compiled from templates
  vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)  // 模板编译的`render`方法
  // normalization is always applied for the public version, used in
  // user-written render functions.
  vm.$createElement = (a, b, c, d) => createElement(vm, a, b, c, d, true) // 用户手写的`render`方法
}
```

实际上，`vm.$createElement`方法定义是在执行`initRender`方法的时候，实际上，我们可以看到除了`vm.$createElement`方法，还有一个`vm._c`的方法，这个方法实际上是，它是当模板被编译成`render`的函数的时候被使用，而`vm.$createElement`是用户手写的`render`方法使用的，而且这两个方法支持的参数都是相同的，并且内部都调用了`createElement`方法
