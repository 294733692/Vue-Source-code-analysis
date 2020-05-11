### parse

编译过程就是对模板做解析，生成*AST（抽象语法树）*，是对源代码的抽象语法结构的树状表现形式。

[大家可以去看看这边文章，详细的介绍了AST](https://segmentfault.com/a/1190000016231512?utm_source=tag-newest)

解析的过程是非常复杂，因为需要用到大量的正则表达式对字符串进行解析。

可以看下这个例子：

```vue
let str = `<ul :class="bindCls" class="list" v-if="isShow">
    		<li v-for="(item,index) in data" @click="clickItem(index)">{{item}}:{{index}}</li>
		   </ul>`
console.log(parse(str))
```

<img src="C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200511133514548.png" alt="image-20200511133514548" />

从这张截图上可以看到，生成的*AST*是一个树状结构，没一个节点都是一个`ast element`，除了自身的一些属性，还维护了它的父子关系，例如`parent`指向它的父节点，`children`指向它的子节点。

接下来 ，我们来看看`parse`函数，该函数定义在

> src/compiler/parser/index.js

`parse`函数，代码很长，这里先将代码拆分为伪代码的形式，方便理解和阅读

```typescript
export function parse (
  template: string,
  options: CompilerOptions
): ASTElement | void {
  // 从options获取方法和配置  
  getFnsAndConfigFromOptions(options)
  
  // ...  
  
  // 解析html模板
  parseHTML(template, {
    // options ...
    start (tag, attrs, unary) {
      let element = createASTElement(tag, attrs)
      processElement(element)
      treeManagement()
    },

    end () {
      treeManagement()
      closeElement()
    },

    chars (text: string) {
      handleText()
      createChildrenASTOfText()
    },
    comment (text: string) {
      createChildrenASTOfComment()
    }
  })
  return astRootElement
}
```



#### 从options获取方法和配置

对应的伪代码：

```typescript
getFnsAndConfigFromOptions(options)
```

`parse`函数的输入的是`template`和`options`，输出的是`AST`的根节点，这里的`template`是我们的模板字符串，而传入了`options`，实际上是和平台相关的一些配置，这个配置定义在

> ​	src/plaforms/web/compiler/options

```typescript
import {
  isPreTag,
  mustUseProp,
  isReservedTag,
  getTagNamespace
} from '../util/index'

import modules from './modules/index'
import directives from './directives/index'
import { genStaticKeys } from 'shared/util'
import { isUnaryTag, canBeLeftOpenTag } from './util'

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

这个方法放到`plaforms`目录下面，是因为在不同的平台`(web和weex)`的实现是不相同的。

接下来我们看看这段伪代码的实现

```typescript
  warn = options.warn || baseWarn

  platformIsPreTag = options.isPreTag || no
  platformMustUseProp = options.mustUseProp || no
  platformGetTagNamespace = options.getTagNamespace || no
  const isReservedTag = options.isReservedTag || no
  maybeComponent = (el: ASTElement) => !!el.component || !isReservedTag(el.tag)

  transforms = pluckModuleFunction(options.modules, 'transformNode')
  preTransforms = pluckModuleFunction(options.modules, 'preTransformNode')
  postTransforms = pluckModuleFunction(options.modules, 'postTransformNode')

  delimiters = options.delimiters
```



#### 解析html模板

对应伪代码

```typescript
parseHTML(template,options)
```

对于`template`模板，主要是通过`parseHTML`函数进行解析的，该函数定义在：

> src/compiler/parser/html-parser.js

```typescript
export function parseHTML (html, options) {
  let lastTag
  while (html) {
    if (!lastTag || !isPlainTextElement(lastTag)){
      let textEnd = html.indexOf('<')
      if (textEnd === 0) {
         if(matchComment) {
           advance(commentLength)
           continue
         }
         if(matchDoctype) {
           advance(doctypeLength)
           continue
         }
         if(matchEndTag) {
           advance(endTagLength)
           parseEndTag()
           continue
         }
         if(matchStartTag) {
           parseStartTag()
           handleStartTag()
           continue
         }
      }
      handleText()
      advance(textLength)
    } else {
       handlePlainTextElement()
       parseEndTag()
    }
  }
}
```

这个解析过程的代码也是非常的长，这里还是采用伪代码的的心事来进行处理。总的来说，`parseHTML`的逻辑就是	循环解析`template`，用各种正则来进行匹配，对于不同的情况做不同的处理，直达整个`template`被解析完毕，在匹配的过程中，会利用`advance`函数不断前进整个模板字符串，直到字符串末尾。

```typescript
function advance (n) {
    index += n
    html = html.substring(n)
  }
```

可以举一个粒子：刚开始`index`在`<div>`标签的前面

```vue
`idnex`<div class='test'><span>粒子</span></div>
```

```js
advance(5)
```

注意，这里index移动5位，那么上面就变成了

```vue
<div `index` class='test'><span>粒子</span></div>
```

在这个匹配的过程中，主要利用了下面几种正则匹配

```typescript
// Regular Expressions for parsing tags and attributes
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const dynamicArgAttribute = /^\s*((?:v-[\w-]+:|@|:|#)\[[^=]+\][^\s"'<>\/=]*)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const ncname = `[a-zA-Z_][\\-\\.0-9_a-zA-Z${unicodeRegExp.source}]*`
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
// 开始标签部分，不包含开始标签的结尾。如 <div class="className" ></div>，匹配的是 '<div class="className"'
const startTagOpen = new RegExp(`^<${qnameCapture}`) 
// 开始标签的结尾部分。如 <div class="className" ></div>，匹配的是 ' >'
const startTagClose = /^\s*(\/?)>/
// '</div><p></p>' 匹配结果为 </div>
const endTag = new RegExp(`^<\\/${qnameCapture}[^>]*>`)
// 匹配 DOCTYPE
const doctype = /^<!DOCTYPE [^>]+>/i
// #7298: escape - to avoid being passed as HTML comment when inlined in page
// 注释匹配
const comment = /^<!\--/
// 匹配条件注释
const conditionalComment = /^<!\[/
```

通过这些正则表达式，我们可以匹配到注释节点、文档类型节点、开始闭合标签等。

- 注释节点、文档类型节点

对于注释、文档类型节点来说、不需要做任何处理，如果匹配到了，只需要做的就是`index`前进即可

```typescript
// Comment:
if (comment.test(html)) {
  const commentEnd = html.indexOf('-->')

  if (commentEnd >= 0) {
    if (options.shouldKeepComment) {
      options.comment(html.substring(4, commentEnd))
    }
    advance(commentEnd + 3)
    continue
  }
}
//http://en.wikipedia.org/wiki/Conditional_comment#Downlevel_revealed_conditional_comment
if (conditionalComment.test(html)) {
  const conditionalEnd = html.indexOf(']>')

  if (conditionalEnd >= 0) {
    advance(conditionalEnd + 2)
    continue
  }
}
// Doctype:
const doctypeMatch = html.match(doctype)
if (doctypeMatch) {
  advance(doctypeMatch[0].length)
  continue
}
```

对于注释节点和条件注释节点，`index`只需要前进到标签的末尾的位置就可以了，而文档类型节点，前进自身的长度就可以了。



- 开始节点

```typescript
// Start tag:
const startTagMatch = parseStartTag()
if (startTagMatch) {
  handleStartTag(startTagMatch)
  if (shouldIgnoreFirstNewline(startTagMatch.tagName, html)) {
    advance(1)
  }
  continue
}
```

首先通过`parseStartTag()`函数来解析标签，来看看`parseStartTag()`的实现

```typescript
function parseStartTag () {
    // 匹配到开始的标签
    const start = html.match(startTagOpen)
    if (start) {
      // 定义match对象  
      const match = {
        tagName: start[1],
        attrs: [],
        start: index
      }
      // 移动开始标签的长度
      advance(start[0].length)
      let end, attr
      // 循环匹配开始标签中的属性
      // 标签结尾不存在 && 动态的attr || attr属性存在
      // 一直匹配到结束标签闭合为止
      while (!(end = html.match(startTagClose)) && (attr = html.match(dynamicArgAttribute) || html.match(attribute))) {
        attr.start = index
        advance(attr[0].length)
        attr.end = index
        // 将匹配到开始标签中的属性添加到match.attrs中  
        match.attrs.push(attr)
      }
      // 匹配到标签结尾符  
      if (end) {
        // 获取标签结尾符的`/`  
        match.unarySlash = end[1]
        // 前进到标签结尾符  
        advance(end[0].length)
        // 结束的索引值，赋值为match.end
        match.end = index
        // 返回match对象  
        return match
      }
    }
  }
```

执行完`parseStartTag`后，会判断返回的`match`是否存在，如果存在，接着就会执行`handleStartTag`对`match`做处理，接下来来看`handleStartTag`函数的实现和对`match`做了什么处理。

```typescript
function handleStartTag (match) {
    // 获取开始标签的tagName
    const tagName = match.tagName
    // 获取一元斜杠符(/)
    const unarySlash = match.unarySlash
	
    // 在web编译的情况下为true，其余情况为false
    if (expectHTML) {  
      if (lastTag === 'p' && isNonPhrasingTag(tagName)) {
        parseEndTag(lastTag)
      }
      if (canBeLeftOpenTag(tagName) && lastTag === tagName) {
        parseEndTag(tagName)
      }
    }
	
    // 判断当前标签是否为一元标签，例如：img、input等
    const unary = isUnaryTag(tagName) || !!unarySlash
	
    // 获取attrs的长度，生成l长度的空数组
    const l = match.attrs.length
    const attrs = new Array(l)
    // 遍历attrs，从新构造attrs属性
    for (let i = 0; i < l; i++) {
      // 记录标签中的属性  
      const args = match.attrs[i]
      const value = args[3] || args[4] || args[5] || ''
      // 处理属性中需要被转义、解码的属性
      const shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
        ? options.shouldDecodeNewlinesForHref
        : options.shouldDecodeNewlines
      attrs[i] = {
        name: args[1],
        value: decodeAttr(value, shouldDecodeNewlines)
      }
      if (process.env.NODE_ENV !== 'production' && options.outputSourceRange) {
        attrs[i].start = args.start + args[0].match(/^\s*/).length
        attrs[i].end = args.end
      }
    }
	
    // 如果不是一元标签，就将标签放到栈里面
    if (!unary) {
      stack.push({ tag: tagName, lowerCasedTag: tagName.toLowerCase(), attrs: attrs, start: match.start, end: match.end })
      lastTag = tagName
    }
	
    // 如果传入了start
    if (options.start) {
      // 就执行start方法  
      options.start(tagName, attrs, unary, match.start, match.end)
    }
  }
```

`handleStartTag`的逻辑比较简单，先是判断了`tagName`是否一元标签，例如:`img、br、input`等，接下来对`match.attrs`做循环遍历，并做了一些处理，最后判断如果不是一元标签，就往`stack`里面push一个对象，并且把`tagName`赋值给`lastTag`。最后调用`start`回调函数。



- 闭合标签

```typescript
// End tag:
const endTagMatch = html.match(endTag)
if (endTagMatch) {
  const curIndex = index
  advance(endTagMatch[0].length)
  parseEndTag(endTagMatch[1], curIndex, index)
  continue
}
```

这里通过正则`endTag`匹配到结束标签，然后前进到闭合标签末尾处，在然后通过`parseEndTag`方法对闭合标签做处理。

```typescript
 function parseEndTag (tagName, start, end) {
    let pos, lowerCasedTagName
    if (start == null) start = index
    if (end == null) end = index

    // Find the closest opened tag of the same type
   	// 如果闭合标签存在 
    if (tagName) {
      // 将闭合标签转换为小写  
      lowerCasedTagName = tagName.toLowerCase()
      for (pos = stack.length - 1; pos >= 0; pos--) {
        // 开始标签和闭合标签是否相等  
        if (stack[pos].lowerCasedTag === lowerCasedTagName) {
          break
        }
      }
    } else {
      // If no tag name is provided, clean shop
      // 如果没有闭合标签，清空  
      pos = 0
    }

    if (pos >= 0) {
      // Close all the open elements, up the stack
      for (let i = stack.length - 1; i >= pos; i--) {
        if (process.env.NODE_ENV !== 'production' &&
          (i > pos || !tagName) &&
          options.warn
        ) {
          options.warn(
            `tag <${stack[i].tag}> has no matching end tag.`,
            { start: stack[i].start, end: stack[i].end }
          )
        }
        if (options.end) {
          options.end(stack[i].tag, start, end)
        }
      }

      // Remove the open elements from the stack
      // 把开始的标签从栈中移除  
      stack.length = pos
      lastTag = pos && stack[pos - 1].tag
    } else if (lowerCasedTagName === 'br') {
      if (options.start) {
        options.start(tagName, [], true, start, end)
      }
    } else if (lowerCasedTagName === 'p') {
      if (options.start) {
        options.start(tagName, [], false, start, end)
      }
      if (options.end) {
        options.end(tagName, start, end)
      }
    }
  }
```

上面通过`handleStartTag`处理开始标签，在处理非一元标签（有endTag）的时候，我们是顺序将`tag`的相关信息给push到`stack`里面，那么在处理闭合标签的时候，那就应该是倒叙处理`stack`，找到第一个和当前`endTag`匹配的标签，如果是正常的标签匹配，那么`stack`的最后一个元素应该和当前的`endTag`匹配，但是有可能在我们开发的粗心的时候，会出现下面这么一类是的情况：

```html
<div><span></div>
```

这个时候当`endTag`为`</div>`的时候，从`stack`尾部找到的标签是`<span>`，就匹配不上，在这个是就会报错警告，匹配后会把栈到`pos`位置弹出，并重`stack`尾部拿到`lastTag`。

最后调用了`end`方法，并传入了一些参数。