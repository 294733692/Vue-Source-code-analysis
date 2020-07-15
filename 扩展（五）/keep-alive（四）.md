### keep-alive

- Props：

  - `include`：字符串或正则表达式。只有名称匹配的组件才会被缓存
  - `exclude`：字符串或正则表达式。任何名称匹配的组件都不会被缓存。
  - `max`：数字。最多可以缓存多少组件实例

- 用法

  `<keep-alive>`包裹动态组件时，会缓存不活动的组件实例，而不是销毁它们。`keep-alive`是一个抽象组件：它自身不会渲染一个DOM元素，也不会出现在组件的父组件链中。

  当组件在`<keep-alive>`内被切换，它的`activeted`和`deactivated`两个生命周期钩子函数会被对应执行

具体用法可以查看Vue的官方文档，[keep-alive](https://cn.vuejs.org/v2/api/#keep-alive)，这里我们来来`keep-alive`的具体实现



#### 内置组件

`<keep-alive`是 Vue源码中实现的一个组件，也就是说Vue源码不仅实现了一套组件化的机制，也实现了一些内置组件，该组件的定义在：

> src/core/components/keep-alive.js

```typescript
export default {
  name: 'keep-alive,
  abstract: true, // 判断当前组件虚拟DOM是否渲染成真实DOM
  
  // 可接受参数  
  props: {
    include: patternTypes, // 字符串或正则表达式。只有名称匹配的组件才会被缓存，即白名单
    exclude: patternTypes, // 字符串或正则表达式。任何名称匹配的组件都不会被缓存。即黑名单
    max: [String, Number]  // 数字。最多可以缓存多少组件实例
  },

  created () {
    this.cache = Object.create(null) // 创建一个空对象，缓存虚拟DOM
    this.keys = [] // 缓存的虚拟DOM的键集合
  },

  destroyed () {
    for (const key in this.cache) {
      // 删除所有缓存
      pruneCacheEntry(this.cache, key, this.keys)
    }
  },

  mounted () {
    // 实现监听黑白名单的变动  
    this.$watch('include', val => {
      pruneCache(this, name => matches(val, name))
    })
    this.$watch('exclude', val => {
      pruneCache(this, name => !matches(val, name))
    })
  },

  render () {
    const slot = this.$slots.default
    const vnode: VNode = getFirstComponentChild(slot)
    const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
    if (componentOptions) {
      // check pattern
      const name: ?string = getComponentName(componentOptions)
      const { include, exclude } = this
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      const { cache, keys } = this
      const key: ?string = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (#3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
        : vnode.key
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance
        // make current key freshest
        remove(keys, key)
        keys.push(key)
      } else {
        cache[key] = vnode
        keys.push(key)
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode)
        }
      }

      vnode.data.keepAlive = true
    }
    return vnode || (slot && slot[0])
  }
}
```

从这里我们可以看到，`keep-alive`组件的实现也是一个对象，这里我们需要注意的它有一个属性`abstract`为`true`，是一个抽象组件，但是`vue`的文档上没有提到一个概念，实际上它在组件实例建立父子关系的时候会被忽略，这个发生在`initLifecycle`的过程中

```typescript
// locate first non-abstract parent
let parent = options.parent
if (parent && !options.abstract) {
  while (parent.$options.abstract && parent.$parent) {
    parent = parent.$parent
  }
  parent.$children.push(vm)
}
vm.$parent = parent
```

`<keep-alive>` 在 `created` 钩子里定义了 `this.cache` 和 `this.keys`，本质上它就是去缓存已经创建过的 `vnode`。它的 `props` 定义了 `include`，`exclude`，它们可以字符串或者表达式，`include` 表示只有匹配的组件会被缓存，而 `exclude` 表示任何匹配的组件都不会被缓存，`props` 还定义了 `max`，它表示缓存的大小，因为我们是缓存的 `vnode` 对象，它也会持有 DOM，当我们缓存很多的时候，会比较占用内存，所以该配置允许我们指定缓存大小。

从上面代码我们可以发现，`keep-alive`在它的生命周期内定义了三个钩子函数

- created

  初始化两个对象分别缓存VNode（虚拟DOM）和VNode对应的键集合

- destroyed

  删除`this.cache`中的缓存的VNode实例，我们看到，这不是简单的将`this.cache`设置为null，而是遍历调用`pruneCacheEntry`函数删除。

`pruneCacheEntry`函数定义在

> ​	src/core/components/keep-alive.js

```typescript
function pruneCacheEntry (
  cache: VNodeCache,
  key: string,
  keys: Array<string>,
  current?: VNode
) {
 const cached = cache[key]
 if (cached && (!current || cached.tag !== current.tag)) {
    cached.componentInstance.$destroyed() // 执行组件的destroy钩子函数
 }
 cache[key] = null
 remove(keys, key)
}
```

删除缓存的VNode还要对应组件实例的`destory`钩子函数

- mounted

  在`mounted`这个钩子函数中对`include`和`exclude`两个参数进行监听，然后实时的更新（删除）`this.cache`对象数据。`pruneCache`函数的核心也是去调用`pruneCacheEntry`

  ```typescript
  function pruneCache (keepAliveInstance: any, filter: Function) {
    const { cache, keys, _vnode } = keepAliveInstance
    for (const key in cache) {
      const cachedNode: ?VNode = cache[key]
      if (cachedNode) {
        const name: ?string = getComponentName(cachedNode.componentOptions)
        if (name && !filter(name)) {
          pruneCacheEntry(cache, key, keys, _vnode)
        }
      }
    }
  }
  ```

- render

  ```typescript
  render () {
      const slot = this.$slots.default
      const vnode: VNode = getFirstComponentChild(slot) // 找到第一个组件对象
      const componentOptions: ?VNodeComponentOptions = vnode && vnode.componentOptions
      if (componentOptions) { // 存在组价参数
        // check pattern
        const name: ?string = getComponentName(componentOptions) // 组件名
        const { include, exclude } = this
        if ( // 条件匹配
          // not included
          (include && (!name || !matches(include, name))) ||
          // excluded
          (exclude && name && matches(exclude, name))
        ) {
          return vnode
        }
  
        const { cache, keys } = this
        // 定义组件的缓存
        const key: ?string = vnode.key == null
          // same constructor may get registered as different local components
          // so cid alone is not enough (#3269)
          ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
          : vnode.key
        if (cache[key]) { // 已经缓存过该组件
          vnode.componentInstance = cache[key].componentInstance
          // make current key freshest
          remove(keys, key)
          keys.push(key) // 调整key排序
        } else {
          cache[key] = vnode // 缓存组件对象
          keys.push(key)
          // prune oldest entry
          if (this.max && keys.length > parseInt(this.max)) {
            // 超过缓存数限制，将一个方案删除  
            pruneCacheEntry(cache, keys[0], keys, this._vnode)
          }
        }
  
        vnode.data.keepAlive = true // 渲染和执行被包裹组件的钩子函数需要用到
      }
      return vnode || (slot && slot[0])
    }
  ```

  首先获取第一个子元素的`vnode`:

  ```typescript
  const slot = this.$slots.default
  const vnode: VNode = getFirstComponentChild(slot)
  ```

  由于我们也是在`keep-alive`标签内部写DOM，所以可以先获取到它的默认插槽，然后再获取到它的第一个子节点。`<keep-alive>`只处理第一个子元素，所以一般和它搭配使用的有`component`动态组件或者是`router-view`。



​	接下里有判断了当当前组件的名称和`include`和`exclude`的关系：

```typescript
// check pattern
const name: ?string = getComponentName(componentOptions)
const { include, exclude } = this
if (
  // not included
  (include && (!name || !matches(include, name))) ||
  // excluded
  (exclude && name && matches(exclude, name))
) {
  return vnode
}

function matches (pattern: string | RegExp | Array<string>, name: string): boolean {
  if (Array.isArray(pattern)) {
    return pattern.indexOf(name) > -1
  } else if (typeof pattern === 'string') {
    return pattern.split(',').indexOf(name) > -1
  } else if (isRegExp(pattern)) {
    return pattern.test(name)
  }
  return false
}
```

`matches`的逻辑很简单，就是做匹配，分别处理了数组、字符串和正则表达式的情况，也就是说我们平时传的`include`和`exclude`可以是这三种类型的任意一种。并且我们的组件名如果满足了配置的`include`且不匹配或者是配置了`exclude`且匹配，那么久返回这个组件的`vnode`，否则的话就走下一步缓存：

```typescript
  const { cache, keys } = this
  // 定义组件的缓存
  const key: ?string = vnode.key == null
    // same constructor may get registered as different local components
    // so cid alone is not enough (#3269)
    ? componentOptions.Ctor.cid + (componentOptions.tag ? `::${componentOptions.tag}` : '')
    : vnode.key
  if (cache[key]) { // 已经缓存过该组件
    vnode.componentInstance = cache[key].componentInstance
    // make current key freshest
    remove(keys, key)
    keys.push(key) // 调整key排序
  } else {
    cache[key] = vnode // 缓存组件对象
    keys.push(key)
    // prune oldest entry
    if (this.max && keys.length > parseInt(this.max)) {
      // 超过缓存数限制，将一个方案删除  
      pruneCacheEntry(cache, keys[0], keys, this._vnode)
    }
  }
```

这部分逻辑挺简单的，如果命中缓存，则直接从缓存中拿`vnode`的组件实例，并且重新调整了`key`的顺序放在了最后一个；否则把`vnode`设置进缓存，最后还有一个逻辑，如果配置了`max`并缓存的最大长度超过了`this.max`，还要从缓存中删除第一个：

```typescript
function pruneCacheEntry (
  cache: VNodeCache,
  key: string,
  keys: Array<string>,
  current?: VNode
) {
  const cached = cache[key]
  if (cached && (!current || cached.tag !== current.tag)) {
    cached.componentInstance.$destroy()
  }
  cache[key] = null 
  remove(keys, key)
}
```

除了从缓存中删除外，还要判断如果要删除的缓存的组件`tag`不是当前渲染组件`tag`，也执行删除缓存的组件实例的`$destroy`方法。

最后设置`vnode.data.keepAlive = true`，这个后面会说明，

总结`<keep-alive>`具体的就是分为以下5步

- 第一步：获取`keep-alive`包裹着的第一个子组件对象及其组件名；
- 第二步：根据设定的黑白名单（如果有）进行条件匹配，决定是否进行缓存。如果不匹配，直接返回组件实例（VNode），否则执行第三步；
- 第三步：根据组件ID和`tag`生成缓存`key`，并在缓存对象中查找是否已缓存过该组件实例。如果存在，直接取出缓存值并更新该`key`在`this.keys`中的位置（更新`key`的位置是实现`LRU`置换策略的关键），否则执行第四步；
- 在`this.cache`对象中存储该组件实例并保存`key`值，之后检查缓存的实例数量是否超过`max`设置值，超过则根据`LRU`置换策略删除最近最久未使用过的实例（即下标为0的那个key）；
- 第五步：最后并且很重要，将该组件实例的`keepAlive`属性值设置为true



#### 组件渲染

上面了解了`<keep-alive>`的组件的实现，但并不知道它 包裹的子组件渲染和普通组件有什么不一样的地方，这里我们关注两个方面，首次渲染和缓存渲染。

举一个例子：

```js
let A = {
  template: '<div class="a">' +
  '<p>A Comp</p>' +
  '</div>',
  name: 'A'
}

let B = {
  template: '<div class="b">' +
  '<p>B Comp</p>' +
  '</div>',
  name: 'B'
}

let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<keep-alive>' +
  '<component :is="currentComp">' +
  '</component>' +
  '</keep-alive>' +
  '<button @click="change">switch</button>' +
  '</div>',
  data: {
    currentComp: 'A'
  },
  methods: {
    change() {
      this.currentComp = this.currentComp === 'A' ? 'B' : 'A'
    }
  },
  components: {
    A,
    B
  }
})
```



#### 首次渲染

我们知道Vue的渲染最后都会到`patch`过程，而组件的`patch`过程会执行`createComponent`方法，它定义在

> src/core/vdom/patch.js

```typescript
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
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

`createComponent`定义了`isReactiveted`的变量，它是根据`vnode.componentInstance`以及`vnode.data.keepAlive`的判断，第一次渲染的时候，`vnode.componentInstance`为`undefined`，`vnode.data.keepAlive`为`true`，因为它的父组件`<keep-alive>`的`render`函数会先执行，那么该`vnode`缓存到内存中，并且设置`vnode.data.keepAlive`为`true`，因此`isReactiveted`为`false`，那么走正常的`init`的钩子函数执行组件的`mount`，当`vnode`已经执行完`patch`后，执行`initComponent`函数。

```typescript
function initComponent (vnode, insertedVnodeQueue) {
  if (isDef(vnode.data.pendingInsert)) {
    insertedVnodeQueue.push.apply(insertedVnodeQueue, vnode.data.pendingInsert)
    vnode.data.pendingInsert = null
  }
  vnode.elm = vnode.componentInstance.$el
  if (isPatchable(vnode)) {
    invokeCreateHooks(vnode, insertedVnodeQueue)
    setScope(vnode)
  } else {
    // empty component root.
    // skip all element-related modules except for ref (#3455)
    registerRef(vnode)
    // make sure to invoke the insert hook
    insertedVnodeQueue.push(vnode)
  }
}
```

这里会有`vnode.elm`缓存`vnode`创建生成的DOM节点。所以对于首次渲染而言，除了在`<keep-alive>`中建立缓存，和普通组件渲染没什么区别。

所以对于我们当前这个例子，初始化渲染`A`组件以及第一次点击`switch`渲染组件`B`组件，都是首次渲染。



#### 缓存渲染

当我们从`B`组件再次点击`switch`切换到`A`组件，就会命中缓存渲染，它会对于新旧`vnode`节点，甚至对白他们的子节点去做更新逻辑，但是对于组件`vnode`而言，是没有`children`的，那么对于`<keep-alive>`组件而言，如果更新它包裹的内容呢？

原来`pathcVnode`在做各种`diff`之前，会先执行`prepatch`的钩子函数，它的定义在

> ​	src/core/vdom/create-component

```typescript
const componentVNodeHooks = {
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
  // ...
}
```

`prepatch`核心逻辑就是执行`updateChildComponent`方法，它的定义在

> src/core/instance/lifecycle.js

```typescript
export function updateChildComponent (
  vm: Component,
  propsData: ?Object,
  listeners: ?Object,
  parentVnode: MountedComponentVNode,
  renderChildren: ?Array<VNode>
) {
  const hasChildren = !!(
    renderChildren ||          
    vm.$options._renderChildren ||
    parentVnode.data.scopedSlots || 
    vm.$scopedSlots !== emptyObject 
  )

  // ...
  if (hasChildren) {
    vm.$slots = resolveSlots(renderChildren, parentVnode.context)
    vm.$forceUpdate()
  }
}
```

`updateChildComponent`方法主要失去更新组件实例的一些属性，这里我们重点关注`slot`部分，由于`<keep-alive>`组件本质上支持了`slot`，所以它执行`prepatch`的时候，需要对自己的`children`，也就是这些`slots`做重新解析，并触发`<keep-alive>`组件实例`$forceUpdate()`逻辑，也就是重新执行`<keep-alive>`的`render`方法，这个时候如果它包裹的第一个组件`vnode`命中缓存，则直接返回缓存中的`vnode.componentInstance`，在我们当前这个例子中就是缓存的`A`组件，接着又会执行`patch`过程，再次执行到`createComponent`方法，这个时候我们再来看看`createComponent`方法

```typescript
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
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

这个时候的`isReacttived`为true，并且在执行`init`钩子函数的时候不会再执行组件的`mount`过程，相关的逻辑在

> `src/core/vdom/create-component.js

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
  // ...
}
```

这也就是被`<keep-alive>`包裹的组件在有缓存的时候就不会再执行组件的`created`、`mounted`等钩子函数的原因了，回到`createComponent`方法，在`isReactivated`为`true`的情况下会执行`reactivateComponent`方法：

```typescript
function reactivateComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  let i
  // hack for #4339: a reactivated component with inner transition
  // does not trigger because the inner node's created hooks are not called
  // again. It's not ideal to involve module-specific logic in here but
  // there doesn't seem to be a better way to do it.
  let innerNode = vnode
  while (innerNode.componentInstance) {
    innerNode = innerNode.componentInstance._vnode
    if (isDef(i = innerNode.data) && isDef(i = i.transition)) {
      for (i = 0; i < cbs.activate.length; ++i) {
        cbs.activate[i](emptyNode, innerNode)
      }
      insertedVnodeQueue.push(innerNode)
      break
    }
  }
  // unlike a newly created component,
  // a reactivated keep-alive component doesn't insert itself
  insert(parentElm, vnode.elm, refElm)
}
```

前面部分的逻辑是解决对`reactived`组件`transition`动画不出发的问题，这里可以暂时可以忽略，最后通过执行`insert(parentElm, vnode.elm, refElm)`就把缓存的DOM对象直接插入到目标元素中，这样就完成了在数据更新的情况下的渲染过程。