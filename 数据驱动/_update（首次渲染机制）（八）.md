Vue的`_update`方法是实例的一个私有方法，它被调用的场景有两个
- 首次渲染的时候
- 数据更新的时候

`_update`方法的作用是把VNode渲染成真实DOM
> 文件位置: `src/core/instance/lifecycle.js`中

```TypeScript
Vue.prototype._update = function (vnode: VNode, hydrating?: boolean) {
 // 数据更新时候用的参数
  const vm: Component = this
  const prevEl = vm.$el
  const prevVnode = vm._vnode  // 首次渲染的时候为空
  const prevActiveInstance = activeInstance
  activeInstance = vm
  vm._vnode = vnode
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    // 首次渲染会走这个方法  VM.__patch__为核心
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
`_update`的核心就是调用`vm.__patch__`方法，这个方法在不同的平台，定义也是不一样的。如果说在`web`和`weex`上定义就是不一样的，在`web`平台的定义是
> 文件位置: `src/platforms/web/runtime/index.js`

```TypeScript
Vue.prototype.__patch__ = inBrowser ? patch : noop
```
这里可以看到，在`web`平台会返回`patch`方法，但是在服务端渲染的时候会返回`noop`，因为在服务端渲染的过程中，没有真实的浏览器DOM环境，所有不需要再VNode最终转换为DOM，所有返回一个空函数。而在浏览器端渲染的时候，返回了`patch`方法，此方法定义在
> 文件位置 `src/platforms/web/runtime/patch.js`

```TypeScript
import * as nodeOps from 'web/runtime/node-ops'
import { createPatchFunction } from 'core/vdom/patch'
import baseModules from 'core/vdom/modules/index'
import platformModules from 'web/runtime/modules/index'

// the directive module should be applied last, after all
// built-in modules have been applied.
const modules = platformModules.concat(baseModules)

// 调用了createPatchFunction的返回值，其中nodeOps封装了DOM操作的方法，module定义了一些模块的钩子函数的实现
export const patch: Function = createPatchFunction({ nodeOps, modules })
```
这个方法的是调用了`createPatchFunction`方法的返回值，这里传入了一个对象，包含`nodeOps`参数和`modules`参数，其中`nodeOps`封装了一系列操作DOM的方法，`modules`定义了一些模块的钩子函数。

`createPatchFunction`的实现
>文件位置 `src/core/vdom/patch.js`
```TypeScript
const hooks = ['create', 'activate', 'update', 'remove', 'destroy']

export function createPatchFunction (backend) {
  let i, j
  const cbs = {}

  const { modules, nodeOps } = backend

  // 遍历所有module的钩子函数，存放到cbs里面
  for (i = 0; i < hooks.length; ++i) {
    cbs[hooks[i]] = []
    for (j = 0; j < modules.length; ++j) {
      if (isDef(modules[j][hooks[i]])) {
        cbs[hooks[i]].push(modules[j][hooks[i]])
      }
    }
  }

  // ...

  return function patch (oldVnode, vnode, hydrating, removeOnly) {
    if (isUndef(vnode)) {
      if (isDef(oldVnode)) invokeDestroyHook(oldVnode)
      return
    }

    let isInitialPatch = false
    const insertedVnodeQueue = []

    if (isUndef(oldVnode)) {
      // empty mount (likely as component), create new root element
      isInitialPatch = true
      createElm(vnode, insertedVnodeQueue)
    } else {
      const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating)) {
            if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
              invokeInsertHook(vnode, insertedVnodeQueue, true)
              return oldVnode
            } else if (process.env.NODE_ENV !== 'production') {
              warn(
                'The client-side rendered virtual DOM tree is not matching ' +
                'server-rendered content. This is likely caused by incorrect ' +
                'HTML markup, for example nesting block-level elements inside ' +
                '<p>, or missing <tbody>. Bailing hydration and performing ' +
                'full client-side render.'
              )
            }
          }
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }

        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)

        // create new node
        createElm(
          vnode,
          insertedVnodeQueue,
          // extremely rare edge case: do not insert if old element is in a
          // leaving transition. Only happens when combining transition +
          // keep-alive + HOCs. (#4590)
          oldElm._leaveCb ? null : parentElm,
          nodeOps.nextSibling(oldElm)
        )

        // update parent placeholder node element, recursively
        if (isDef(vnode.parent)) {
          let ancestor = vnode.parent
          const patchable = isPatchable(vnode)
          while (ancestor) {
            for (let i = 0; i < cbs.destroy.length; ++i) {
              cbs.destroy[i](ancestor)
            }
            ancestor.elm = vnode.elm
            if (patchable) {
              for (let i = 0; i < cbs.create.length; ++i) {
                cbs.create[i](emptyNode, ancestor)
              }
              // #6513
              // invoke insert hooks that may have been merged by create hooks.
              // e.g. for directives that uses the "inserted" hook.
              const insert = ancestor.data.hook.insert
              if (insert.merged) {
                // start at index 1 to avoid re-invoking component mounted hook
                for (let i = 1; i < insert.fns.length; i++) {
                  insert.fns[i]()
                }
              }
            } else {
              registerRef(ancestor)
            }
            ancestor = ancestor.parent
          }
        }

        // destroy old node
        if (isDef(parentElm)) {
          removeVnodes(parentElm, [oldVnode], 0, 0)
        } else if (isDef(oldVnode.tag)) {
          invokeDestroyHook(oldVnode)
        }
      }
    }

    invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch)
    return vnode.elm
  }
}
```
`craetePatchFunction`内部定义了一系列的辅助方法，最终返回了一个`patch`方法，这个方法就赋值给了`vm._update`函数里调用的`vm._patch`

### 为什么Vue.js把`patch`的相关代码分散到各个目录？

`patch`是和平台相关的，在web和Weex环境，他们把虚拟DOM映射到“平台DOM”的方法不同，并且对"DOM"包含的属性模块创建和更新也不完全相同，所有每个平台都有各自的`nodeOps`和`modules`，他们的代码都放在`src/platforms`这个目录下

在不同的平台`patch`的主要逻辑部分是相同的，所有者部分公共的部分托管在`core`这个大目录下，差异的部分通过参数来进行区分，这里用了一个函数颗粒化的技巧，通过`createPatchFunction`把差异化参数提前固定好，这样不用每次都调用`patch`的时候都传递`nodeOps`和`modules`了。

`patch`方法，本身接受4个参数，`oldVnode`表示旧的VNode节点，也可以不存在或是一个DOM对象；`VNode`表示执行`_render`后返回的VNode节点；`hydrating`表示是否是服务端渲染；`removeOnly`是给`transition-group`用的。

接下来一个例子：
```js
var app = new Vue({
  el: '#app',
  render: function (createElement) {
    return createElement('div', {
      attrs: {
        id: 'app'
      },
    }, this.message)
  },
  data: {
    message: 'Hello Vue!'
  }
})
```
这段代码是首次渲染DOM，会通过`vm._update`的方法调用`patch`
```TypeScript
vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /*removeOnly*/)
```
因为这段代码执行的是首次渲染DOM，所以在执行`patch`函数的时候，传入`vm.$el`对应的例子中Id为`app`的DOM对象，也就是`<div id="app"></div>`，`vm.$el`的赋值是在之前的`mountComponent`函数做的，`vnode`对应是调用`render`函数的返回值，`hydrating`在非服务端渲染情况下为false（服务端渲染情况下为true），`removeOnly`为false

明确了这几个参数，回到`patch`函数的执行过程中，
```TypeScript
const isRealElement = isDef(oldVnode.nodeType)
if (!isRealElement && sameVnode(oldVnode, vnode)) {
  // patch existing root node
  patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)
} else {
  if (isRealElement) {
    // mounting to a real element
    // check if this is server-rendered content and if we can perform
    // a successful hydration.
    if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
      oldVnode.removeAttribute(SSR_ATTR)
      hydrating = true
    }
    if (isTrue(hydrating)) {
      if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
        invokeInsertHook(vnode, insertedVnodeQueue, true)
        return oldVnode
      } else if (process.env.NODE_ENV !== 'production') {
        warn(
          'The client-side rendered virtual DOM tree is not matching ' +
          'server-rendered content. This is likely caused by incorrect ' +
          'HTML markup, for example nesting block-level elements inside ' +
          '<p>, or missing <tbody>. Bailing hydration and performing ' +
          'full client-side render.'
        )
      }
    }      
    // either not server-rendered, or hydration failed.
    // create an empty node and replace it
    oldVnode = emptyNodeAt(oldVnode)
  }

  // replacing existing element
  const oldElm = oldVnode.elm
  const parentElm = nodeOps.parentNode(oldElm)

  // create new node
  // 把VNode挂载到真实DOM上
  createElm(
    vnode,
    insertedVnodeQueue,
    // extremely rare edge case: do not insert if old element is in a
    // leaving transition. Only happens when combining transition +
    // keep-alive + HOCs. (#4590)
    oldElm._leaveCb ? null : parentElm,
    nodeOps.nextSibling(oldElm)
  )
}
```
由于传入的`oldVnode`实际上是一个DOM Comtainer，所有`isRealElement`为true，接下又通过`emptyNodeAt`方法把`oldVnode`转换成`VNode`对象，然后在调用`createEle`方法，通过这个方法在这里非常重要。

```TypeScript
function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // This vnode was used in a previous render!
    // now it's used as a new node, overwriting its elm would cause
    // potential patch errors down the road when it's used as an insertion
    // reference node. Instead, we clone the node on-demand before creating
    // associated DOM element for it.
    vnode = ownerArray[index] = cloneVNode(vnode)
  }

  vnode.isRootInsert = !nested // for transition enter check
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }

  const data = vnode.data
  const children = vnode.children
  const tag = vnode.tag
  if (isDef(tag)) {
    if (process.env.NODE_ENV !== 'production') {
      if (data && data.pre) {
        creatingElmInVPre++
      }
      if (isUnknownElement(vnode, creatingElmInVPre)) {
        warn(
          'Unknown custom element: <' + tag + '> - did you ' +
          'register the component correctly? For recursive components, ' +
          'make sure to provide the "name" option.',
          vnode.context
        )
      }
    }

    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode)
    setScope(vnode)

    /* istanbul ignore if */
    if (__WEEX__) {
      // ...
    } else {
      createChildren(vnode, children, insertedVnodeQueue)
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue)
      }
      insert(parentElm, vnode.elm, refElm)
    }

    if (process.env.NODE_ENV !== 'production' && data && data.pre) {
      creatingElmInVPre--
    }
  } else if (isTrue(vnode.isComment)) {
    vnode.elm = nodeOps.createComment(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  } else {
    vnode.elm = nodeOps.createTextNode(vnode.text)
    insert(parentElm, vnode.elm, refElm)
  }
}
```
`createEle`的作用是通过虚拟节点创建真实DOM并插入到它的父节点中。`createComponent`方法的目的是尝试创建子组件，在当前这个case下它会返回值为false；接下来判断`vnode`是否包含tag,如果包含，就简单对tag的合法性在非生产环境下做校验，看是否是一个合法标签（例如：如果我们使用一个没全局注册或没局部注册的组件，就会是在这里报错）。然后再去调用平台DOM操作去创建一个占位符元素。
```TypeScript
vnode.elm = vnode.ns
    ? nodeOps.craeteElementNS(vnode.ns, tag)
    : nodeOps.createElement(tag, vnode)
```
接下来调用`createChildren`方法创建子元素：
```TypeScript
createChildren(vnode, children, insertedVnodeQueue)

function createChildren (vnode, children, insertedVnodeQueue) {
  // 如果传入children是个数组
  // 遍历children，并将当前的vnode.el作为父节点插入
  if (Array.isArray(children)) {
    if (process.env.NODE_ENV !== 'production') {
      checkDuplicateKeys(children)
    }
    for (let i = 0; i < children.length; ++i) {
      createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
    }
  } else if (isPrimitive(vnode.text)) { // 是个普通节点
  // 这里的appendChild实际上是原生JS操作DOM的API，就是对这个操作做了一层封装
    nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
  }
}
```
`createchildren`的逻辑很简单，实际上是遍历虚拟节点，递归调用`craeteElm`，这是一种常用的深度优先的遍历算法（类似于树的深度优先算法），这里需要注意的是，在遍历过程中把`vnode.elm`作为父容器的DOM节点占位符传入。
接着再调用`invokeCreateHooks`方法执行所有的created的钩子，并把`VNode`push到`insertedVnodeQueue`中
```TypeScript
if (isDef(data)) {
  invokeCreateHooks(vnode, insertedVnodeQueue)
}

function invokeCreateHooks (vnode, insertedVnodeQueue) {
  for (let i = 0; i < cbs.create.length; ++i) {
    cbs.create[i](emptyNode, vnode)
  }
  i = vnode.data.hook // Reuse variable
  if (isDef(i)) {
    if (isDef(i.create)) i.create(emptyNode, vnode)
    if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
  }
}
```
最后调用`insert`方法把`DOM`插入到父节点中，因为是递归调用，子元素会优先调用`insert`，所以整个`vnode`树节点的插入顺序是先子节点后父节点。在下`insert`方法的实现
> 文件位置： `src/core/cdom/patch.js`

```TypeScript
insert(parentElm, vnode.elm, refElm)

function insert (parent, elm, ref) {
  if (isDef(parent)) {
    if (isDef(ref)) {
      if (ref.parentNode === parent) {
        nodeOps.insertBefore(parent, elm, ref)
      }
    } else {
      nodeOps.appendChild(parent, elm)
    }
  }
}
```
就是调用一些`nodeOps`把子节点插入到父节点中。这些辅助方法本质上就是调用原生DOM的API进行DOM操作。Vue就是这样动态创建DOM的。
这些辅助方法的定义在`src/platforms/web/runtime/node-ops.js`中

在`createElm`过程中。如果`vnode`节点如果不包含`tag`，则它有可能是一个注释或者是纯文本节点，可以直接插入到父元素中。在这个例子中。最内层就是一个文本`vnode`，它的`text`值去的就是之前的`this.message`的值`hello Vue!`。

接下来回到`patch`方法，首次渲染时调用了`createElm`方法，这里传入的`parentElm`是`oldVnode.elm`的父元素，在这个例子中是id为`#pp`的div的父元素，也就是body；实际上整个递归过程就是递归创建一个完成的DOM树，并插入到Body上

最后，根据之前递归`createEle`生成的`vnode`插入顺序队列，执行相关的`insert`钩子函数。

实现过程
`new Vue` -> `init Vue` -> `vm.$mount` -> `cpmplie` -> `render` -> `vnode` -> `patch` -> `DOM`
