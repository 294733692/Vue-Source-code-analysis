Virtual DOM（虚拟Dom）产生的原因是因为浏览器中的DOM是很‘昂贵’的，我们可以创建一个div，并把div打印出来

```js
let div = document.createElement('div')
let str = ''
for (let key in div) {
    str += key + ' '
}
```
可以尝试一下这个代码，在浏览器中打印出来。

之所以这么庞大，是因为浏览器标准就把DOM设计的非常的复杂，当我们频繁的去做DOM更新，会产生一定的性能问题。

创建Virtual DOM就是用一个原生的JS对象去描述一个DOM节点，这比创建一个DOM的代价要小很多，而`vue`中，Virtual DOM是用`VNode`这个Class去描述的。

> 文件位置：`src/core/vdom/vnode.js`中


```typeScript
export default class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```
这里可以看到这里定义了很多的参数，因为这里包含了许多`Vue.js`的特性，这里定了几个关键的属性，比如标签名，数据，子节点，键值等，其他的属性都是用来扩展`VNode`的灵活性以及实现一些特殊的`feature`，由于`VNode`只是用来映射到真实DOM的渲染，不需要包含操作DOM的方法，所有说是非常轻量和简单的。

`VNode`映射到真实DOM实际上要经历`VNode`的create、diff、patch等过程。而`VNode`的create是通过之前提到过的`createElement`方法创建的。
