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

