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