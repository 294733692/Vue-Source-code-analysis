### 检测变化的注意事项

当我们了解到响应式数据对对象以及它的`getter`和`setter`做了一些了解，但是这些都是常规场景的情况，也有一些特殊的场景。这里我们来看看vue在特殊场景下是怎么对这些数据是怎么处理的

#### 对象添加属性

对于使用`Object.defineProperty`实现响应式对象，当我们去给这个对象添加一个新的属性的时候，是不能触发它的`setter`的，举一个例子：

```js
let vm = new Vue({
    data() {
        return {
            a: 1
        }
    }
})
// 注意，这里的b是不是响应式的
vm.b = 2
```

Vue官方给定的定义是：Vue不允许在已创建的实例上添加新的根级响应式属性（root-level reactive property）。

那么如果Vue不允许，我们应该怎么将新增加的属性变为响应式的呢。

相信大家都用过`Vue.set(object, key, value)`这个Api把。这个就是Vue官方提供的方法，它可以将将响应式属性添加嵌套到对象上。

那么我们来看看这个全局方法，方法定义在

> ​	src/core/global-api/index.js

```typescript
Vue.set = set
```

我们可以看到这里调用set方法，该方法定义在

> ​	src/core/observer/index.js

```typescript
/**
 * Set a property on an object. Adds the new property and
 * triggers change notification if the property doesn't
 * already exist.
 */
export function set (target: Array<any> | Object, key: any, val: any): any {
  // 判断目标数组是否是为null或undefined或者是基本类型是否是string、number、symbol、boolean  
  if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
  // 判断修改目前是否是数组，判断key是否是数组有效索引  
  if (Array.isArray(target) && isValidArrayIndex(key)) {
    // 将target.length 设置为target.length和key的最大值
    // 防止出现,下面这种情况
    //    arr = [1, 3]
    //	  Vue.set(arr, 10, 1)
    target.length = Math.max(target.length, key)
    // 将目标值给替换  
    target.splice(key, 1, val)
    return val
  }
  // 判断key是目标对象的属性，并且key不是Object原型上面的属性
  // 说明这个key本来就在对象上面定义过了，直接修改值就可以了  
  if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
  // 如果属性有`__ob__`的话，那么说明这个对象就是响应式数据 
  const ob = (target: any).__ob__
  // 判断目标对象是否是Vue实例，或者是根数据对象
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
  // 如果当前target对象不是响应式对象，直接返回赋值  
  if (!ob) {
    target[key] = val
    return val
  }
  // 这段代码是重点，这里才是Vue.set()真正处理对象的地方
  // 这里给新添加的属性添加依赖，以后直接修改这个新属性的时候就会触发页面的从新渲染
  // 将新添加的属性变为响应式对象  
  defineReactive(ob.value, key, val)
  // 触发当前依赖通知
  ob.dep.notify()
  return val
}
```

`set`方法接收三个参数

- `target`： `target`可以是任意类型数组，也可以是对象
- `key`：`key`值代表数组的下标或者而是对象的键值
- `val`：`val`代表添加的值

第一个判断是判断是否是生产环境

```js
if (process.env.NODE_ENV !== 'production' &&
    (isUndef(target) || isPrimitive(target))
  ) {
    warn(`Cannot set reactive property on undefined, null, or primitive value: ${(target: any)}`)
  }
```

这段代码判断目标`target`是否是为null或undefined或者基本类型是否是string、number、symbol、boolean

接着向下看

```typ
// 判断修改目前是否是数组，判断key是否是数组有效索引  
if (Array.isArray(target) && isValidArrayIndex(key)) {
    // 将target.length 设置为target.length和key的最大值
    target.length = Math.max(target.length, key)
    // 将目标值给替换  
    target.splice(key, 1, val)
    return val
  }
```

这里首先判断了`target`是否是数组并且`key`是否是`target`的有效索引，接下来看`targer.length = Math.max(target.length, key)`，这里为什么要把`target`的长度设置为`target.length`和`key`的最大值呢。这里是为了我们出现下面的骚炒作：

```js
arr = [1, 3]
Vue.set(arr, 10, 1)
```

最后利用`splice`方法将值给替换了，看到这里，没有想到这里实现方法这么简单。

接下来看下一个判断

```typescript
if (key in target && !(key in Object.prototype)) {
    target[key] = val
    return val
  }
```

这里首先判断了`key`是否是`target`里面的属性，并且不是对象原型上的属性，满足这两个条件，说明这个key本来就在对象上面定义过了，直接修改值就可以了 。

```typescript
// 如果属性有`__ob__`的话，那么说明这个对象就是响应式数据 
  const ob = (target: any).__ob__
  // 判断目标对象是否是Vue实例，或者是根数据对象
  if (target._isVue || (ob && ob.vmCount)) {
    process.env.NODE_ENV !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    )
    return val
  }
```

这里首先将`target`的``__ob__`数据给取出来（如果有`__ob__`这个属性的话，那么说明`target`已经是响应式数据了）。接下来判断`target`是否是`Vue`实例（如果是Vue的实例的话，`target`有`_isVue`的属性)或者`target`是响应式数据对象并且是根数据对象。在生成环境报错提示用户。并返回值。

下一段代码

```js
  if (!ob) {
    target[key] = val
    return val
  }
```

这里判断`target`对象是不是响应对象，如果不是响应式对象，直接赋值返回

下面一段代码是最重要的，也是`Vue.set()`真正处理对象的地方。通过`defineReactive(ob.value, key, val)`把新添加的属性变成响应式对象，并且通过`ob.dep.notify()`手动的触发依赖通知

```js
defineReactive(ob.value, key, val)
ob.dep.notify()
return val
```

在给对象添加`getter`的时候，有这么一段逻辑：可以看看[依赖收集(getter)](https://github.com/294733692/Vue-Source-code-analysis/blob/master/响应式原理（三）/依赖收集（二）.md)

```js
/**
 * Define a reactive property on an Object.
 */
export function defineReactive (
  obj: Object,
  key: string,
  val: any,
  customSetter?: ?Function,
  shallow?: boolean
) {
  const dep = new Dep()

  // 拿到obj上的属性描述符
  const property = Object.getOwnPropertyDescriptor(obj, key)
  // configurable 属性描述符为true，才能修改数据对象
  // 如果configurable为false，那么不能修改数据对象，直接return
  if (property && property.configurable === false) {
    return
  }

  // cater for pre-defined getter/setters
  // 保存property对象的get和set函数  
  const getter = property && property.get
  const setter = property && property.set
  // 当只传递了两个参数的时候，说明没有传递第三个参数val
  if ((!getter || setter) && arguments.length === 2) {
    // 那么这个时候需要根据key主动去对象获取相依的值  
    val = obj[key]
  }

  let childOb = !shallow && observe(val)
  // 重新定义getter和setter方法，覆盖原来的set和get方法。
  // 上面将原有方法get就set缓存，并在重新定义的get和set方法中调用缓存的get和set函数
  // 这样做到了不影响原有属性的读写
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    // 触发get方法，
    // 完成依赖收集  
    get: function reactiveGetter () {
      // 判断getter函数是否存在，是就直接调用该函数，没有就返回val  
      const value = getter ? getter.call(obj) : val
      // 依赖收集
      // target是Dep类里面的静态属性，用于保存要收集的依赖（Watcher）
      if (Dep.target) {
        // 通过dep的depend方法，将依赖收集到dep中  
        dep.depend()
        // 
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
      return value
    },
    // ...
  })
}
```

在`getter`的过程中，判断了`childOb`，并且调用了`childOb.dep.depend()`收集了依赖。所有在使用`Vue.set()`的时候通过`Ob.dep.notity()`能够通知到`watcher`，从而让新添加的新属性到对象也可以检测到变化。如果是数组的话，就通过`dependArray(value)`把每个数组也做依赖收集



#### 数组

在开发的时候，遇到过这么一种情况，当我们在Vue中修改某一个数组的值的时候，会发现视图没有更新，而当我们使用`push、splice、pop`等属性的时候就可以更新。

在Vue开发中有会这么两种情况，Vue不能检测到数组的变化

- 利用索引直接设置值的时候：`vm.item[index] = newValue`
- 直接修改数组长度的时候：`vm.item.leng = newLength`

这两种情况，我们可以采取`Vue.set(item, index, newValue)`和`vm.item.splice(newLength)`方法进行处理，让Vue能够检测到数组数据的变化。

在上面`Vue.set()`的实现的时候，有这么一段代码

```js
// 判断修改目前是否是数组，判断key是否是数组有效索引  
if (Array.isArray(target) && isValidArrayIndex(key)) {
    // 将target.length 设置为target.length和key的最大值
    target.length = Math.max(target.length, key)
    // 将目标值给替换  
    target.splice(key, 1, val)
    return val
  }
```

这里当`target`是数组的时候，直接使用`target.splice(key, 1, val)`来添加的。

我们知道，通过`observe`方法去观测对象的时候会实例化`Observer`类，这里对数组做了专门的处理。`Observer`类定义在

> src/core/observer/index.js

```js
/**
 * Observer class that is attached to each observed
 * object. Once attached, the observer converts the target
 * object's property keys into getter/setters that
 * collect dependencies and dispatch updates.
 */
export class Observer {
  value: any;
  dep: Dep;
  vmCount: number; // number of vms that has this object as root $data

  constructor (value: any) {
    // 实例对象的Value属性，引用为数据对象  
    this.value = value
    // 实例对象的dep属性，获取Dep构造函数实例的引用，用于依赖收集  
    this.dep = new Dep()
    this.vmCount = 0
    // 为value定义了一个__ob__属性  
    def(value, '__ob__', this)
    if (Array.isArray(value)) {
      // hasProto是一个布尔值，用来检测当前环境是否可以使用__proto__属性  
      const augment = hasProto
        ? protoAugment
        : copyAugment
      augment(value, arrayMethods, arrayKeys)
      this.observeArray(value)
    } else {
      this.walk(value)
    }
  }
  // ...
}
```

这里我们只保留和数组相关的代码。这里的`hasProto`是一个Boolean值，用来检测当前环境是否可以使用`__proto__`属性。如果可以使用就调用`protoAugment`方法，不可以使用就调用`copyAugment`方法。

来看看这两个函数的实现



##### copyAugment

```js
/**
 * Augment an target Object or Array by defining
 * hidden properties.
 */
/* istanbul ignore next */
function copyAugment (target: Object, src: Object, keys: Array<string>) {
  for (let i = 0, l = keys.length; i < l; i++) {
    const key = keys[i]
    def(target, key, src[key])
  }
}
```

这个方法就是遍历**keys**，通过`def`，也就是`Object.defineProperty`去定义自身的属性。

##### protoAugment 

```js
/**
 * Augment an target Object or Array by intercepting
 * the prototype chain using __proto__
 */
function protoAugment (target, src: Object, keys: any) {
  /* eslint-disable no-proto */
  target.__proto__ = src
  /* eslint-enable no-proto */
}
```

这里实现很简单，就是想`target.__proto__`直接修改为`src`。这里的`src`实际上就是上面传入的`arrayMethods`。`arrayMethods`定义在

> src/core/observer/array.js

```typescript
import { def } from '../util/index'

// 获取数组的原型。上面有常用的数组方法
const arrayProto = Array.prototype
// 创建了一个空对象arrayMethods，并将arrayMethods的原型指向arrayProto
export const arrayMethods = Object.create(arrayProto)

// 列出需要重写的数组方法名
const methodsToPatch = [
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]

/**
 * Intercept mutating methods and emit events
 */
// 遍历上述数组方法名，依次将重写后的数组方法添加到arrayMethods对象上
methodsToPatch.forEach(function (method) {
  // cache original method
  // 保存一份当前数组名称对应的原始的数组方法  
  const original = arrayProto[method]
  // 将重写的方法定义到arrayMethods对象上，mutator函数就是对数组的重写
  def(arrayMethods, method, function mutator (...args) {
    // 调用数组的原始方法，并传入参数args,将执行结果赋值给result  
    const result = original.apply(this, args)
    // 当数组调用重写后的方法的时候，this就指向该数组，当数组为响应式的时候，就可以获取到__ob__属性
    const ob = this.__ob__
    /
    // 获取到插入的值
    let inserted
    // 对能够增加数组长度的push、unshift、splice方法做判断
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    // 把新添加的值变成一个响应式对象  
    if (inserted) ob.observeArray(inserted)
    // notify change
    // 手动触发依赖通知  
    ob.dep.notify()
    return result
  })
})
```

