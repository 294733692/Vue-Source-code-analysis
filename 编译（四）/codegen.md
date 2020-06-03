### codegen

`template`模板通过`parse`、`optimize`处理成AST树后，编译的最后一步就是把优化后的AST树，转化为可执行的代码。

这里还是使用之前的例子

```vue
<ul :class="bindCls" class="list" v-if="isShow">
    <li v-for="(item,index) in data" @click="clickItem(index)">{{item}}:{{index}}</li>
</ul>
```

它经过`const code = generate(ast, options)`编译过后，生成的`render`代码串如下：

```typescript
with(this){
  return (isShow) ?
    _c('ul', {
        staticClass: "list",
        class: bindCls
      },
      _l((data), function(item, index) {
        return _c('li', {
          on: {
            "click": function($event) {
              clickItem(index)
            }
          }
        },
        [_v(_s(item) + ":" + _s(index))])
      })
    ) : _e()
}
```

这里的`_c`函数定义在`src/core/instance/render.js`中

```typescript
vm._c = (a, b, c, d) => createElement(vm, a, b, c, d, false)
```

而`_l`、`-v`则定义在`src/core/instance/render-helpers/index.js`中：

```typescript
export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
}
```

这里的`_c`就是执行`createElement`去创建VNode，而`_l`对应`renderList`渲染列表；`_v`则对应`createTextVNode`创建文本VNode；`_e`对应`createEmptyVNode`创建空的VNode。

在`compilerToFuncTions`中，会把这个`render`代码串转化成函数，它的定义在

> ​	src/compiler/to-function.js

```typescript
const compiled = compile(template, options)
res.render = createFunction(compiled.render, fnGenErrors)

function createFunction (code, errors) {
  try {
    return new Function(code)
  } catch (err) {
    errors.push({ err, code })
    return noop
  }
}
```

这里实际上就是把`render`代码串通过`new Function`的方式转换成可执行的函数，赋值给`vm.options.render`，这样当组件通过`vm._render`的时候，就会执行这个`render`函数，接下来来看看`render`代码串生成的过程。



#### generate

相关代码：

```typescript
const code = generate(ast, options)
```

`generate`函数定义在:

> ​	src/compiler/codegen/index.js

```typescript
export function generate (
  ast: ASTElement | void,
  options: CompilerOptions
): CodegenResult {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns
  }
}
```

`generate`先是通过`genElement(ast, state)`生成`code`，再把`code`用`with(this){return ${code}}`包裹起来，这里的`state`是`CodegenState`的一个实例。先来看`genElement`函数的实现：

```typescript
export function genElement (el: ASTElement, state: CodegenState): string {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre
  }

  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // component or element
    let code
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      let data
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        data = genData(el, state)
      }

      const children = el.inlineTemplate ? null : genChildren(el, state, true)
      code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
    // module transforms
    for (let i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code)
    }
    return code
  }
}

```

这个函数就是判断当前AST元素节点的属性执行不同的代码生成函数，在当前这里例子中，先来看看`genIf`和`genFor`的实现



- `genIf`

```typescript
export function genIf (
  el: any,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  el.ifProcessed = true // avoid recursion
  return genIfConditions(el.ifConditions.slice(), state, altGen, altEmpty)
}

function genIfConditions (
  conditions: ASTIfConditions,
  state: CodegenState,
  altGen?: Function,
  altEmpty?: string
): string {
  if (!conditions.length) {
    return altEmpty || '_e()'
  }

  const condition = conditions.shift()
  if (condition.exp) {
    return `(${condition.exp})?${
      genTernaryExp(condition.block)
    }:${
      genIfConditions(conditions, state, altGen, altEmpty)
    }`
  } else {
    return `${genTernaryExp(condition.block)}`
  }

  // v-if with v-once should generate code like (a)?_m(0):_m(1)
  function genTernaryExp (el) {
    return altGen
      ? altGen(el, state)
      : el.once
        ? genOnce(el, state)
        : genElement(el, state)
  }
}
```

`genIf`主要是通过执行`genIfConditions`，它是一次从`conditions`获取第一个`condition`，然后通过对`condition.exp`去生成一段三元运算符的代码，如果`condition.exp`存在，执行`genTernaryExp(condition.block)`，不存在就递归调用`genIfConditions`，这样如果有多个`condition`，就生成了多层三元运算逻辑，暂时忽略`v-once`的情况，所有`genTernaryExp`最终是调用了`genElement`。

在当前这里例子中，只有一个`condition`，`exp`为`isShow`。生成的代码如下:

```typescript
return (isShow) ? genElement(el, state) : _e()
```



- `genFor`

```typescript
export function genFor (
  el: any,
  state: CodegenState,
  altGen?: Function,
  altHelper?: string
): string {
  const exp = el.for
  const alias = el.alias
  const iterator1 = el.iterator1 ? `,${el.iterator1}` : ''
  const iterator2 = el.iterator2 ? `,${el.iterator2}` : ''

  if (process.env.NODE_ENV !== 'production' &&
    state.maybeComponent(el) &&
    el.tag !== 'slot' &&
    el.tag !== 'template' &&
    !el.key
  ) {
    state.warn(
      `<${el.tag} v-for="${alias} in ${exp}">: component lists rendered with ` +
      `v-for should have explicit keys. ` +
      `See https://vuejs.org/guide/list.html#key for more info.`,
      el.rawAttrsMap['v-for'],
      true /* tip */
    )
  }

  el.forProcessed = true // avoid recursion
  return `${altHelper || '_l'}((${exp}),` +
    `function(${alias}${iterator1}${iterator2}){` +
      `return ${(altGen || genElement)(el, state)}` +
    '})'
}
```

从AST元素节点中获取了和`for`相关的属性，然后返回了一个代码字符串。

在当前这个例子中,`exp`是`data`，`alias`是`item`、`iterator1`，生成的伪代码如下：

```js
_l((data), function(item, index) {
  return genElememt(el, state)
})
```



##### `genData & genChildren`

来看当前的例子，它的最外层是`ul`，首先执行了`genIf`，它最终调用了`getElement(el, state)`去生成子节点，需要注意的是，这里的`el`仍然指向的是`ul`对应的`AST`节点，但是此时的`el.ifProcessed`为`true`，所以命中最有一个`else`逻辑。

```typescript
// component or element
let code
if (el.component) {
  code = genComponent(el.component, el, state)
} else {
  const data = el.plain ? undefined : genData(el, state)

  const children = el.inlineTemplate ? null : genChildren(el, state, true)
  code = `_c('${el.tag}'${
    data ? `,${data}` : '' // data
  }${
    children ? `,${children}` : '' // children
  })`
}
// module transforms
for (let i = 0; i < state.transforms.length; i++) {
  code = state.transforms[i](el, code)
}
return code
```

这里我们需要关注，里面两个逻辑`genData`和`genChildren`

- `genData`

```typescript
export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'

  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  if (dirs) data += dirs + ','

  // key
  if (el.key) {
    data += `key:${el.key},`
  }
  // ref
  if (el.ref) {
    data += `ref:${el.ref},`
  }
  if (el.refInFor) {
    data += `refInFor:true,`
  }
  // pre
  if (el.pre) {
    data += `pre:true,`
  }
  // record original tag name for components using "is" attribute
  if (el.component) {
    data += `tag:"${el.tag}",`
  }
  // module data generation functions
  for (let i = 0; i < state.dataGenFns.length; i++) {
    data += state.dataGenFns[i](el)
  }
  // attributes
  if (el.attrs) {
    data += `attrs:{${genProps(el.attrs)}},`
  }
  // DOM props
  if (el.props) {
    data += `domProps:{${genProps(el.props)}},`
  }
  // event handlers
  if (el.events) {
    data += `${genHandlers(el.events, false, state.warn)},`
  }
  if (el.nativeEvents) {
    data += `${genHandlers(el.nativeEvents, true, state.warn)},`
  }
  // slot target
  // only for non-scoped slots
  if (el.slotTarget && !el.slotScope) {
    data += `slot:${el.slotTarget},`
  }
  // scoped slots
  if (el.scopedSlots) {
    data += `${genScopedSlots(el.scopedSlots, state)},`
  }
  // component v-model
  if (el.model) {
    data += `model:{value:${
      el.model.value
    },callback:${
      el.model.callback
    },expression:${
      el.model.expression
    }},`
  }
  // inline-template
  if (el.inlineTemplate) {
    const inlineTemplate = genInlineTemplate(el, state)
    if (inlineTemplate) {
      data += `${inlineTemplate},`
    }
  }
  data = data.replace(/,$/, '') + '}'
  // v-bind data wrap
  if (el.wrapData) {
    data = el.wrapData(data)
  }
  // v-on data wrap
  if (el.wrapListeners) {
    data = el.wrapListeners(data)
  }
  return data
}
```

`genData`函数就是根据传入的AST元素节点的属性构造出一个`data`字符串，这个在后面创建VNode的时候会作为参数传入。

之前提到了`CodegenState`的实例`state`，这里有一段关于`state`的逻辑：

```typescript
for (let i = 0; i < state.dataGenFns.length; i++) {
  data += state.dataGenFns[i](el)
}
```

`state.dataGenFns`的初始化在它的构造器中，

```typescript
export class CodegenState {
  constructor (options: CompilerOptions) {
    // ...
    this.dataGenFns = pluckModuleFunction(options.modules, 'genData')
    // ...
  }
}
```

实际上就是获取所有的`module`中的`genData`函数，其中，`class module`和`style module`定义了`genData`函数，比如定义在`src/platforms/web/compiler/modules/class.js`中的`genData`方法。

```typescript
function genData (el: ASTElement): string {
  let data = ''
  if (el.staticClass) {
    data += `staticClass:${el.staticClass},`
  }
  if (el.classBinding) {
    data += `class:${el.classBinding},`
  }
  return data
}
```

在当前这个例子中，`ul`AST元素节点定义了`el.staticClass`和`el.classBinding`，因此最终生成的`data`字符串如下:

```js
{
    staticClass: 'list',
    class: bindCls
}
```

- genChildren

  接下来看看`genChildren`，它的定义在`src/compiler/codegen/index.js`中

```typescript
export function genChildren (
  el: ASTElement,
  state: CodegenState,
  checkSkip?: boolean,
  altGenElement?: Function,
  altGenNode?: Function
): string | void {
  const children = el.children
  if (children.length) {
    const el: any = children[0]
    if (children.length === 1 &&
      el.for &&
      el.tag !== 'template' &&
      el.tag !== 'slot'
    ) {
      return (altGenElement || genElement)(el, state)
    }
    const normalizationType = checkSkip
      ? getNormalizationType(children, state.maybeComponent)
      : 0
    const gen = altGenNode || genNode
    return `[${children.map(c => gen(c, state)).join(',')}]${
      normalizationType ? `,${normalizationType}` : ''
    }`
  }
}
```

在当前这个例子中，`li`AST元素节点是`ul`AST元素节点的`children`之一，满足`(children.length === 1) && el.for && el.tag !== 'template' && el.tag !== 'solt'`的条件，因此通过`genElement(el, state)`生成`li`AST元素节点的代码，也就回到了之前调用的`genFor`生成的代码，我们把他们拼在一起生成的伪代码就是：

```js
return (isShow) ?
    _c('ul', {
        staticClass: "list",
        class: bindCls
      },
      _l((data), function(item, index) {
        return genElememt(el, state)
      })
    ) : _e()
```

在这个例子中，在执行`genElement(el, state)`的时候，`el`还是`li`AST元素节点，`el.processed`已经为`true`，所以会继续执行`genData`和`genChildren`的逻辑。由于`el.evnets`不为空，在执行`genData`的时候，会执行下面的逻辑：

```js
if (el.events) {
  data += `${genHandlers(el.events, false, state.warn)},`
}
```

`genHandlers`定义在：

> ​	src/compiler/codegen/events.js

```typescript
export function genHandlers (
  events: ASTElementHandlers,
  isNative: boolean,
  warn: Function
): string {
  let res = isNative ? 'nativeOn:{' : 'on:{'
  for (const name in events) {
    res += `"${name}":${genHandler(name, events[name])},`
  }
  return res.slice(0, -1) + '}'
}
```

`genHandler`很大一部分都是对`modifer`的处理，它最终`genData`生成的`data`如下：

```js
{
  on: {
    "click": function($event) {
      clickItem(index)
    }
  }
}
```

`genChildren`的时候，逻辑如下：

```js
export function genChildren (
  el: ASTElement,
  state: CodegenState,
  checkSkip?: boolean,
  altGenElement?: Function,
  altGenNode?: Function
): string | void {
  // ...
  const normalizationType = checkSkip
    ? getNormalizationType(children, state.maybeComponent)
    : 0
  const gen = altGenNode || genNode
  return `[${children.map(c => gen(c, state)).join(',')}]${
    normalizationType ? `,${normalizationType}` : ''
  }`
}

function genNode (node: ASTNode, state: CodegenState): string {
  if (node.type === 1) {
    return genElement(node, state)
  } if (node.type === 3 && node.isComment) {
    return genComment(node)
  } else {
    return genText(node)
  }
}
```

`genChildren`的就是遍历`children`，然后执行`genNode`方法，根据不同的`type`执行具体的方法。在当前这个例子中，`li`AST元素节点的`children`是type为2的表达式AST元素节点，那么会执行到`genText(node)`逻辑

```typescript
export function genText (text: ASTText | ASTExpression): string {
  return `_v(${text.type === 2
    ? text.expression
    : transformSpecialNewlines(JSON.stringify(text.text))
  })`
}
```

因此在我们的例子中，`genChildren` 生成的代码串如下：

```typescript
[_v(_s(item) + ":" + _s(index))]
```

和之前拼在一起，最终生成的 `code` 如下：

```typescript
 return (isShow) ?
    _c('ul', {
        staticClass: "list",
        class: bindCls
      },
      _l((data), function(item, index) {
        return _c('li', {
          on: {
            "click": function($event) {
              clickItem(index)
            }
          }
        },
        [_v(_s(item) + ":" + _s(index))])
      })
    ) : _e()
```

