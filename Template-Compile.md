# 模版编译原理

上一章我们知道了Vue利用虚拟DOM来进行对比然后精准操作DOM，其中虚拟DOM是利用vnode来进行对比，我们这章介绍的就是如何生成vnode。我们可以利用template，手写渲染函数和JSX三种方法创建HTML。我们这张主要介绍的就是模版如何通过编译，变成渲染函数的。然后每一次state更新，就能够通过渲染函数生成新的vnode。

## 1、模版编译

### 1.1 模版编译是什么

模版是Vue中非常大的一个特点，在template中可以直接写类似于HTML的语法，然后可以直接在其中添加变量。Vue框架可以将模版编译成渲染函数。

### 1.2 模版编译成渲染函数

模版编译分三个部分：

* 将模版解析为AST
* 遍历AST标记静态节点
* 使用AST生成渲染函数

这三部分内容在模版编译中分别对应三个模块：

* 解析器
* 优化器
* 代码生成器

接下来我们就来分别详细介绍一下这三个模块是如何实现的。

## 2. 解析器

解析器的目的就是将模版变成AST，在解析器内部分成多个小解析器，其中包括过滤器解析器、文本解析器和HTML解析器。通过一条主线将这些解析器组装在一起。当模版解析过程中遇到过滤器，就利用过滤器解析器，遇到文本就用文本解析器。我们在后面的章节再介绍过滤器解析器，这里只介绍文本解析器和HTML解析器。其中文本解析器解析带有变量的模版，例如：`Hello {{ name }}`，如果是纯文本的不需要文本解析器。

### 2.1 解析器的作用

解析器的作用就是将模版转换成AST，例如：

``` javascript
<div>
  <p>{{name}}</p>
</div>
```

转换成AST是：

```javascript
{
  tag: "div",
  type: 1,
  staticRoot: false,
  static: false,
  plain: true,
  parent: undefined,
  attrsList: [],
  attrsMap: {},
  children: [
    {
      tag: "p",
      type: 1,
      staticRoot: false,
      static: false,
      plain: true,
      parent: {tag: "div", ...},
      attrsList: [],
      attrsMap: {},
      children: [{
        type: 2,
        text: "{{name}}",
        static: false,
        expression: "_s(name)"
      }]
    }
  ]
}
```

type属性表示一个节点的类型，parent属性保存了父节点的描述对象，children是一个数组，保存子节点的描述对象。这样就利用parent和children连在一起，形成了一棵树，这个树就是AST。

### 2.2 解析器内部运行原理

解析器中最重要的就是HTML解析器，我们就用HTML解析器来讲解原理。解析HTML的过程中会不断触发各种钩子函数，这些钩子函数包括开始标签钩子函数、结束标签钩子函数、文本钩子函数以及注释钩子函数。
伪代码：

```javascript
parseHTML (template, {
  start (tag, attrs, unary) {
    // 每当解析到标签的开始标签时，触发该函数
  },
  end () {
    // 每当解析到标签的结束位置时，触发该函数
  },
  chars (text) {
    // 每当解析到文本时，触发该函数
  },
  comment (text) {
    // 每当解析到注释时，触发该函数
  }
})
```

在start钩子函数中构建元素类型节点，chars函数中构建文本类型节点，comment钩子函数中构建注释类型的节点。当解析器不再触发钩子函数时，就说明解析完毕了。

start钩子函数有三个参数，tag、attrs和unary分别说明标签名、标签属性以及是否是自闭合标签。而文本节点和注释节点只需要text即可。
自闭合函数： `<input type="text"/>`，`<div></div>`就不是自闭合函数。
start钩子函数的代码：

```javascript
function createASTElement (tag, attrs, parent) {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    parent,
    children: []
  }
}

parseHTML (template, {
  start (tag, attrs, unary) {
    let element = createASTElement(tag, attrs, currentParent)
  }
})
```

上面的代码中，我们在start钩子函数中构建了一个元素类型的AST节点。

chars钩子函数的代码：

```javascript
parseHTML(template, {
  chars (text) {
    let element = {type: 3, text}
  }
})
```

comment钩子函数的代码：

```javascript
parseHTML(template, {
  comment (text) {
    let element = {type: 3, text, isComment: true}
  }
})
```

AST还需要有层级关系，但是目前这种从左向右解析无法有层级结构，我们需要维护一个栈，每当遇到开始标签的时就会触发start钩子函数，start钩子函数会把当前构建的节点推入栈中，遇到结束标签的时候，就会触发end钩子函数，然后end钩子函数会从栈中弹出一个节点。这样就可以保证每当触发start时，栈的最后一个节点就是当前构建节点的父节点。
假设有这样一个模版：

```javascript
<div>
  <h1>我是Berwin</h1>
  <p>我今年23岁</p>
</div>
```

上面这个模版被解析成AST的过程如图：
图9-2

* 1 模版的开始位置是div的开始标签，于是会触发钩子函数start。start触发后，会先构建一个div节点。此时发现栈是空的，这说明div节点是根节点，因为它没有父节点。最后，将div节点推入栈中，并将模版字符串中的div开始标签从模版中截取掉。
* 2 这是模版的开始位置是一些空格，这些空格会触发文本节点的钩子函数，在钩子函数里会忽略这些空格。同时会在模版中将这些空格截取掉。
* 3 这是模版的开始位置是h1的开始标签，于是会触发钩子函数start。与前面流程一样，start触发后，会先构建一个h1节点。此时发现栈的最后一个节点是div节点，这说明h1节点的父节点是div，于是将h1添加到div的子节点中，并且将h1节点推入栈中，同时从模版中将h1的开始标签截取掉。
* 4 这是模版的开始位置是一段文本，于是会触发钩子函数chars.chars触发后，会先构建一个文本节点，此时发现栈中的最后一个节点是h1，这说明文本节点的父节点是h1，于是将文本节点添加到h1节点的子节点中。由于文本节点没有子节点，所以文本节点不回被推入栈中。最后，将文本从模版中截取掉。
* 5 这是模版的开始位置是h1结束标签，于是会触发钩子函数end。end触发后，会把栈中最后一个节点弹出来。
* 6 与第二步一样，这是模版的开始位置是一些空格，这些空格会触发文本节点的钩子函数，在钩子函数里会忽略这些空格。同时会在模版中将这些空格截取掉。
* 7 这时模版的开始位置是p开始标签，于是会触发钩子函数start。start触发后，会先构建一个p节点。由于第五步已经从栈中弹出了一个节点，所以此时栈中的最后一个节点是div，这说明p节点的父节点是div。于是将p推入div的子节点中，最后将p推入到栈中，并将p的开始标签从模版中截取掉。
* 8 这时模版的开始位置又是一段文本，于是会触发钩子函数chars。当chars触发后，会先构建一个文本节点，此时发现栈中的最后一个节点是p节点，这说明文本节点的父节点是p节点。于是将文本节点推入p节点的子节点中，并将文本从模版中截取掉。
* 9 这是模版的开始位置是p的结束标签，于是会触发钩子函数end。当end触发后，会从栈中弹出一个节点出来，也就是把p标签从栈中弹出来，并将p的结束标签从模版中截取掉。
* 10 与第二步和第六步一样，这是模版的开始位置是一些空格，这些空格会触发文本节点的钩子函数并且在钩子函数里会忽略这些空格。同时会在模版中将这些空格截取掉。
* 11 这是模版的开始位置是div的结束标签，于是会触发钩子函数end。期逻辑与之前一样，把栈中的最后一个节点弹出来，也就是把div弹了出来，并将div的结束标签从模版中截取掉。
* 12 这是模版已经被截取空了，也就说明HTML解析器已经运行完毕。这是我们会发现栈已经空了，但是我们得到了一个俄完整的带层级关系的AST语法叔。这个AST中清洗血命了每个节点的父节点、子节点及其节点类型。

### 2.3 HTML解析器

HTML解析器可以截取以下几种类型的片段：

* 开始标签，例如`<div>`
* 结束标签，例如`</div>`
* HTML注释，例如`<!-- 我是注释 -->`
* DOCTYPE，例如`<!DOCTYPE html>`
* 条件注释，例如`<!--[if !IE]>-->我是注释<!--<![endif]-->`
* 文本，例如 我是注释
最常见的是开始标签、结束标签、文本以及注释。

然后解析的过程就是循环html模版字符串，然后调用HTML解析器进行截取片段，然后删除模版字符串中的片段，最后把模版字符串都截取完循环就停了。

那我们接下来看一下如何截取开始标签

#### 2.3.1 截取开始标签

截取开始标签是利用正则表达式，代码如下：

```javascript
const ncname = '[a-zA-Z][\\w\\-\\.]*'
const qnameCapture = '((?:${ncname}\\:)?${ncname})'
const startTagOpen = new RegExp(`^<${qnameCapture}`)

// 以开始标签开始的模版
'<div></div>'.match(startTagOpen) // ["<div", "div", index: 0, input: "<div></div>"]

// 以结束标签开始的模版
'</div><div>'.match(startTagOpen) // null

// 以文本开始的模版
'我是Berwin</p>'.match(startTagOpen) // null
```

当匹配之后，可以发现只匹配了`<div`，并不是完整的开始标签，但足以分辨出这是一个开始标签。这么做的目的是为了解析标签属性和自闭合标识。
开始标签可以被拆为三部分，标签名、属性和结尾，如图所示：
图 9-3

#### 2.3.1.1 解析标签属性

因为标签属性很多时候不止一个，例如：`<div class="box" id="el></div>`,那我们就利用标签属性正则表达式去循环截取，直到剩余的模版不存在符合正则表达式的规则了。代码如下:

```javascript
const startTagClose = /^\s*(\/?)>/
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
let html = ' class="box" id="el"></div>'
let end, attr
const match = {tagName: 'div', attrs: []}

while (!(end = html.match(startTagClose)) && (attr = html.match(attribute))) {
  html = html.substring(attr[0].length)
  match.attrs.push(attr)
}
```

这样循环解析与截取后的结果为：

```javascript
{
  tagName: 'div',
  attrs: [
    [' class="box"', 'class', '=', 'box', null, null],
    [' id="el"', 'id', '=', 'el', null, null]
  ]
}
```

剩余的模版字符串变成`"></div>"`

#### 2.3.1.2 解析自闭合标识

解析自闭合标识的代码：

```javascript
function parseStartTagEnd (html) {
  const startTagClose = /^\s*(\/?)>/
  const end = html.match(startTagClose)
  const match = {}

  if (end) {
    match.unarySlash = end[1]
    html = html.substring(end[0].length)
    return match
  }
}

console.log(parseStartTagEnd('></div>'))        // {unarySlash: ""}
console.log(parseStartTagEnd('/><div></div>'))  // {unarySlash: "/"}
```

可以看出，自闭合标签解析后的unarySlash为"/"。

### 2.3.2 截取结束标签

截取结束标签和截取开始标签一样，用正则表达式去匹配，然后直接把模版删除掉结束标签即可。
代码如下：

```javascript
const ncname = '[a-zA-Z_][\\w\\-\\.]*'
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const endTag = new RegExp(`^<\\/${qnameCapture}[^>]*>`)

const endTagMatch = '</div>'.match(endTag)
const endTagMatch2 = '<div>'.match(endTag)

console.log(endTagMatch)  // ["</div>", "div", index: 0, input: "</div>"]
console.log(endTagMatch2) // null
```
截取注释、截取条件注释以及截取DOCTYPE都是一样的道理，只是正则表达式不同而已，就不再讨论了。

### 2.3.3 截取文本

截取文本的操作比上面的正则表达式匹配都要简单，只要开头不是'<'，我们就认为这是一个文本。我们再利用indexOf找到下一个'<'的位置在哪里，中间这段就是文本了。但有种特殊的情况，如果'<'是文本其中一部分怎么办？例如：'1<2</div>'
这种情况我们利用循环，截取了第一个'<'以后，模版就变成了`<2<div>`,我们循环利用其他的正则表达式去判断，不是开始标签，不是结束标签，不是注释标签，都不符合的时候，我们就判断'<'是文本的一部分，那就再找下一个'<'，直到剩余部分能被其他标签截取。

### 2.3.4 纯文本内容元素的处理

这里的纯文本内容元素指的不是上面所说的文本，而是指script、style和textarea这三种元素，这些元素和其他元素处理的逻辑不一样。我们在循环模版的逻辑的外层再套一层逻辑判断是否是这三种元素。

```javascript
while (html) {
  if (!lastTag || !isPlainTextElement(lastTag)) {
    // 父元素为正常元素的处理逻辑
  } else {
    // 父元素为script、style、textarea的处理逻辑
  }
}
```

如果父元素不存在或者不是纯文本内容元素，那么进行正常的处理。
纯文本内容元素的处理逻辑很简单，例如：

```javascript
<div id="el">
  <script>console.log(1)</script>
</div>
```

当解析到script时候，模版就是:

```javascript
console.log(1)</script>
</div>
```

此时父元素为script，进入else中的循环，调用钩子函数chars和end。

最后剩下`</div>`,就不需要走正常元素逻辑去循环调用开始标签，结束标签等。

## 2.4 文本解析器

文本是在HTML解析器中就被解析了，这里的文本解析器是用来解析带有变量的文本。
利用文本解析器解析代码如下：

```javascript
parseHTML(template, {
  start (tag, attrs, unary) {
    // 每当解析到标签的开始位置时，触发该函数
  },
  end () {
    // 每当解析到标签的结束位置时，触发该函数
  },
  chars (text) {
    text = text.trim()
    if (text) {
      const children = currentParent.children
      let expression
      if (expression = parseText(text)) {
        children.push({
          type: 2,
          expression,
          text
        })
      } else {
        children.push({
          type: 3,
          text
        })
      }
    }
  },
  comment (text) {
    // 每当解析到注释是，触发该函数
  }
})
```

如果parseText执行后有结果，那么说明文本是带变量的文本并且已经通过parseText二次加工，此时就构建一个带变量的文本类型的AST并将其添加到父节点的children属性中。
一个text文本如下：
`Hello {{name}}`
解析后的expression变成：
`Hello "+_s(name)"`
上面代码中的_s是一个toString函数的别名。举个例子：

```javascript
var obj = {name: 'Berwin'}
with (obj) {
  function toString (val) {
    return val == null
    ? ''
    : typeof val === 'object'
      ? JSON.stringify(val, null, 2)
      : String(val)
  }
  console.log("Hello "+toString(name)) // "Hello Berwin"
}
```

最终AST会转换成代码字符串放在with中执行，后面代码生成器会讲。

接着我们来看文本解析器如果实现，第一步利用正则表达式去匹配文本中是否包含{{xxx}}这样的语法，如果没有就返回undefined，如果有就再进行二次加工。二次加工也非常简单，用正则匹配出文本中的变量，然后把变量左边的文本添加到数组中，再把变量替换成_s(x)，这样的形式，也push进数组，然后再重复上面的动作，直到左右文本都添加完。最后利用join以下数组就可以了。

# 3、优化器

优化器的目的标记静态子树，静态子树指的是那些在AST中永远都不会发生变化的点。例如，一个纯文本节点就是一个静态子树，而带变量的文本节点就不是静态子树。
标记静态子树有两点好处：
* 每次重新渲染时，不需要为静态子树创建新节点。
* 在虚拟DOM中打补丁（patching）的过程可以跳过。

每次渲染都需要利用渲染函数生成一个新的vnode，如果一个节点被标记为静态子树，那就除了首次渲染之外，重新渲染不会重新生成新的子节点树，而是克隆已存在的静态子树。

如果两个节点都是静态子树，那么也不需要进行对比，就不需要patching过程。

优化器的内部实现主要分为两个步骤：
* 在AST中找出所有静态节点并打上标记
* 在AST中找出所有静态根节点并打上标记

## 3.1 找出所有静态节点并标记

我们从跟节点开始判断是不是静态节点，然后一次利用递归处理所有节点。先使用isStatic函数来判断是否为静态节点，如果节点的类型等于1说明节点是元素节点，那么循环该节点的子节点，调用markStatic函数用同样的处理逻辑来处理子节点。只有所有的子节点都是静态节点，那么这个节点才会是一个静态节点。所以如果有其中一个子节点不是静态节点，那么就把这个节点再改回static为false。

```javascript
function markStatic (node) {
  node.static = isStatic(node)
  if (node.type === 1) {
    for (let i = 0, l = node.children.length; i < l; i++>) {
      const child = node.children[i]
      markStatic(child)
      if (!child.static) {
        node.static = false
      }
    }
  }
}
```

isStatic函数：

```javascript
function isStatic(node) {
  if (node.type === 2) { // 带变量的动态文本节点
    return false
  }
  if (node.type === 3) { // 不带变量的纯文本节点
    return true
  }
  return !!(node.pre || (
    !node.hasBindings && // 没有动态绑定
    !node.if && !node.for && // 没有 v-if 或 v-for 或 v-else
    !isBuildtInTag(node.tag) && // 不是内置标签
    isPlatformReservedTag(node.tag) && // 不是组件
    !isDirectChildOfTemplateFor(node) && 
    Object.keys(node).every(isStaticKey)
  ))
}
```
不同元素类型设置不同的type值：

| type的值 | 说明 |
| ------ | ------ |
| 1 | 元素节点 |
| 2 | 带变量的动态文本节点 |
| 3 | 不带变量的纯文本节点 |

必须同时满足以下条件才会被认为是一个静态基点：
* 不能使用动态绑定语法，也就是标签上不能有以v-、@、：开头的属性。
* 不能使用v-if、v-for或者v-else指令。
* 不能是内置标签，不能是slot或者component。
* 不能是组建，即标签名必须是保留标签，例如<div></div>是保留标签，而<list></list>不是保留标签。
* 当前节点的父节点不能是带v-for指令的template标签。
* 节点中不存在动态节点才会有的属性。

## 3.2 找出所有静态根节点并标记

当标记完成所有静态节点以后，我们再重新开始，标记静态根节点。依然利用递归，当我们碰到的第一个节点，我们就把它标记为静态跟节点，不需要再向下找它的子节点了。因为这个点如果是静态节点了，那么它的子节点也全都是静态节点，它就是一个静态根节点。

# 4、代码生成器

## 4.1 通过AST生成代码字符串
代码生成器通过递归，从顶向下处理每一个AST节点，然后根据同的节点，创建别名，然后把字符串拼接在一起。

| 类型 | 创建方法 | 别名 |
| ------ | ------ | ------ |
| 元素节点 | createElement | _c |
| 文本节点 | createTextNode | _v |
| 注释节点 | createEmptyNode | _e |
例如： 

```javascript
<div id="el">
  <div>
    <p>Hello {{name}}</p>
  </div>
</div>
```
从最外面的div开始生成代码字符串，最开始是 `_c("div", {attrs: {"id":"el"}})`,然后下一个AST之后，就变成了`_c("div", {attrs: {"id":"el"}},[_c('div')])`,
最后变成`_c("div", {attrs: {"id":"el"}},[_c('div',[_c('p',[_v("Hello"+_s(name))])])])`
