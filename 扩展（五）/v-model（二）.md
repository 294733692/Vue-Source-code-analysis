### v-model

我们理解的双向绑定是：除了数据驱动DOM以外，DOM的变化返回来影响数据。这是一个双向的管理。

这里我们来看`v-model`的实现。

`v-model`可以作用在普通表单元素上，又可以作用在组件上，`v-model`实际上就是vue提供的一个语法糖。



#### 表单元素

举一个例子

```js
let vm = new Vue({
  el: '#app',
  template: '<div>'
  + '<input v-model="message" placeholder="edit me">' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  }
})
```

在这个例子上，我们给`input`元素设置上了`v-model`属性，绑定了`message`，当我们在`input`输入文字的时候，`message`会同步的发生变化，接下来我们来分析Vue是如何实现这一效果的。

这个过程也是先从编译过程开始的，首先是`parse`阶段，`v-model`指令被当做普通指令解析到`el-directives`中，然后在`codegen`阶段，执行`genData`的时候，会执行`const dirs  = genDirectives(el, state)`，这个函数定义在

> ​	src/compiler/codegen/index.js

```typescript
function genDirectives (el: ASTElement, state: CodegenState): string | void {
  const dirs = el.directives
  if (!dirs) return
  let res = 'directives:['
  let hasRuntime = false
  let i, l, dir, needRuntime
  for (i = 0, l = dirs.length; i < l; i++) {
    dir = dirs[i]
    needRuntime = true
    const gen: DirectiveFunction = state.directives[dir.name]
    if (gen) {
      // compile-time directive that manipulates AST.
      // returns true if it also needs a runtime counterpart.
      needRuntime = !!gen(el, dir, state.warn)
    }
    if (needRuntime) {
      hasRuntime = true
      res += `{name:"${dir.name}",rawName:"${dir.rawName}"${
        dir.value ? `,value:(${dir.value}),expression:${JSON.stringify(dir.value)}` : ''
      }${
        dir.arg ? `,arg:"${dir.arg}"` : ''
      }${
        dir.modifiers ? `,modifiers:${JSON.stringify(dir.modifiers)}` : ''
      }},`
    }
  }
  if (hasRuntime) {
    return res.slice(0, -1) + ']'
  }
}
```

`genDirectives`方法实际上遍历`el.directives`，然后获取每个指令对应的方法`const gen: DirectiveFunction = state.directives[dir.name]`，这个指令方法实际上是在实例化`codeGenState`的时候通过`option`传入的，这个`option`就是编译相关的配置，它在不同的平台下配置不同，在`web`环境下的定义在

> ​	src/platforms/web/compiler/options.js

```typescript
export const baseOptions: CompilerOptions = {
  expectHTML: true,
  modules,
  directives,
  isPreTag,
  isUnaryTag,
  mustUseProp,
  canBeLeftOpenTag,
  isReservedTag,
  getTagNamespace,
  staticKeys: genStaticKeys(modules)
}
```

`directives`定义在

> ​	src/platforms/web/compiler/directives/index.js

```js
export default {
  model,
  text,
  html
}
```

对于`v-model`而言，对应的`directives`函数是在

> ​	src/platforms/web/compiler/directives/model.js

中的`model`函数：

```typescript
export default function model (
  el: ASTElement,
  dir: ASTDirective,
  _warn: Function
): ?boolean {
  warn = _warn
  const value = dir.value
  const modifiers = dir.modifiers
  const tag = el.tag
  const type = el.attrsMap.type

  if (process.env.NODE_ENV !== 'production') {
    // inputs with type="file" are read only and setting the input's
    // value will throw an error.
    if (tag === 'input' && type === 'file') {
      warn(
        `<${el.tag} v-model="${value}" type="file">:\n` +
        `File inputs are read only. Use a v-on:change listener instead.`
      )
    }
  }

  if (el.component) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (tag === 'select') {
    genSelect(el, value, modifiers)
  } else if (tag === 'input' && type === 'checkbox') {
    genCheckboxModel(el, value, modifiers)
  } else if (tag === 'input' && type === 'radio') {
    genRadioModel(el, value, modifiers)
  } else if (tag === 'input' || tag === 'textarea') {
    genDefaultModel(el, value, modifiers)
  } else if (!config.isReservedTag(tag)) {
    genComponentModel(el, value, modifiers)
    // component v-model doesn't need extra runtime
    return false
  } else if (process.env.NODE_ENV !== 'production') {
    warn(
      `<${el.tag} v-model="${value}">: ` +
      `v-model is not supported on this element type. ` +
      'If you are working with contenteditable, it\'s recommended to ' +
      'wrap a library dedicated for that purpose inside a custom component.'
    )
  }

  // ensure runtime directive metadata
  return true
}
```

也就是说我们执行`needRunTime = !!gen(el, dir, state.warn)`就是在执行`model`函数，它会根据AST元素节点的不同情况执行不同的逻辑，对于我们当前这个例子来说，它会执行`genDefaultModel(el, value, modifiers)`这个逻辑。来看看这个逻辑都实现。

```typescript
function genDefaultModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
): ?boolean {
  const type = el.attrsMap.type

  // warn if v-bind:value conflicts with v-model
  // except for inputs with v-bind:type
  if (process.env.NODE_ENV !== 'production') {
    const value = el.attrsMap['v-bind:value'] || el.attrsMap[':value']
    const typeBinding = el.attrsMap['v-bind:type'] || el.attrsMap[':type']
    if (value && !typeBinding) {
      const binding = el.attrsMap['v-bind:value'] ? 'v-bind:value' : ':value'
      warn(
        `${binding}="${value}" conflicts with v-model on the same element ` +
        'because the latter already expands to a value binding internally'
      )
    }
  }

  const { lazy, number, trim } = modifiers || {}
  const needCompositionGuard = !lazy && type !== 'range'
  const event = lazy
    ? 'change'
    : type === 'range'
      ? RANGE_TOKEN
      : 'input'

  let valueExpression = '$event.target.value'
  if (trim) {
    valueExpression = `$event.target.value.trim()`
  }
  if (number) {
    valueExpression = `_n(${valueExpression})`
  }

  let code = genAssignmentCode(value, valueExpression)
  if (needCompositionGuard) {
    code = `if($event.target.composing)return;${code}`
  }

  addProp(el, 'value', `(${value})`)
  addHandler(el, event, code, null, true)
  if (trim || number) {
    addHandler(el, 'blur', '$forceUpdate()')
  }
}
```

`genDefaultModel`函数先处理了`modifies`，它的不同主要影响的是`event`和`valueExpression`的值，对于当前这个例子，`event`为`input`，`valueExpression`为`$event.target.value.trim()`，然后去执行`genAssignmentCode`去生成代码，它的定义在

> ​	src/compiler/directives/model.js

```typescript
/**
 * Cross-platform codegen helper for generating v-model value assignment code.
 */
export function genAssignmentCode (
  value: string,
  assignment: string
): string {
  const res = parseModel(value)
  if (res.key === null) {
    return `${value}=${assignment}`
  } else {
    return `$set(${res.exp}, ${res.key}, ${assignment})`
  }
}
```

这个方法先对`v-model`对应的`value`做了解析，处理了很多的情况，对于我们当前这个例子来说，`value`就是`message`，所以返回的`res.key`为`null`，然后我们就得到了`message=$event.target.value`。然后我们进入了`needCompositionGuard`为true的逻辑，所以最终的`code`为`if ($event.target.composing) return ;message=$event.target.value`

`code`生成完成后，又执行了两句非常关键的代码：

```typescript
addProp(el, 'value', `(${value})`)
addHandler(el, event, code, null, true)
```

这实际上就是`input`实现`v-model`的精髓。通过修改AST元素，给`el`添加一个`prop`，相当于我们在`input`上动态绑定了`value`，又给`el`添加了事件处理，相当于在`input`上绑定了`input`事件，转换为模板如下

```vue
<input v-bind:value='message' v-on:input='message=$message.target.value'/>
```

其实就是动态绑定了`input`的`value`指向`message`变量，并且在触发`input`事件的时候动态把`message`设置为目标值，这样实际上就完成了数据双向绑定，所以说`v-model`就是语法糖。



接下来回到`genDirectives`函数，它接下来的逻辑就是根据指令生成一些`data`的代码；

```typescript
if (needRuntime) {
  hasRuntime = true
  res += `{name:"${dir.name}",rawName:"${dir.rawName}"${
    dir.value ? `,value:(${dir.value}),expression:${JSON.stringify(dir.value)}` : ''
  }${
    dir.arg ? `,arg:"${dir.arg}"` : ''
  }${
    dir.modifiers ? `,modifiers:${JSON.stringify(dir.modifiers)}` : ''
  }},`
}
```

对于我们当前这个例子而言，最终生成的`render`代码如下

```js
with(this) {
  return _c('div',[_c('input',{
    directives:[{
      name:"model",
      rawName:"v-model",
      value:(message),
      expression:"message"
    }],
    attrs:{"placeholder":"edit me"},
    domProps:{"value":(message)},
    on:{"input":function($event){
      if($event.target.composing)
        return;
      message=$event.target.value
    }}}),_c('p',[_v("Message is: "+_s(message))])
    ])
}
```

这里的`_c`和`_v`就是前面提到过的[事件处理](https://github.com/294733692/Vue-Source-code-analysis/blob/master/编译（四）/codegen.md)已经分析了，所以对于`input`的`v-model`来说，完全就是语法糖，并且对于其他表单元素基本的套路都是一样的，区别在于生成事件的代码可能会有不同。

除了`v-model`作用在表单元素上，新版的vue还把这个语法糖作用到了组件上，来看看这块的实现。



#### 组件

举一个例子

```js
let Child = {
  template: '<div>'
  + '<input :value="value" @input="updateValue" placeholder="edit me">' +
  '</div>',
  props: ['value'],
  methods: {
    updateValue(e) {
      this.$emit('input', e.target.value)
    }
  }
}

let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child v-model="message"></child>' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  },
  components: {
    Child
  }
})
```

从这个例子可以看到，父组件引用`child`子组件的地方使用了`v-model`关联了数据`message`；而子组件定义了一个`value`的`prop`，并且在`input`的回调函数中，通过`this.$emit('input', e.target.value)`派发一个事件，为了让`v-model`生效，这两点是必须的。

来看下实现原理，还是从编译阶段开始，对于父组件而言，在编译阶段会解析`v-model`指令，依然会执行`genData`函数中的`genDirectives`函数，接着执行

> ​	src/platforms/web/compiler/directives/model.js

中定义的`model`函数，并执行了下面的逻辑：

```typescript
else if (!config.isReservedTag(tag)) {
  genComponentModel(el, value, modifiers);
  return false
}
```

`genComponentModel`函数定义在

> ​	src/compiler/directives/model.js

```typescript
export function genComponentModel (
  el: ASTElement,
  value: string,
  modifiers: ?ASTModifiers
): ?boolean {
  const { number, trim } = modifiers || {}

  const baseValueExpression = '$$v'
  let valueExpression = baseValueExpression
  if (trim) {
    valueExpression =
      `(typeof ${baseValueExpression} === 'string'` +
        `? ${baseValueExpression}.trim()` +
        `: ${baseValueExpression})`
  }
  if (number) {
    valueExpression = `_n(${valueExpression})`
  }
  const assignment = genAssignmentCode(value, valueExpression)

  el.model = {
    value: `(${value})`,
    expression: `"${value}"`,
    callback: `function (${baseValueExpression}) {${assignment}}`
  }
}
```

上面这个函数的逻辑不怎么复杂，对于当前这个例子来说，生成的代码如下：

```js
el.model = {
    callback: 'function ($$v) {message=$$v}',
    expression: '"message"',
    value: '(message)'
}
```

在执行完`genDirectives`的逻辑后，`genData`函数中有这么一段逻辑

```typescript
if (el.model) {
  data += `model:{value:${
    el.model.value
  },callback:${
    el.model.callback
  },expression:${
    el.model.expression
  }},`
}
```

这里最终父组件生成的`render`代码如下：

```js
with(this){
  return _c('div',[_c('child',{
    model:{
      value:(message),
      callback:function ($$v) {
        message=$$v
      },
      expression:"message"
    }
  }),
  _c('p',[_v("Message is: "+_s(message))])],1)
}
```

这里父组件就处理完了，接下来就是创建子组件`vnode`，会执行`createComponent`函数，它的定义在

> src/core/vdom/create-components.js

```typescript
export function createComponent (
 Ctor: Class<Component> | Function | Object | void,
 data: ?VNodeData,
 context: Component,
 children: ?Array<VNode>,
 tag?: string
): VNode | Array<VNode> | void {
 // ...
 // transform component v-model data into props & events
 if (isDef(data.model)) {
   transformModel(Ctor.options, data)
 }

 // extract props
 const propsData = extractPropsFromVNodeData(data, Ctor, tag)
 // ...
 // extract listeners, since these needs to be treated as
 // child component listeners instead of DOM listeners
 const listeners = data.on
 // ...
 const vnode = new VNode(
   `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
   data, undefined, undefined, undefined, context,
   { Ctor, propsData, listeners, tag, children },
   asyncFactory
 )
 
 return vnode
}
```

这里会对`data.model`做处理，执行了`transformModel(Ctor.options, data)`方法。

```typescript
// transform component v-model info (value and callback) into
// prop and event handler respectively.
function transformModel (options, data: any) {
  const prop = (options.model && options.model.prop) || 'value'
  const event = (options.model && options.model.event) || 'input'
  ;(data.props || (data.props = {}))[prop] = data.model.value
  const on = data.on || (data.on = {})
  if (isDef(on[event])) {
    on[event] = [data.model.callback].concat(on[event])
  } else {
    on[event] = data.model.callback
  }
}
```

`trabsformModel`逻辑很简单，给`data.props`添加`data.model.value`，并且给`data.on`添加了`data.model.callback`，对于我们当前这个例子来说，扩展的结果如下：

```typescript
data.props = {
  value: (message),
}
data.on = {
  input: function ($$v) {
    message=$$v
  }
} 
```

其实就相当于我们是这样编辑父组件的

```js
let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child :value="message" @input="message=arguments[0]"></child>' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  },
  components: {
    Child
  }
})
```

当子组件传递的`value`绑定到当前父组件的`message`，同时监听自定义`input`事件，当子组件派发`input`事件的时候，父组件会在事件回调哈数中修改`message`的值，同时`value`也会发生变化，子组件的`input`值被更新。

这里就是vue的父子通讯模式，父组件通过`prop`把数据传递到子组件，子组件修改了数据后把改变通过`$emit`事件的方式通知父组件，所以说组件上的`v-model`也是一个语法糖。

这里，我们可以看到组件的`v-model`的实现，子组件`value prop`以及派发的`input`事件名是可配的，可以看到`transformModel`中对这部分的处理

```typescript
function transformModel (options, data: any) {
  const prop = (options.model && options.model.prop) || 'value'
  const event = (options.model && options.model.event) || 'input'
  //
}
```

也就是说可以在定义子组件的时候通过`model`选项配置子组件接收的`prop`名以及派发的事件名，举一个例子：

```typescript
let Child = {
  template: '<div>'
  + '<input :value="msg" @input="updateValue" placeholder="edit me">' +
  '</div>',
  props: ['msg'],
  model: {
    prop: 'msg',
    event: 'change'
  },
  methods: {
    updateValue(e) {
      this.$emit('change', e.target.value)
    }
  }
}

let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child v-model="message"></child>' +
  '<p>Message is: {{ message }}</p>' +
  '</div>',
  data() {
    return {
      message: ''
    }
  },
  components: {
    Child
  }
})
```

子组件修改了接收的`prop`名以及派发事件名，然而这一切父组件作为调用方是不同关心的，那么这么做的好处就是我们可以把`value`这个`prop`作为其他用途。