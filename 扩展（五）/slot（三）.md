### slot 插槽

Vue的插槽分为两种插槽，分别是

- 普通插槽
- 作用域插槽



#### 普通插槽

举一个例子：

```js
let AppLayout = {
  template: '<div class="container">' +
  '<header><slot name="header"></slot></header>' +
  '<main><slot>默认内容</slot></main>' +
  '<footer><slot name="footer"></slot></footer>' +
  '</div>'
}

let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<app-layout>' +
  '<h1 slot="header">{{title}}</h1>' +
  '<p>{{msg}}</p>' +
  '<p slot="footer">{{desc}}</p>' +
  '</app-layout>' +
  '</div>',
  data() {
    return {
      title: '我是标题',
      msg: '我是内容',
      desc: '其它信息'
    }
  },
  components: {
    AppLayout
  }
})
```

这里定义了一个名为`AppLayout`的子组件，组件的内部定义了3个插槽，两个为具名插槽，一个`name`为`header`，另外一个`name`为`footer`，这里还定义了一个没有`name`的默认插槽。`<slot`和`</slot`之间的填写的内容为默认内容。我们组件注册引用了`AppLayout`组件，并在里面定义了一些元素，来替换插槽，那么最终生成的DOM如下:

```html
<div>
    <div class="container">
        <header><h1>我是标题</h1></header>
        <main><p>我是内容</p></main>
        <footer><p>其他内容</p></footer>
    </div>	
</div>
```

接下来我们来看看编译的过程：



##### 编译

从编译开始，编译是发生在调用`vm.$mount`的时候，所以编译的顺序是先编译父组件，在编译子组件。

首先开始编译父组件，在`page`阶段，会执行`processSlot`处理`slot`，该方法定义在

> src/compiler/parser/index.js

```js
function processSlot (el) {
  if (el.tag === 'slot') {
    el.slotName = getBindingAttr(el, 'name')
    if (process.env.NODE_ENV !== 'production' && el.key) {
      warn(
        `\`key\` does not work on <slot> because slots are abstract outlets ` +
        `and can possibly expand into multiple elements. ` +
        `Use the key on a wrapping element instead.`
      )
    }
  } else {
    let slotScope
    if (el.tag === 'template') {
      slotScope = getAndRemoveAttr(el, 'scope')
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && slotScope) {
        warn(
          `the "scope" attribute for scoped slots have been deprecated and ` +
          `replaced by "slot-scope" since 2.5. The new "slot-scope" attribute ` +
          `can also be used on plain elements in addition to <template> to ` +
          `denote scoped slots.`,
          true
        )
      }
      el.slotScope = slotScope || getAndRemoveAttr(el, 'slot-scope')
    } else if ((slotScope = getAndRemoveAttr(el, 'slot-scope'))) {
      /* istanbul ignore if */
      if (process.env.NODE_ENV !== 'production' && el.attrsMap['v-for']) {
        warn(
          `Ambiguous combined usage of slot-scope and v-for on <${el.tag}> ` +
          `(v-for takes higher priority). Use a wrapper <template> for the ` +
          `scoped slot to make it clearer.`,
          true
        )
      }
      el.slotScope = slotScope
    }
    const slotTarget = getBindingAttr(el, 'slot')
    if (slotTarget) {
      el.slotTarget = slotTarget === '""' ? '"default"' : slotTarget
      // preserve slot as an attribute for native shadow DOM compat
      // only for non-scoped slots.
      if (el.tag !== 'template' && !el.slotScope) {
        addAttr(el, 'slot', slotTarget)
      }
    }
  }
}
```

当解析到标签上有`slot`属性的时候，会给对应的AST元素节点添加`slotTarget`属性，然后在`codegen`阶段，在`genData`中会处理`slotTarget`，相关代码在

> src/compiler/codegen/index.js

```js
if (el.slotTarget && !el.slotScope) {
    data += `slot:${el.slotTarget},`
}
```

从这里看到，会给`data`添加一个`slot`属性，并指向`slotTarget`，之后会用到，在当前这个例子中，父组件最终生成的代码如下：

```js
with(this){
  return _c('div',
    [_c('app-layout',
      [_c('h1',{attrs:{"slot":"header"},slot:"header"},
         [_v(_s(title))]),
       _c('p',[_v(_s(msg))]),
       _c('p',{attrs:{"slot":"footer"},slot:"footer"},
         [_v(_s(desc))]
         )
       ])
     ],
   1)
}
```

接下来是编译子组件，同样在`parser`阶段会执行`processSlot`处理函数，它的定义在

> src/compiler/parser/index.js

```js
function processSlot (el) {
  if (el.tag === 'slot') {
    el.slotName = getBindingAttr(el, 'name')
  }
  // ...
}
```

当遇到`slot`标签的时候会给对应的AST元素节点添加`slotName`属性，然后在`codegen`阶段，会判断如果当前AST元素节点是`slot`标签，则会执行`genSlot`函数，它的定义在

> src/complier/codegen/index.js

```typescript
function genSlot (el: ASTElement, state: CodegenState): string {
  const slotName = el.slotName || '"default"'
  const children = genChildren(el, state)
  let res = `_t(${slotName}${children ? `,${children}` : ''}`
  const attrs = el.attrs && `{${el.attrs.map(a => `${camelize(a.name)}:${a.value}`).join(',')}}`
  const bind = el.attrsMap['v-bind']
  if ((attrs || bind) && !children) {
    res += `,null`
  }
  if (attrs) {
    res += `,${attrs}`
  }
  if (bind) {
    res += `${attrs ? '' : ',null'},${bind}`
  }
  return res + ')'
}
```

这里暂时不考虑`slot`标签上有`attrs`以及`v-bind`的情况，那么它生成的代码实际上就只有：

```js
const slotName = el.slotName || 'default'
const children = genChildren(el, state)
let res = `_t(${slotName}${children ? `,${children}` : ''}`
```

这里的`slotName`从AST元素节点对应的属性上取，默认是`default`，而`children`对应的就是`slot`开始和闭合标签包裹的内容，当前我们子组件生成的最终代码如下：

```js
with(this) {
  return _c('div',{
    staticClass:"container"
    },[
      _c('header',[_t("header")],2),
      _c('main',[_t("default",[_v("默认内容")])],2),
      _c('footer',[_t("footer")],2)
      ]
   )
}
```

在编译章节我们了解到，'_t'函数对应的就是`renderSlot`方法，这个方法定义在

> src/core/instance/render-heplpers/render-slot.js

```typescript
/**
 * Runtime helper for rendering <slot>
 */
export function renderSlot (
  name: string,
  fallback: ?Array<VNode>,
  props: ?Object,
  bindObject: ?Object
): ?Array<VNode> {
  const scopedSlotFn = this.$scopedSlots[name]
  let nodes
  if (scopedSlotFn) { // scoped slot
    props = props || {}
    if (bindObject) {
      if (process.env.NODE_ENV !== 'production' && !isObject(bindObject)) {
        warn(
          'slot v-bind without argument expects an Object',
          this
        )
      }
      props = extend(extend({}, bindObject), props)
    }
    nodes = scopedSlotFn(props) || fallback
  } else {
    const slotNodes = this.$slots[name]
    // warn duplicate slot usage
    if (slotNodes) {
      if (process.env.NODE_ENV !== 'production' && slotNodes._rendered) {
        warn(
          `Duplicate presence of slot "${name}" found in the same render tree ` +
          `- this will likely cause render errors.`,
          this
        )
      }
      slotNodes._rendered = true
    }
    nodes = slotNodes || fallback
  }

  const target = props && props.slot
  if (target) {
    return this.$createElement('template', { slot: target }, nodes)
  } else {
    return nodes
  }
}
```

`render-slot`的参数`name`代表插槽名称`slotName`，`fallback`代表插槽的默认内容生成的`vnode`数组，现在暂时忽略`scope-slot`，只看默认插槽逻辑，如果`this.$slot[name]`有值，就返回它的对应的`vnode`数组，否则就返回`fallback`。那么这个`this.$slot`是从哪里来的呢？我们知道子组件的`init`时机是在父组件执行`patch`过程的时候，那么这个时候父组件已经编译完成了，并且子组件在`init`过程中会执行`initRender`函数，`initRender`的时候获取到`vm.$slot`，相关代码在

> src/core/instance/render.js

```typescript
export function initRender (vm: Component) {
  // ...
  const parentVnode = vm.$vnode = options._parentVnode // the placeholder node in parent tree
  const renderContext = parentVnode && parentVnode.context
  vm.$slots = resolveSlots(options._renderChildren, renderContext)
}
```

`vm.$slot`是通过执行`resolveSlots(optoins._renderChildren, renderContext)`返回的，它的定义在

> src/core/instance/render-helpers/resolve-slots.js

```typescript
/**
 * Runtime helper for resolving raw children VNodes into a slot object.
 */
export function resolveSlots (
  children: ?Array<VNode>,
  context: ?Component
): { [key: string]: Array<VNode> } {
  const slots = {}
  if (!children) {
    return slots
  }
  for (let i = 0, l = children.length; i < l; i++) {
    const child = children[i]
    const data = child.data
    // remove slot attribute if the node is resolved as a Vue slot node
    if (data && data.attrs && data.attrs.slot) {
      delete data.attrs.slot
    }
    // named slots should only be respected if the vnode was rendered in the
    // same context.
    if ((child.context === context || child.fnContext === context) &&
      data && data.slot != null
    ) {
      const name = data.slot
      const slot = (slots[name] || (slots[name] = []))
      if (child.tag === 'template') {
        slot.push.apply(slot, child.children || [])
      } else {
        slot.push(child)
      }
    } else {
      (slots.default || (slots.default = [])).push(child)
    }
  }
  // ignore slots that contains only whitespace
  for (const name in slots) {
    if (slots[name].every(isWhitespace)) {
      delete slots[name]
    }
  }
  return slots
}
```

`resolveSlots`方法接收两个参数

- 第一个参数`children`对应的是父`VNode`的`children`，在我们的例子中就是`<app-layout`和`<app-layout>`包裹的内容，
- 第二个参数`context`是父`VNode`的上下文，也就是父组件的`vm`实例

`resolveSlots`函数的逻辑就是遍历`children`，拿到每一个`child`的`data`，然后通过`data.slot`获取到插槽名称，这个`slot`就是我们之前编译父组件在`codegen`阶段设置的`data.slot`。接着以插槽名称为`key`把`child`添加到`slots`中，如果`data.slot`不存在，则是默认插槽内容，则把对应的`child`添加到`slots.defaults`中。这样就获取到整个`slots`，它是一个对象，`key`是插槽名称，`value`是一个`vnode`类型的数组，因为它可以有多个同名的插槽。

这样我们就拿到了`vm.$slots`了，回到`renderSlot`函数，`const slotNames = this.$slot[name]`，我们也就能根据插槽名称获取到对应 `vnode`数组了，这个数组的`vnode`都是在父组件创建的，这样就实现了父组件替换子组件插槽的内容。

在普通的插槽中，父组件应用到子组件插槽里的数据都是必须绑定到父组件的，因为它渲染成`vnode`的时机的上下文是父组件的实例。但是在一些实际的开发中，我们想通过子组件的一些数据来决定父组件实现的插槽的逻辑。

这里Vue提供了另外一种插槽——作用域插槽。接下来我们来看看实现的逻辑