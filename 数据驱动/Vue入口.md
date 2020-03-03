### 1、从入口开始
> 在web应用下，分析Runtime + Compiler构建出来的Vue.js，他的入口是"src/platforms/web/entry-runtime-with-compiler"

```
/* @flow */

import config from 'core/config'
import { warn, cached } from 'core/util/index'
import { mark, measure } from 'core/util/perf'

import Vue from './runtime/index'
import { query } from './util/index'
import { compileToFunctions } from './compiler/index'
import { shouldDecodeNewlines, shouldDecodeNewlinesForHref } from './util/compat'
....
```

> 当我们执行到import Vue from './runtime/index'的时候，就是从这个入口执行代码来初始化Vue。那么是怎么初始化Vue的。


> 从Vue入口文件我们找到来源: ==import Vue from './runtime/index'==,我们先来看这边个文件的实现。位置在==src/platforms/web/runtime/index.js==

```
import Vue from 'core/index'
import config from 'core/config'
import { extend, noop } from 'shared/util'
import { mountComponent } from 'core/instance/lifecycle'
import { devtools, inBrowser, isChrome } from 'core/util/index'

import {
  query,
  mustUseProp,
  isReservedTag,
  isReservedAttr,
  getTagNamespace,
  isUnknownElement
} from 'web/util/index'

import { patch } from './patch'
import platformDirectives from './directives/index'
import platformComponents from './components/index'
....
```

> 这里的关键代码是import Vue from 'core/index' ，之后的逻辑都是对Vue这个对对象做的一些扩展，可以先不用看，我们来看真正初始化Vue的地方，在==src/core/index.js==中


```
import Vue from './instance/index'
import { initGlobalAPI } from './global-api/index'
import { isServerRendering } from 'core/util/env'
import { FunctionalRenderContext } from 'core/vdom/create-functional-component'

initGlobalAPI(Vue)

Object.defineProperty(Vue.prototype, '$isServer', {
  get: isServerRendering
})

Object.defineProperty(Vue.prototype, '$ssrContext', {
  get () {
    /* istanbul ignore next */
    return this.$vnode && this.$vnode.ssrContext
  }
})

// expose FunctionalRenderContext for ssr runtime helper installation
Object.defineProperty(Vue, 'FunctionalRenderContext', {
  value: FunctionalRenderContext
})

Vue.version = '__VERSION__'

export default Vue
```
>这里有两处关键代码 ==import Vue from './instance/index'== 和 ==initGlobalAPI(Vue)==
>初始化全局Api，先看第一部分,在src/core/instance/index.js

### Vue的定义
```
import { initMixin } from './init'
import { stateMixin } from './state'
import { renderMixin } from './render'
import { eventsMixin } from './events'
import { lifecycleMixin } from './lifecycle'
import { warn } from '../util/index'

function Vue (options) {
  if (process.env.NODE_ENV !== 'production' &&
    !(this instanceof Vue)
  ) {
    warn('Vue is a constructor and should be called with the `new` keyword')
  }
  this._init(options)
}

initMixin(Vue)
stateMixin(Vue)
eventsMixin(Vue)
lifecycleMixin(Vue)
renderMixin(Vue)

export default Vue
```
> Vue其实就是一个function实现的类，我们只能通过new Vue去实例化它。

- 为什么不用ES6的class去实现它?
> 看到后面我们会发现这里面有许多的类似于xxxMixin的函数的调用。并把Vue单做参数传入，他们的功能都是给Vue的prototype上扩展一些方法，Vue按实际功能把这些扩展分散到多个模块中去，而不是在一个模板里面全部实现，这种方式用class是很难实现的，这么做的好处就是方便代码的维护和管理。

- initGlobalAPI
> Vue.js在整个初始化的过程中，出了给它prototype上扩展的方法，还是给Vue这个对象本身扩展全局静态方法，它定义在'./core/global-api/index'


```
/* @flow */

import config from '../config'
import { initUse } from './use'
import { initMixin } from './mixin'
import { initExtend } from './extend'
import { initAssetRegisters } from './assets'
import { set, del } from '../observer/index'
import { ASSET_TYPES } from 'shared/constants'
import builtInComponents from '../components/index'

import {
  warn,
  extend,
  nextTick,
  mergeOptions,
  defineReactive
} from '../util/index'

export function initGlobalAPI (Vue: GlobalAPI) {
  // config
  const configDef = {}
  configDef.get = () => config
  if (process.env.NODE_ENV !== 'production') {
    configDef.set = () => {
      warn(
        'Do not replace the Vue.config object, set individual fields instead.'
      )
    }
  }
  Object.defineProperty(Vue, 'config', configDef)

  // exposed util methods.
  // NOTE: these are not considered part of the public API - avoid relying on
  // them unless you are aware of the risk.
  Vue.util = {
    warn,
    extend,
    mergeOptions,
    defineReactive
  }

  Vue.set = set
  Vue.delete = del
  Vue.nextTick = nextTick

  Vue.options = Object.create(null)
  ASSET_TYPES.forEach(type => {
    Vue.options[type + 's'] = Object.create(null)
  })

  // this is used to identify the "base" constructor to extend all plain-object
  // components with in Weex's multi-instance scenarios.
  Vue.options._base = Vue

  extend(Vue.options.components, builtInComponents)

  initUse(Vue)
  initMixin(Vue)
  initExtend(Vue)
  initAssetRegisters(Vue)
}
```
> 这里就是在Vue上扩展的一些全局方法的定义。有一点需要注意的是Vue.util暴露的方法最好不要依赖，因为它可能经常发生变化，是很不稳定的。

