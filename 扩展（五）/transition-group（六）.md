前面一节我们分析了`<transition>`组件的实现原理，它只能针对单一元素实现过渡效果，我们有时会遇到列表的需求，我们队列表元素进行添加和删除，有时候也希望有过渡效果，`Vue.js`提供了`<transition-group>`组件，很好的实现了列表的过渡效果，接下来分析也个实现的过程

举一个粒子

```js
let vm = new Vue({
  el: '#app',
  template: '<div id="list-complete-demo" class="demo">' +
  '<button v-on:click="add">Add</button>' +
  '<button v-on:click="remove">Remove</button>' +
  '<transition-group name="list-complete" tag="p">' +
  '<span v-for="item in items" v-bind:key="item" class="list-complete-item">' +
  '{{ item }}' +
  '</span>' +
  '</transition-group>' +
  '</div>',
  data: {
    items: [1, 2, 3, 4, 5, 6, 7, 8, 9],
    nextNum: 10
  },
  methods: {
    randomIndex: function () {
      return Math.floor(Math.random() * this.items.length)
    },
    add: function () {
      this.items.splice(this.randomIndex(), 0, this.nextNum++)
    },
    remove: function () {
      this.items.splice(this.randomIndex(), 1)
    }
  }
})
```

```css
 .list-complete-item {
  display: inline-block;
  margin-right: 10px;
}
.list-complete-move {
  transition: all 1s;
}
.list-complete-enter, .list-complete-leave-to {
  opacity: 0;
  transform: translateY(30px);
}
.list-complete-enter-active {
  transition: all 1s;
}
.list-complete-leave-active {
  transition: all 1s;
  position: absolute;
}
```

这个数字初始会展示`1-9`十个数字，当我们点击`add`按钮的时候，会生成`nextNum`并随机在当前数列表中插入；当我们点击`remove`按钮时，会随机删除掉一个数。在这个过程中，无论是添加还是删除都会有过渡动画，这就是`<transition-group>`组件配合我们定义的css产生的效果。



接下来我们来分析`<transition-group>`组件的实现，它的定义在:

> src/platforms/web/runtime/components/transition.js

```typescript
const props = extend({
  tag: String,
  moveClass: String
}, transitionProps)

delete props.mode

export default {
  props,

  beforeMount () {
    const update = this._update
    this._update = (vnode, hydrating) => {
      // force removing pass
      this.__patch__(
        this._vnode,
        this.kept,
        false, // hydrating
        true // removeOnly (!important, avoids unnecessary moves)
      )
      this._vnode = this.kept
      update.call(this, vnode, hydrating)
    }
  },

  render (h: Function) {
    const tag: string = this.tag || this.$vnode.data.tag || 'span'
    const map: Object = Object.create(null)
    const prevChildren: Array<VNode> = this.prevChildren = this.children
    const rawChildren: Array<VNode> = this.$slots.default || []
    const children: Array<VNode> = this.children = []
    const transitionData: Object = extractTransitionData(this)

    for (let i = 0; i < rawChildren.length; i++) {
      const c: VNode = rawChildren[i]
      if (c.tag) {
        if (c.key != null && String(c.key).indexOf('__vlist') !== 0) {
          children.push(c)
          map[c.key] = c
          ;(c.data || (c.data = {})).transition = transitionData
        } else if (process.env.NODE_ENV !== 'production') {
          const opts: ?VNodeComponentOptions = c.componentOptions
          const name: string = opts ? (opts.Ctor.options.name || opts.tag || '') : c.tag
          warn(`<transition-group> children must be keyed: <${name}>`)
        }
      }
    }

    if (prevChildren) {
      const kept: Array<VNode> = []
      const removed: Array<VNode> = []
      for (let i = 0; i < prevChildren.length; i++) {
        const c: VNode = prevChildren[i]
        c.data.transition = transitionData
        c.data.pos = c.elm.getBoundingClientRect()
        if (map[c.key]) {
          kept.push(c)
        } else {
          removed.push(c)
        }
      }
      this.kept = h(tag, null, kept)
      this.removed = removed
    }

    return h(tag, null, children)
  },

  updated () {
    const children: Array<VNode> = this.prevChildren
    const moveClass: string = this.moveClass || ((this.name || 'v') + '-move')
    if (!children.length || !this.hasMove(children[0].elm, moveClass)) {
      return
    }

    // we divide the work into three loops to avoid mixing DOM reads and writes
    // in each iteration - which helps prevent layout thrashing.
    children.forEach(callPendingCbs)
    children.forEach(recordPosition)
    children.forEach(applyTranslation)

    // force reflow to put everything in position
    // assign to this to avoid being removed in tree-shaking
    // $flow-disable-line
    this._reflow = document.body.offsetHeight

    children.forEach((c: VNode) => {
      if (c.data.moved) {
        var el: any = c.elm
        var s: any = el.style
        addTransitionClass(el, moveClass)
        s.transform = s.WebkitTransform = s.transitionDuration = ''
        el.addEventListener(transitionEndEvent, el._moveCb = function cb (e) {
          if (!e || /transform$/.test(e.propertyName)) {
            el.removeEventListener(transitionEndEvent, cb)
            el._moveCb = null
            removeTransitionClass(el, moveClass)
          }
        })
      }
    })
  },

  methods: {
    hasMove (el: any, moveClass: string): boolean {
      /* istanbul ignore if */
      if (!hasTransition) {
        return false
      }
      /* istanbul ignore if */
      if (this._hasMove) {
        return this._hasMove
      }
      // Detect whether an element with the move class applied has
      // CSS transitions. Since the element may be inside an entering
      // transition at this very moment, we make a clone of it and remove
      // all other transition classes applied to ensure only the move class
      // is applied.
      const clone: HTMLElement = el.cloneNode()
      if (el._transitionClasses) {
        el._transitionClasses.forEach((cls: string) => { removeClass(clone, cls) })
      }
      addClass(clone, moveClass)
      clone.style.display = 'none'
      this.$el.appendChild(clone)
      const info: Object = getTransitionInfo(clone)
      this.$el.removeChild(clone)
      return (this._hasMove = info.hasTransform)
    }
  }
}
```

接下来分析代码：

#### render函数

`<transition-group>`组件，`<transition-group>`组件非抽象组件，它会渲染成一个真实元素，默认`tag`是`span`。`prevChildren`用来存储上一次的子节点，；`children`用来存储当前的子节点；`rawChildren`表示`<transition-group>`包裹的原始子节点；`transitionData`是从`<transition-group>`组件上提取出来的一些渲染数据，这点和`<transition>`组件的实现是一样的。

- 遍历`rawChildren`和`children`

```typescript
for (let i = 0; i < rawChildren.length; i++) {
  const c: VNode = rawChildren[i]
  if (c.tag) {
    if (c.key != null && String(c.key).indexOf('__vlist') !== 0) {
      children.push(c)
      map[c.key] = c
      ;(c.data || (c.data = {})).transition = transitionData
    } else if (process.env.NODE_ENV !== 'production') {
      const opts: ?VNodeComponentOptions = c.componentOptions
      const name: string = opts ? (opts.Ctor.options.name || opts.tag || '') : c.tag
      warn(`<transition-group> children must be keyed: <${name}>`)
    }
  }
}
```

其实就是对`rawChildren`的遍历，拿到每个`vnode`，然后判断每个`vnode`是否设置了`key`，这个是`<transition-group>`对列表元素的要求。然后把`vnode`添加到`children`中，然后把刚刚提取出来的过渡数据`transitionData`添加到`vnode.data.transition`中，这点是很关键的，只有这样才能实现列表中单个元素的过渡动画。

- 处理`prevChildren`

```typescript
if (prevChildren) {
  const kept: Array<VNode> = []
  const removed: Array<VNode> = []
  for (let i = 0; i < prevChildren.length; i++) {
    const c: VNode = prevChildren[i]
    c.data.transition = transitionData
    c.data.pos = c.elm.getBoundingClientRect()
    if (map[c.key]) {
      kept.push(c)
    } else {
      removed.push(c)
    }
  }
  this.kept = h(tag, null, kept)
  this.removed = removed
}

return h(tag, null, children)
```

当有`prevchildren`的时候，我们会对它做遍历，获取到每个`vnode`，然后把`transitionData`赋值到`vnode.data.transition`，这个是为了当它在`enter`和`leave`的钩子函数中有过渡动画，接着又调用了原生的DOM的`getBoundingClientRect`方法获取到原生DOM的位置信息，记录到`vnode.data.pos`中，然后判断一下`vnode.key`是否在`map`中，如果存在，则存放到`kept`中，否则表示该节点已经被删除，放入`removed`中，然后通过执行`h(tag, null, kept)`渲染后放入`this.kept`中，把`removed`用`this.removed`保存。整个`render`函数通过`h(tag, null, children)`生成渲染`vnode`。

如果`transition-group`只实现了这个`render`函数，那么每次插入和删除的元素的缓动画是可以实现的，但是在我们当前这个例子中，当新增一个元素，它的插入的过渡动画是有的，但是剩余元素平移的过渡效果是出不来的，所以接下来我们来分析`<transition-group>`组件是如何实现剩余元素平移的过渡效果的。

