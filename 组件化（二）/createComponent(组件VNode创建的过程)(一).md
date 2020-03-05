我们在看`createElement`的实现的时候，它最终会调用`_createElement`方法，其中有一段逻辑是对`tag`的判断。如果是普通的`html`标签，上一个例子是普通的`div`,则会实例化普通 `VNode`节点，否则会通过`createComponent`方法创建一个组件`VNode`

```typescript
export function _createElement (
  context: Component,
  tag?: string | Class<Component> | Function | Object,
  data?: VNodeData,
  children?: any,
  normalizationType?: number
): VNode | Array<VNode> {
  // data不能为响应式的
  if (isDef(data) && isDef((data: any).__ob__)) {
    process.env.NODE_ENV !== 'production' && warn(
      `Avoid using observed data object as vnode data: ${JSON.stringify(data)}\n` +
      'Always create fresh vnode data objects in each render!',
      context
    )
    // 返回一个注释节点
    return createEmptyVNode()
  }
  // object syntax in v-bind
  if (isDef(data) && isDef(data.is)) {
    tag = data.is
  }
  if (!tag) {
    // in case of component :is set to falsy value
    return createEmptyVNode()
  }
  // warn against non-primitive key
  // 判断data.key是一个非基础类型
  if (process.env.NODE_ENV !== 'production' &&
    isDef(data) && isDef(data.key) && !isPrimitive(data.key)
  ) {
    if (!__WEEX__ || !('@binding' in data.key)) {
      warn(
        'Avoid using non-primitive value as key, ' +
        'use string/number value instead.',
        context
      )
    }
  }
  // support single function children as default scoped slot
  if (Array.isArray(children) &&
    typeof children[0] === 'function'
  ) {
    data = data || {}
    data.scopedSlots = { default: children[0] }
    children.length = 0
  }
  // 对chilren做normalizeChildren
  if (normalizationType === ALWAYS_NORMALIZE) {
    children = normalizeChildren(children)
  } else if (normalizationType === SIMPLE_NORMALIZE) {
    children = simpleNormalizeChildren(children)
  }
  let vnode, ns
  // tag可以为string，也可以是组件形式
  if (typeof tag === 'string') {
    let Ctor
    // 对netSpace的处理
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    // 判断tag是否是html的原生标签
    // 是就创建一个平台保留标签的VNode
    if (config.isReservedTag(tag)) {
      // platform built-in elements
      vnode = new VNode(
        config.parsePlatformTagName(tag), data, children,
        undefined, undefined, context
      )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
      // component
      // 如果是组件，创建组件VNode
      vnode = createComponent(Ctor, data, context, children, tag)
    } else {
      // unknown or unlisted namespaced elements
      // check at runtime because it may get assigned a namespace when its
      // parent normalizes children
      // 如果是未识别的标签（类似于组定义组件这种，el-input这种），创建VNdode
      vnode = new VNode(
        tag, data, children,
        undefined, undefined, context
      )
    }
  } else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
  if (Array.isArray(vnode)) {
    return vnode
  } else if (isDef(vnode)) {
    if (isDef(ns)) applyNS(vnode, ns)
    if (isDef(data)) registerDeepBindings(data)
    return vnode
  } else {
    return createEmptyVNode()
  }
}
```

这里使用`vue-cli`初始化代码为例子

```Vue
import Vue form 'vue'
import App from './App.vue'

var app = new Vue({
    el: '#app',
    // 这里的h是createElement方法
	render: h => h(app)
})
```

这里传入的是一个`App`对象，本质上还是是一个`component`类型，那么，它会走到

```typescript
else {
    // direct component options / constructor
    vnode = createComponent(tag, data, context, children)
  }
```

这一块逻辑，直接通过`createComponent`方法来创建`VNode`，这个方法定义在

> `src/core/vdom/create-component.js`中

```typescript
export function createComponent (
  Ctor: Class<Component> | Function | Object | void, // 组件类型的类、函数、对象、空值
  data: ?VNodeData, // VNode相关的data
  context: Component, // 上下文，也就是当前的VN实例
  children: ?Array<VNode>, // 组件的子VNode
  tag?: string
): VNode | Array<VNode> | void {
  if (isUndef(Ctor)) {
    return
  }

  // 实际上取的VN.$options._base  
  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  // 判断如果传入的构造器是对象的话
  if (isObject(Ctor)) {
    // 就是调用vue.extend方法，将Ctor转化成新的构造器  
    Ctor = baseCtor.extend(Ctor)
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
      warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
  }

  // async component
  let asyncFactory
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  data = data || {}

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor)

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data)
  }

  // extract props
  const propsData = extractPropsFromVNodeData(data, Ctor, tag)

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  const listeners = data.on
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn

  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    const slot = data.slot
    data = {}
    if (slot) {
      data.slot = slot
    }
  }

  // install component management hooks onto the placeholder node
  installComponentHooks(data)

  // return a placeholder vnode
  const name = Ctor.options.name || tag
  const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
  )

  // Weex specific: invoke recycle-list optimized @render function for
  // extracting cell-slot template.
  // https://github.com/Hanks10100/weex-native-directive/tree/master/component
  /* istanbul ignore if */
  if (__WEEX__ && isRecyclableComponent(vnode)) {
    return renderRecyclableComponentTemplate(vnode)
  }

  return vnode
}
```

这块逻辑相对复杂，我们先看比较重要的核心代码.

- 构造子类构造函数
- 安装组件钩子函数
- 实例化`VNode`



### 构造子类构造函数

```typescript
  // 实际上取的VN.$options._base  
  const baseCtor = context.$options._base

  // plain options object: turn it into a constructor
  // 判断如果传入的构造器是对象的话
  if (isObject(Ctor)) {
    // 就是调用vue.extend方法，将Ctor转化成新的构造器  
    Ctor = baseCtor.extend(Ctor)
  }
```

我们在编写一个组件的时候，通常是创建一个普通对象

```Vue
import HelloWorld from './components/helloWorld'

export default {
	name: 'app',
	components: {
		HelloWorld
	}
}
```

可以看到这里export出了一个对象，所以会调用`createComponent`函数，会执行`baseCtor = context.$options._base`这段代码，这里的`baseCtor`实际上就`Vue`，这个定义在最开始初始化`Vue`的阶段，这段代码在

> `src/core/global-api/index.js`

中的`initGlobalAPI`函数中

```typescript
// this is used to identify the "base" constructor to extend all plain-object
// components with in Weex's multi-instance scenarios.
Vue.options._base = Vue
```

可以发现的是，我们`createComponent`函数中执行的是`context.$options._base`，这块逻辑，而这段代码是`Vue.options._base`，`Vue`和`context`明显不相同。

实际上在

> src/core/instance/init.js

中Vue的原型上有这么一段代码

```typescript
vm.$options = mergeOptions({
    resolveConstructorOptions(vm.constructor)
    options || {}
    vm
})
```

通过这么一段代码，将`Vue`上的一些`options`给扩展到`vm.$option`上，所有，我们就可以通过`vm.$options._base`拿到`Vue`这个构造函数了。`mergeOptions`的实现，暂时先可以不用管，现在只需要理解它的功能是把`Vue`构造函数的`options`和用户传入的`options`做一层合并，到`vm.$options`上。



接下看`baseCtor.extend`函数。实际上`baseCtor.extend`函数就是`Vue.extend`函数，这段代码定义在

> src/code/global-api/extend.js

```typescript
/**
 * Class inheritance
 */
Vue.extend = function (extendOptions: Object): Function {  // 传入一个对象，返回一个函数
  extendOptions = extendOptions || {}
  // 这里需要注意的是，this不是指向`VM`,而是指向Vue，这里的extend是一个静态方法，谁调用，就指向谁  
  const Super = this
  const SuperId = Super.cid // Vue的cid
  // 实际上是做了一层缓存的优化
  const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
  // 当多次引入同一个组件的时候，如果这个组件是由同一个父构造器构建，那么只会执行一次extend逻辑，不用重复执行初始化构造器的逻辑
  if (cachedCtors[SuperId]) {
    return cachedCtors[SuperId]
  }

  // 这里的name实际上就是组件name  
  const name = extendOptions.name || Super.options.name 
  if (process.env.NODE_ENV !== 'production' && name) {
    // 检验是否是组件名称和原生标签是否冲突
    validateComponentName(name)
  }

  // 定义子构造函数  
  const Sub = function VueComponent (options) {
    this._init(options)
  }
  // 实际上就是把子的构造原型继承父的构造原型
  Sub.prototype = Object.create(Super.prototype)
  // 把构造器constructor指向自身
  Sub.prototype.constructor = Sub
  // 构造器的唯一标识
  Sub.cid = cid++
  // 自身的option和Vue的options做了一层合并  
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  Sub['super'] = Super

  // For props and computed properties, we define the proxy getters on
  // the Vue instances at extension time, on the extended prototype. This
  // avoids Object.defineProperty calls for each instance created.
  if (Sub.options.props) {
    initProps(Sub)
  }
  if (Sub.options.computed) {
    initComputed(Sub)
  }

  // allow further extension/mixin/plugin usage
  // 将一些全局的静态方法赋值给sub
  Sub.extend = Super.extend
  Sub.mixin = Super.mixin
  Sub.use = Super.use

  // create asset registers, so extended classes
  // can have their private assets too.
  ASSET_TYPES.forEach(function (type) {
    Sub[type] = Super[type]
  })
  // enable recursive self-lookup
  if (name) {
    Sub.options.components[name] = Sub
  }

  // keep a reference to the super options at extension time.
  // later at instantiation we can check if Super's options have
  // been updated.
  Sub.superOptions = Super.options
  Sub.extendOptions = extendOptions
  Sub.sealedOptions = extend({}, Sub.options)

  // cache constructor
  // 将这些东西给缓存下来，并赋值给它  
  cachedCtors[SuperId] = Sub
  return Sub
}
```

​	实际上`Vue.extend`的作用就是构造了一个`Vue`的子类，它使用了原型继承的方式把一个纯对象转换为一个继承于`Vue`的构造器`Sub`并返回，然后对`Sub`这个对象本身扩展了一些属性，例如扩展了`options`、添加了全局API；并且对配置中的`props`和`computed`做了初始化工作；最后对`Sub`函数做了缓存，避免多次执行`Vue.extend`的时候对同一个组件重复构造。

​	这样当我们去实例化`Sub` 的时候，就会执行`this._init`逻辑再次走到了`Vue`实例的初始化逻辑。

```typescript
const Sub = function VueComponent (options) {
    this._init(options)
}
```

### 安装钩子函数

```typescript
// install component management hooks onto the placeholder node
installComponentHooks(data)
```

Vue.js使用的是 Virtual DOM ，参考了开源库[snabbdom](https://github.com/snabbdom/snabbdom)，这个库的特点就是在VNode的patch流程中对外暴露各种时机的钩子函数。方便我们做一些额外的事情，Vue.js也充分利用了这一点，在初始化一个`Component`类型的`VNode`的过程中实现了几个钩子函数。

```typescript
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },

  prepatch (oldVnode: MountedComponentVNode, vnode: MountedComponentVNode) {
    const options = vnode.componentOptions
    const child = vnode.componentInstance = oldVnode.componentInstance
    updateChildComponent(
      child,
      options.propsData, // updated props
      options.listeners, // updated listeners
      vnode, // new parent vnode
      options.children // new children
    )
  },

  insert (vnode: MountedComponentVNode) {
    const { context, componentInstance } = vnode
    if (!componentInstance._isMounted) {
      componentInstance._isMounted = true
      callHook(componentInstance, 'mounted')
    }
    if (vnode.data.keepAlive) {
      if (context._isMounted) {
        // vue-router#1212
        // During updates, a kept-alive component's child components may
        // change, so directly walking the tree here may call activated hooks
        // on incorrect children. Instead we push them into a queue which will
        // be processed after the whole patch process ended.
        queueActivatedComponent(componentInstance)
      } else {
        activateChildComponent(componentInstance, true /* direct */)
      }
    }
  },

  destroy (vnode: MountedComponentVNode) {
    const { componentInstance } = vnode
    if (!componentInstance._isDestroyed) {
      if (!vnode.data.keepAlive) {
        componentInstance.$destroy()
      } else {
        deactivateChildComponent(componentInstance, true /* direct */)
      }
    }
  }
}

const hooksToMerge = Object.keys(componentVNodeHooks)

function installComponentHooks (data: VNodeData) {
  const hooks = data.hook || (data.hook = {})
  for (let i = 0; i < hooksToMerge.length; i++) {
    const key = hooksToMerge[i]
    const existing = hooks[key]
    const toMerge = componentVNodeHooks[key]
    if (existing !== toMerge && !(existing && existing._merged)) {
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}

function mergeHook (f1: any, f2: any): Function {
  const merged = (a, b) => {
    // flow complains about extra args which is why we use any
    f1(a, b)
    f2(a, b)
  }
  merged._merged = true
  return merged
}
```

整个`installComponentHooks`的过程就是把`componentVNodeHooks`的钩子函数合并到`data.hook`中，在`VNode`执行`patch`的过程中执行相关的钩子函数，在合并过程中，如果某个时机的钩子已经存在`data.hook`中，那么通过`mergeHook`函数做合并，合并逻辑挺简单的。就是在最终执行的时候，一次执行这两个钩子函数

### 实例化VNode

```typescript
const name = Ctor.options.name || tag
const vnode = new VNode(
`vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
data, undefined, undefined, undefined, context,
{ Ctor, propsData, listeners, tag, children },
asyncFactory
)
return vnode
```

这一步非常简单，就是通过 `new VNode`实例化一个·`vnode`并返回。这里需要注意的是，这里的`vnode`和普通的元素节点`vnode`不同，组件的`vnode`是没有`children`属性的。



总结

`createComponent`函数的实现，里面有三个重要的实现逻辑：构造子类构造函数、安装组件钩子函数和实例化`vnode`。

`createComponent`后返回的是组件`vnode`，它也会走到`vn._update`方法，进而执行`patch`函数
