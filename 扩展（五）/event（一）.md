### event

在平时开发过程中，处理组件的通信，原生的交互，都离不开事件。对于一个组件元素来说，我们不仅可以绑定原生的DOM事件，也可以绑定自定义事件。

下面来看一个例子

```js
let child = {
    template: '<button @click=chlickHandler($event)' + 
    'click me' + 
    '</button>',
    methods:{
        chickHandler(e) {
            console.log('button Clicked', e)
            this.$emit('select')
        }
    }
}

let vm = new Vue({
  el: '#app',
  template: '<div>' +
  '<child @select="selectHandler" @click.native.prevent="clickHandler"></child>' +
  '</div>',
  methods: {
    clickHandler() {
      console.log('Child clicked!')
    },
    selectHandler() {
      console.log('Child select!')
    }
  },
  components: {
    Child
  }
})
```



### 编译

先从编译阶段开始，在`parse`阶段，会执行`processAttrs`，这个方法定义在

> ​	`src/compiler/parser/index.js`

```typescript
export const onRE = /^@|^v-on:/
export const dirRE = /^v-|^@|^:/
export const bindRE = /^:|^v-bind:/
function processAttrs (el) {
  const list = el.attrsList
  let i, l, name, rawName, value, modifiers, isProp
  for (i = 0, l = list.length; i < l; i++) {
    name = rawName = list[i].name
    value = list[i].value
    if (dirRE.test(name)) {   
      el.hasBindings = true
          modifiers = parseModifiers(name)
      if (modifiers) {
        name = name.replace(modifierRE, '')
      }
      if (bindRE.test(name)) {
        // ..
      } else if (onRE.test(name)) {
        name = name.replace(onRE, '')
        addHandler(el, name, value, modifiers, false, warn)
      } else {
        // ...
      }
    } else {
      // ...
    }
  }
}

function parseModifiers (name: string): Object | void {
  const match = name.match(modifierRE)
  if (match) {
    const ret = {}
    match.forEach(m => { ret[m.slice(1)] = true })
    return ret
  }
}
```

在对标签属性的处理过程中，判断如果是指令，首先通过了`parseModifiers`解析出修饰符，然后判断如果是事件指令，则执行`addHandler(el, name, value, modifiers, false, warn)`方法，它的定义在

> src/compiler/helpers.js

```typescript
export function addHandler (
  el: ASTElement,
  name: string,
  value: string,
  modifiers: ?ASTModifiers,
  important?: boolean,
  warn?: Function
) {
  modifiers = modifiers || emptyObject
  // warn prevent and passive modifier
  /* istanbul ignore if */
  if (
    process.env.NODE_ENV !== 'production' && warn &&
    modifiers.prevent && modifiers.passive
  ) {
    warn(
      'passive and prevent can\'t be used together. ' +
      'Passive handler can\'t prevent default event.'
    )
  }

  // check capture modifier
  if (modifiers.capture) {
    delete modifiers.capture
    name = '!' + name // mark the event as captured
  }
  if (modifiers.once) {
    delete modifiers.once
    name = '~' + name // mark the event as once
  }
  /* istanbul ignore if */
  if (modifiers.passive) {
    delete modifiers.passive
    name = '&' + name // mark the event as passive
  }

  // normalize click.right and click.middle since they don't actually fire
  // this is technically browser-specific, but at least for now browsers are
  // the only target envs that have right/middle clicks.
  if (name === 'click') {
    if (modifiers.right) {
      name = 'contextmenu'
      delete modifiers.right
    } else if (modifiers.middle) {
      name = 'mouseup'
    }
  }

  let events
  if (modifiers.native) {
    delete modifiers.native
    events = el.nativeEvents || (el.nativeEvents = {})
  } else {
    events = el.events || (el.events = {})
  }

  const newHandler: any = {
    value: value.trim()
  }
  if (modifiers !== emptyObject) {
    newHandler.modifiers = modifiers
  }

  const handlers = events[name]
  /* istanbul ignore if */
  if (Array.isArray(handlers)) {
    important ? handlers.unshift(newHandler) : handlers.push(newHandler)
  } else if (handlers) {
    events[name] = important ? [newHandler, handlers] : [handlers, newHandler]
  } else {
    events[name] = newHandler
  }

  el.plain = false
}
```

`addHandler`实际上做了3件事，首先根据`modifier`修饰符对事件名`name`做处理，接着处理`modifier.native`判断是一个纯原生事件还是普通事件，最后按照`name`对事件做归类，并把回调函数的字符串保留到对应的事件中。

在当前这个例子中，父组件的`child`节点生成的`el.events`和`el.nativeEvents`如下：

```js
el.events = {
    select: {
        value: 'selectHandler'
    }
}

el.nativeEvents = {
    click: {
        value: 'clickHandler',
        modifier:{
            prevent: true
        }
    }
}
```

子组件的`button`节点生成的`el.events`如下

```js
el.events = {
    click: {
        value: 'handlerClick($event)'
    }
}
```

然后在`codogen`的阶段，会在`genData`函数中根据AST元素节点上的`events`和`nativeEvents`生成`data`数据，它的定义在

> ​	src/compiler/codegen/index.js

```js
export function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'
  // ...
  if (el.events) {
    data += `${genHandlers(el.events, false, state.warn)},`
  }
  if (el.nativeEvents) {
    data += `${genHandlers(el.nativeEvents, true, state.warn)},`
  }
  // ...
  return data
}
```

对于这两个属性，都会调用`genHandlers`函数，定义在

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

const fnExpRE = /^\s*([\w$_]+|\([^)]*?\))\s*=>|^function\s*\(/
const simplePathRE = /^\s*[A-Za-z_$][\w$]*(?:\.[A-Za-z_$][\w$]*|\['.*?']|\[".*?"]|\[\d+]|\[[A-Za-z_$][\w$]*])*\s*$/
function genHandler (
  name: string,
  handler: ASTElementHandler | Array<ASTElementHandler>
): string {
  if (!handler) {
    return 'function(){}'
  }

  if (Array.isArray(handler)) {
    return `[${handler.map(handler => genHandler(name, handler)).join(',')}]`
  }

  const isMethodPath = simplePathRE.test(handler.value)
  const isFunctionExpression = fnExpRE.test(handler.value)

  if (!handler.modifiers) {
    if (isMethodPath || isFunctionExpression) {
      return handler.value
    }
    /* istanbul ignore if */
    if (__WEEX__ && handler.params) {
      return genWeexHandler(handler.params, handler.value)
    }
    return `function($event){${handler.value}}` // inline statement
  } else {
    let code = ''
    let genModifierCode = ''
    const keys = []
    for (const key in handler.modifiers) {
      if (modifierCode[key]) {
        genModifierCode += modifierCode[key]
        // left/right
        if (keyCodes[key]) {
          keys.push(key)
        }
      } else if (key === 'exact') {
        const modifiers: ASTModifiers = (handler.modifiers: any)
        genModifierCode += genGuard(
          ['ctrl', 'shift', 'alt', 'meta']
            .filter(keyModifier => !modifiers[keyModifier])
            .map(keyModifier => `$event.${keyModifier}Key`)
            .join('||')
        )
      } else {
        keys.push(key)
      }
    }
    if (keys.length) {
      code += genKeyFilter(keys)
    }
    // Make sure modifiers like prevent and stop get executed after key filtering
    if (genModifierCode) {
      code += genModifierCode
    }
    const handlerCode = isMethodPath
      ? `return ${handler.value}($event)`
      : isFunctionExpression
        ? `return (${handler.value})($event)`
        : handler.value
    /* istanbul ignore if */
    if (__WEEX__ && handler.params) {
      return genWeexHandler(handler.params, code + handlerCode)
    }
    return `function($event){${code}${handlerCode}}`
  }
}
```

`genHandlers`方法遍历事件对象`events`，对同一事件名称的事件调用`genHandler(name,events[name])`方法，它的内容看起来比较多，实际上逻辑比较简单，首先判断如果`handler`是一个数组，就遍历它然后递归调用`genHandler`方法拼接结果，然后判断`hanlder.value`是一个函数的调用路径还是一个函数表达式，接着对`modifiers`做判断，对于没有`modifiers`的情况，就根据`handler.value`不同情况处理，要么直接返回，要么返回一个函数包裹表达式；对于有`modifiers`的情况，则对各个不同的`modifier`情况做不同处理，添加相应的代码串。

对于当前这个例子而言，父组件生成的`data`串为：

```js
{
    on: {'select': selectHandler},
    nativeOn: {
        'click': function($event) {
            $event.preventDefault()
            return clickHandler($event)
		}
    }
}
```

子组件生成的`data`串为：

```js
{
    on: {"click": function($event){
        	clickHandler($event)
    	}
    }
}
```

到这里就编译完了，接下来看一下运行时，这部分是怎么实现的。其实Vue的事件有两种，一种是原生DOM事件，一种是用户自定义事件。接下来分别看看。