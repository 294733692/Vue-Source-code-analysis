### optimize

当`tmeplate`模板经过`parse`处理后，生成了AST树，接下来就是对AST树的优化。

之所以有这么优化过程，是因为Vue是数据驱动、响应式的，但是`template`模板不是所有数据都是响应式的，也有很多数据就是在首次渲染后就不会发生变化的，那么生成的这部分相关的DOM也是不会发生改变的。

下面来看`optimize`函数的实现，该函数定义在

> src/compiler/optimizer.js



```typescript
/**
 * Goal of the optimizer: walk the generated template AST tree
 * and detect sub-trees that are purely static, i.e. parts of
 * the DOM that never needs to change.
 *
 * Once we detect these sub-trees, we can:
 *
 * 1. Hoist them into constants, so that we no longer need to
 *    create fresh nodes for them on each re-render;
 * 2. Completely skip them in the patching process.
 */
export function optimize (root: ?ASTElement, options: CompilerOptions) {
  if (!root) return
  isStaticKey = genStaticKeysCached(options.staticKeys || '')
  isPlatformReservedTag = options.isReservedTag || no
  // first pass: mark all non-static nodes.
  markStatic(root)
  // second pass: mark static roots.
  markStaticRoots(root, false)
}

function genStaticKeys (keys: string): Function {
  return makeMap(
    'type,tag,attrsList,attrsMap,plain,parent,children,attrs,start,end,rawAttrsMap' +
    (keys ? ',' + keys : '')
  )
}
```

在编译的阶段可以把一些AST节点优化成静态节点，所有整个`optimize`的过程实际上就做了两件事情，`markStatic(root)`标记静态节点，`markStaticRoots(root,false)`，标记静态根。



#### 标记静态节点

```typescript
function markStatic (node: ASTNode) {
  // 判断AST节点元素是否是静态节点  
  node.static = isStatic(node)
  if (node.type === 1) {
    // do not make component slot content static. this avoids
    // 1. components not able to mutate slot nodes
    // 2. static slot content fails for hot-reloading
    // 不要设置静态组建插槽，这样可以避免
    // 1. 无法改变插槽节点的组件
    // 2. 静态插槽内容无法热重新加载  
    if (
      !isPlatformReservedTag(node.tag) &&
      node.tag !== 'slot' &&
      node.attrsMap['inline-template'] == null
    ) {
      return
    }
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      if (!child.static) {
        node.static = false
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        const block = node.ifConditions[i].block
        markStatic(block)
        if (!block.static) {
          node.static = false
        }
      }
    }
  }
}

function isStatic (node: ASTNode): boolean {
  if (node.type === 2) { // expression
    return false
  }
  if (node.type === 3) { // text
    return true
  }
  return !!(node.pre || (
    !node.hasBindings && // no dynamic bindings
    !node.if && !node.for && // not v-if or v-for or v-else
    !isBuiltInTag(node.tag) && // not a built-in
    isPlatformReservedTag(node.tag) && // not a component
    !isDirectChildOfTemplateFor(node) &&
    Object.keys(node).every(isStaticKey)
  ))
}
```

首选执行了`node.static = isStatic(node)`。其中`isStatic`是对一个AST元素节点是否是静态的判断。

- 如果是表达式，就是非静态的；
- 如果是存文本，就是静态
- 普通元素，如果有`pre`属性，那么它使用过了`v-pre`指令，是静态
  - 如果没有`pre`，那么必须要同时满足一下条件：
    - 没有使用`v-if`、`v-for`，没有使用其他指令（不包括`v-once`），非内置组件，是平台保留标签、非带有`v-for`的`template`标签的直接节点，节点的所有属性的`key`都满足静态key，这些都满足AST元素才是一个静态节点

如果这个节点是一个普通元素，则遍历它的所有`children`，递归执行`markStatic`，因为所有的`elseif`和`else`节点都不在`children`中，如果节点的`ifconditions`不为空，则遍历`ifconditions`拿到所有的`block`，也就是他们对应的AST节点，递归执行`markStatic`。在这些递归的过程中，一旦子节点有不是`static`的情况，那么它的父节点的`static`均变成`false`



##### 标记静态根

```typescript
function markStaticRoots (node: ASTNode, isInFor: boolean) {
  if (node.type === 1) {
    if (node.static || node.once) {
      node.staticInFor = isInFor
    }
    // For a node to qualify as a static root, it should have children that
    // are not just static text. Otherwise the cost of hoisting out will
    // outweigh the benefits and it's better off to just always render it fresh.
    if (node.static && node.children.length && !(
      node.children.length === 1 &&
      node.children[0].type === 3
    )) {
      node.staticRoot = true
      return
    } else {
      node.staticRoot = false
    }
    if (node.children) {
      for (let i = 0, l = node.children.length; i < l; i++) {
        markStaticRoots(node.children[i], isInFor || !!node.for)
      }
    }
    if (node.ifConditions) {
      for (let i = 1, l = node.ifConditions.length; i < l; i++) {
        markStaticRoots(node.ifConditions[i].block, isInFor)
      }
    }
  }
}
```

`markStaticRoots`第二个参数是`isInFor`，对于已经是`static`的节点或者是`v-once`指令的节点，`node.staticInFor = isInFor`，接着就是对`staticRoot`的逻辑判断，从注释中可以看到，对于有资格成为`staticRoot`的节点，除了本身是一个静态节点外，必须满足拥有`children`，并且`children`不能只是一个文本节点，不然的话把它标记成今天根节点的收益很小了。

接下来和标记静态节点的逻辑一样，遍历`children`以及`ifConditions`，递归执行`markStaticRoots`。



`optimize`过程就是深度遍历AST树，去检测它的每一颗子树是不是静态节点，如果是静态节点则它们生成的DOM永远不改变，这对运行时对模板更新起了极大的优化作用。

通过`optimize`把整个AST树中的每一个元素节点标记了`static`和`staticRoot`，它会影响接下来执行代码的生成过程。

