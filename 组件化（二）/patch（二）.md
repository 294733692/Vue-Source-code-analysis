### 组件patch

当我们通过`_createComponent`创建了组件`VNode`，接下来会走到`vm._update`，执行`vm.__patch__`去把`VNode`转化为真正的DOM，上一次分析了普通VNode的渲染，组件VNode和普通VNode有什么不同呢。

Patch的过程中会调用`createElm`创建元素节点，这个函数定义在

> src/core/vdom/patch.js

```typescript
function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  // ...
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }
  // ...
}
```

这里去除了多余代码，值保留了关键的核心代码，这里判断了`createComponent(vnode, insertedVnodeQueue, parentElm, refElm)`的返回值，如果返回true，则直接结束，接下来看看`createComponet`函数的实现。

```typescript
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
  // 这里是keepAlive的逻辑，暂时可以不用关注
  if (isDef(i)) {
    const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
    if (isDef(i = i.hook) && isDef(i = i.init)) {
      i(vnode, false /* hydrating */)
    }
    // after calling the init hook, if the vnode is a child component
    // it should've created a child instance and mounted it. the child
    // component also has set the placeholder vnode's elm.
    // in that case we can just return the element and be done.
    if (isDef(vnode.componentInstance)) {
      initComponent(vnode, insertedVnodeQueue)
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }
  }
}
```

这里对`vnode.data`做了一些判断，如果`vnode`是一个组件`vnode`，那么会满足判断条件，并且得到`i`，就是`init`钩子函数，上节分析创建组件`vnode`的时候，合并钩子函数中就包含了`init`钩子函数。定义在

> src/core/vnode/create-component.js

```typescript
init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
  // if这里的逻辑是keepAlive的逻辑，暂时忽略  
  if (
    vnode.componentInstance &&
    !vnode.componentInstance._isDestroyed &&
    vnode.data.keepAlive
  ) {
    // kept-alive components, treat as a patch
    const mountedNode: any = vnode // work around flow
    componentVNodeHooks.prepatch(mountedNode, mountedNode)
  } else {
    // 返回的实际就是vnode示例  
    const child = vnode.componentInstance = createComponentInstanceForVnode(
      vnode, // 这里的vnode指的是组件vnode
      activeInstance
    )
    child.$mount(hydrating ? vnode.elm : undefined, hydrating)
  }
},
```

这里`init`实现的逻辑也很简单，暂时不考虑`keepAlive`的逻辑，它是通过`createComponentInstanceForVnode`方法创建了一个`Vue`实例，然后调用`$mount`方法挂载子组件。

接下来看`createComponentInstanceForVnode`的实现

```typescript
export function createComponentInstanceForVnode (
//vnode: 组件vnode
// parent: 实际上就是当前vm的实例
  vnode: any, // we know it's MountedComponentVNode but flow doesn't
  parent: any, // activeInstance in lifecycle state
): Component {
  const options: InternalComponentOptions = {
    _isComponent: true,
    _parentVnode: vnode,
    parent
  }
  // check inline-template render functions
  const inlineTemplate = vnode.data.inlineTemplate
  if (isDef(inlineTemplate)) {
    options.render = inlineTemplate.render
    options.staticRenderFns = inlineTemplate.staticRenderFns
  }
  return new vnode.componentOptions.Ctor(options)
}
```

这里`createComponentInstanceForVnode`函数构造的一个内部组件的参数，然后执行`new vnode.componentOptions.Ctor(options)`。这里的`vnode.componentOptions.Ctor`对应的就是子组件的构造函数。这一节分析组件vnode的创建过程中，分析了`Sub`实际上就是继承了Vue的一个构造函数，相当于`new Sub(options)`。

这里几个关键参数需要注意，

- _isComponent ： 为`true`表明这是一个组件，为`fasle` 说明不是组件
- _parentVnode：表示当前激活的组件实例

所以，子组件的实例化实际就是在这个时机执行的，并且它会执行实例的`init`方法。这个过程有一些和之前不同的地方。代码在

> src/core/instance/init.js

```typescript
Vue.prototype._init = function (options?: Object) {
  const vm: Component = this
  // merge options
  if (options && options._isComponent) {
    // optimize internal component instantiation
    // since dynamic options merging is pretty slow, and none of the
    // internal component options needs special treatment.
    initInternalComponent(vm, options)
  } else {
    vm.$options = mergeOptions(
      resolveConstructorOptions(vm.constructor),
      options || {},
      vm
    )
  }
  // ...
  if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  } 
}
```

这里和之前有了一些变化，组件vnode的时候，在合并`options`的时候，`_isComponent`为`true`，所以走到了`initInternalComponent`，

```typescript
xport function initInternalComponent (vm: Component, options: InternalComponentOptions) {
  const opts = vm.$options = Object.create(vm.constructor.options)
  // doing this because it's faster than dynamic enumeration.
  const parentVnode = options._parentVnode
  opts.parent = options.parent
  opts._parentVnode = parentVnode

  const vnodeComponentOptions = parentVnode.componentOptions
  opts.propsData = vnodeComponentOptions.propsData
  opts._parentListeners = vnodeComponentOptions.listeners
  opts._renderChildren = vnodeComponentOptions.children
  opts._componentTag = vnodeComponentOptions.tag

  if (options.render) {
    opts.render = options.render
    opts.staticRenderFns = options.staticRenderFns
  }
}
```

这里我们只需要记住：`opts.parent = options.parent`、`opts._parentVnode = parentVnode`，它们是把之前通过`createComponentInstanceForVnode`函数传入的几个参数合并到内部的`$options`里面去了。

最后在看`_init`最后执行的代码

```typescript
if (vm.$options.el) {
    vm.$mount(vm.$options.el)
  } 
```

由于初始化组件的时候，不用传`el`的，因此组件是自己接管了`$mount`的过程。回到组件`init`的过程中。`componentVnodeHooks`的`init`钩子函数，在完成实例化的`_init`后，接着会执行`child.$mount(hydrating ? vnode .elm : undefined, hydrating)`。这里`hydrating`为true，一遍是服务器渲染的情况，我们只考虑客户端让渲染，所有这里`$mount`相当于执行`child.$mount(undefined, false)`，最终它会调用`mountComponent`函数，进而执行`vm._render`方法

```typescript
Vue.prototype._render = function (): VNode {
  const vm: Component = this
  const { render, _parentVnode } = vm.$options

  // set parent vnode. this allows render functions to have access
  // to the data on the placeholder node.
  vm.$vnode = _parentVnode
  // render self
  let vnode
  try {
    vnode = render.call(vm._renderProxy, vm.$createElement)
  } catch (e) {
    // ...
  }
  // set parent
  // 这里的_parentVnode相当于当前组件的父VNode
  vnode.parent = _parentVnode
  return vnode
}
```

这里需要注意的是，`_parentVnode`相当于当前组建的父`VNode`，而`render`函数生成的`VNode`是当前组件渲染生成的`vnode`，`vnode`的`parent`指向了`_parentVnode`，也就是`vm.$vnode`，也就是父子关系。



在执行完`vm._render`后，接下来就是去执行`vm._update`去渲染VNode了。组件VNode渲染函数定义在

> src/code/instance/lifecycle.js

```typescript
export let activeInstance: any = null
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode
  const prevActiveInstance = activeInstance
  activeInstance = vm
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode)
  }
  activeInstance = prevActiveInstance
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```

`_update`过程中有几个比较关键的代码，首先`vm._vnode = vnode`的逻辑，这里的`vnode`是通过`vm._render()`返回的组件渲染`VNode`，`vm._vnode`和`vm.$vnode`的关系就是一个父与子的关系，用代码理解就是`vm._vnode.parent = vn.$vnode`。

```typescript
export let activeInstance: any = null
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
    // ...
    const prevActiveInstance = activeInstance
    activeInstance = vm
    if (!prevVnode) {
      // initial render
      vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */)
    } else {
      // updates
      vm.$el = vm.__patch__(prevVnode, vnode)
    }
    activeInstance = prevActiveInstance
  }
```

这里的`activeInstance`的作用就是保持当前上下文的vue实例，它是在`lifecycle`模块的全局变量，定义是`export let activeInstance: any = null`，并且在之前我们调用`createComponentInstanceForVnode`方法的时候，从`lifecycle`模块获取的，并且作为参数传入的，因为实际上JavaScript是一个单线程，Vue整个初始化是一个深度遍历的过程，在实例化子组件的过程中，它需要知道当前上下文的Vue实例是什么，并把它作为子组件的父Vue实例，之前提到过，对子组件的实例化过程会先调用`initInternalComponent(vm, options)`合并`options`，把`parent`存储在`vm.$options`中，在`mount`之前会调用`initLifecycle(vm)`方法。

```typescript
export function initLifecycle (vm: Component) {
  const options = vm.$options

  // locate first non-abstract parent
  let parent = options.parent
  if (parent && !options.abstract) {
    while (parent.$options.abstract && parent.$parent) {
      parent = parent.$parent
    }
    parent.$children.push(vm)
  }

  vm.$parent = parent
  // ...
}
```

这里可以看到`vm.$parent`就是用来保留当前`vm`的父实例，并且通过`parent.$children.push(vm)`来把当前的`vm`存储到父实例的`children`中。

在`vm._update`的过程中，把当前的`vm`赋值给`activeInstance`，同时通过`prevActiveInstance = activeInstance`用`prevActiveInstance`保留上一次的`activeInstance`。实际上`prevActiveInstance`和当前的`vm`是父子关系，当一个`vm`实例完成它所有的子树的patch或update过程后，`activeInstance`会回到父实例。这样就保证了`createComponentInstanceForVnode`整个深度遍历过程中，我们在实例化子组件的时候能传入当前子组件的父Vue实例，并在`_init`的过程中，通过`vm.$parent`把这个父子关系保留。

这就又回到`_update`，最后就是调用`__patch__`渲染VNode了