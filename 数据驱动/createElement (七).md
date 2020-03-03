`VNode`通过`createElement`方法创建
> 文件位置: `src/core/vdom/create-element..js`中，

```typescript
// wrapper function for providing a more flexible interface
// without getting yelled at by flow
export function createElement (
  context: Component, // VN实例
  tag: any, // VNode的标签
  data: any, // VNode的data
  children: any, // VNode的子节点也可以说是子Vnode
  normalizationType: any,
  alwaysNormalize: boolean
): VNode | Array<VNode> {
  // 检测传入的第三个参数，是否第children类型，也就是data参数传
  // 将data后面三个参数位子依次上移一位
  if (Array.isArray(data) || isPrimitive(data)) {
    normalizationType = children
    children = data
    data = undefined
  }
  // 判断alwaysNormalize
  if (isTrue(alwaysNormalize)) {
    normalizationType = ALWAYS_NORMALIZE // ALWAYS_NORMALIZE = 2  SIMPLE_NORALIZE = 1
  }
  return _createElement(context, tag, data, children, normalizationType)
}
```
这里的`createElement`方法实际上是对`_createElement`方法的封装，允许传入的参数更加灵活处理，在处理这些参数后，调用真正创建`VNode`的函数`_createElement`：

```typeScript
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
`_createElement`方法有5个参数，`context`表示VNode的上下文环境也就是`VN`，它是`component`类型；`tag`表示标签，可以是字符串也可以是一个`Component`；`data`表示VNode的数据，是一个`VNodeData`类型，可以在`flow/vnode.js`中找到定义;`children`表示点钱VNode的子节点，它是任意类型的，它接下来需要被规范为标准的VNode数组；`normalizationType`表示子节点的规范类型，类型不同规范的，规范方法也就不同，主要是参考`render`函数是用户手写的，还是是编译生成的。

### children的规范化

由于Virtual DOM实际上是一个树状结构，每一个VNode可能会有若干个子节点，这个子节点应该也是VNode的类型，`_createElement`接受的第4个参数children是任意类型的，因此需要规范成VNode类型。

这里会根据`normalizationType`的不同，调用不同的规范方法，`ormalizeChildren(children)`和`simpleNormalizeChildren(children)`
>文件位置：`src/core/vdom/helpers/mormalzie-children.js`

```typeScript
// The template compiler attempts to minimize the need for normalization by
// statically analyzing the template at compile time.
//
// For plain HTML markup, normalization can be completely skipped because the
// generated render function is guaranteed to return Array<VNode>. There are
// two cases where extra normalization is needed:

// 1. When the children contains components - because a functional component
// may return an Array instead of a single root. In this case, just a simple
// normalization is needed - if any child is an Array, we flatten the whole
// thing with Array.prototype.concat. It is guaranteed to be only 1-level deep
// because functional components already normalize their own children.
export function simpleNormalizeChildren (children: any) {
  for (let i = 0; i < children.length; i++) {
    if (Array.isArray(children[i])) {
      return Array.prototype.concat.apply([], children)
    }
  }
  return children
}

// 2. When the children contains constructs that always generated nested Arrays,
// e.g. <template>, <slot>, v-for, or when the children is provided by user
// with hand-written render functions / JSX. In such cases a full normalization
// is needed to cater to all possible types of children values.
export function normalizeChildren (children: any): ?Array<VNode> {
  return isPrimitive(children)
    ? [createTextVNode(children)]
    : Array.isArray(children)
      ? normalizeArrayChildren(children)
      : undefined
}
```
`simpleNormalizeChildren`方法的调用场景是`render`函数当函数编译生成的。理论上编译生成的`children`都已经是VNode类型的，但是这里有一个例外，就是`functional component`函数式组件返回的是一个数组而不是一个根节点，所以会通过`Array.prototype.concat`方法把整个数组给扁平化，让深度只有一层

`normalizeChildren`方法调用场景有两个
- `render`函数是用户手写的
-- 当`children`只要䘝节点的时候，vue.js从接口层面允许用户把`children`写成基础类型来创建单个简单的文本节点这种情况会调用`createTextVNode`创建一个文本节点的VNode

- 当编译`solt、v-for`的时候会产生嵌套数组的情况，会调用`normalizeArratChildren`方法

实现
```TypeScript
function normalizeArrayChildren (children: any, nestedIndex?: string): Array<VNode> {
  const res = []
  let i, c, lastIndex, last
  for (i = 0; i < children.length; i++) {
    c = children[i]
    if (isUndef(c) || typeof c === 'boolean') continue
    lastIndex = res.length - 1
    last = res[lastIndex]
    //  nested
    // 如果子节点是Array
    // 递归处理children
    if (Array.isArray(c)) {
      if (c.length > 0) {
        c = normalizeArrayChildren(c, `${nestedIndex || ''}_${i}`)
        // merge adjacent text nodes
        // 如果当前节点和下一次节点都是文本节点，把这两个节点合并到一个节点里面
        if (isTextNode(c[0]) && isTextNode(last)) {
          res[lastIndex] = createTextVNode(last.text + (c[0]: any).text)
          c.shift()
        }
        res.push.apply(res, c)
      }
    } else if (isPrimitive(c)) { // 如果是基础节点
      if (isTextNode(last)) { // 如果是文本节点
        // merge adjacent text nodes
        // this is necessary for SSR hydration because text nodes are
        // essentially merged when rendered to HTML strings
        res[lastIndex] = createTextVNode(last.text + c)
      } else if (c !== '') {
        // convert primitive to vnode
        res.push(createTextVNode(c))
      }
    } else {
    // 正常VNode
      if (isTextNode(c) && isTextNode(last)) {
        // merge adjacent text nodes
        res[lastIndex] = createTextVNode(last.text + c.text)
      } else {
        // default key for nested array children (likely generated by v-for)
        if (isTrue(children._isVList) &&
          isDef(c.tag) &&
          isUndef(c.key) &&
          isDef(nestedIndex)) {
          c.key = `__vlist${nestedIndex}_${i}__`
        }
        res.push(c)
      }
    }
  }
  return res
}
```
`normalizeArrayChildren`接收两个参数，`children`表示要规范的子节点，`nestedIndex`表示嵌套的索引，因为单个`child`可能是一个数组类型。`normalizeArrayChildren`主要的逻辑就是遍历`children`，获得单个节点`c`，然后对`c`的类型判断，如果是数组，则递归调用`normalizeArrayChildren`；如果是基础类型，则通过`createTextVnode`方法转化成VNode类型；否则就已经是VNode类型了，如果`children`是一个列表并且列表还存在着嵌套的情况，则根据`nestedIndex`去更新它的key。这里需要主要的是，在遍历的过程中对这3中情况都了如下处理：如果存在两个连续的`text`节点，会把他们合并成一个`text`节点

经过对`children`的规范化，`children`变成了一个类型为VNode的Array

### VNode的创建
经过上面过程后,接下来会去创建一个VNode的实例

```TypeScript
let vnode, ns
if (typeof tag === 'string') {
  let Ctor
  ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
  if (config.isReservedTag(tag)) {
    // platform built-in elements
    vnode = new VNode(
      config.parsePlatformTagName(tag), data, children,
      undefined, undefined, context
    )
  } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
    // component
    vnode = createComponent(Ctor, data, context, children, tag)
  } else {
    // unknown or unlisted namespaced elements
    // check at runtime because it may get assigned a namespace when its
    // parent normalizes children
    vnode = new VNode(
      tag, data, children,
      undefined, undefined, context
    )
  }
} else {
  // direct component options / constructor
  vnode = createComponent(tag, data, context, children)
}
```

这里先对`tag`做判断，如果是`string`类型，则接着判断如果是内置的一些节点，则直接创建一个普通的VNode，如果是为已注册的组件名，则通过`createComponent`创建一个组件类型的VNode，否则创建一个未知标签的VNode，如果`tag`一个`Component`类型，则直接调用`createComponent`创建组件类型的VNode节点，本质上还是返回一个VNode

回到`mountComponent`函数的过程，我们已经知道`vm._render`是如何创建了一个 VNode，接下来就是要把这个 VNode 渲染成一个真实的 DOM 并渲染出来，这个过程是通`vm._update`完成的
