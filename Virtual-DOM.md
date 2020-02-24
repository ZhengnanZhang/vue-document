# Virtual DOM

我们这里所要讲的Virtual DOM指的是在利用render函数生成了vnode之后，进行对比进而利用patch函数生成真正DOM元素的过程。
在React和Vue中都引入了Virtual DOM，在Vue中引用的是snabbdom库来实现的虚拟DOM。

## Virtual DOM是什么

虚拟DOM指示利用JS的对象这个数据结构来描述一个真实的DOM，这样做的目的大家也都十分了解，就是因为操作真实的DOM非常的费时，昂贵，但是JS的操作由于V8引擎，所以非常的快。引入虚拟DOM的目的就是利用JS先比对出需要进行操作的DOM，然后进行精准的一次性修改真实DOM，避免多余的DOM操作，从而来提高效率。所以，这个用来描述真实DOM的JS对象，就是虚拟DOM，在Vue中命名为vnode。

* 虚拟DOM使用的场景举例：

  > 假设一个ul下面有1000个li，其中一个li的text属性变化了，如果没有框架帮我们通过Virtual DOM精准的找到需要改变的地方，我们需要将1000个li都删除，然后重新添加1000个新的li。这样就十分的费时。

我们上面举了一个例子来说明为什么引入虚拟DOM就会比操作真实DOM快。

那么下面我们就开始来按照顺序来介绍整个虚拟DOM的过程。

## 1、VNode

我们现在已经理解了什么是虚拟DOM，就是比对Vnode树，然后找到不同，再根据不同来操作真实的DOM，整个这个过程叫做虚拟DOM。那我们首先来了解Vnode

### 1.1 什么是Vnode

Vnode是Vue中用JS的对象来描述一个真实节点。

下面是Vue中的生成vnode的vnode类的代码：

``` javascript
export default class VNode {
  constructor (tag, data, children, text, elm, context, componentOptions, asyncFactory) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.functionalContext = undefined
    this.functionalOptions = undefined
    this.functionalScopedId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment =false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  get child () {
    return this.componentInstance
  }
}
```

vnode中的一些属性，我们会在接下来的内容中进行介绍

### 1.2 什么是Vnode

Vnode的主要作用就是能够进行新旧vnode的对比，找到不同点。

首先我们这里描述的是框架已经帮我们通过render函数得到了vnode，也就是已经生成好了整个页面的所有真实节点都转换成了vnode，进而形成了Vnode树，至于是如何生成的vnode那我们在下一章再去介绍，现在你只要知道我们已经根据最新的状态（state）生成了一棵vnode树。由于vnode其实是JS的对象，所以我们是可以缓存旧的vnode树，当新生成一棵vnode树的时候，我们只需要和旧的vnode树进行对比，找到不同的地方然后再操作DOM就可以。

Vue现在采用的侦测策略是中等粒度，也就是说当状态发生改变的时候，只通知到组件，然后组件内使用Virtual DOM来渲染试图，其他的组件不受干扰。这样做是为了提高效率，避免整个页面的vnode都进行对比，如果侦测策略太细，又会保存的watcher太多，导致占用内存。

### 1.3 VNode的类型

Vnode分为以下几种：

* 1 注释节点
* 2 文本节点
* 3 元素节点
* 4 组件节点
* 5 函数式节点
* 6 克隆节点

那我们接下来就详细介绍一下每一种节点。

#### 1.3.1 注释节点

源代码：

``` javascript
export const createEmptyVNode = text => {
  const node = new VNode()
  node.text = text
  node.isComment = true
  return node
}
```

只有text和isComment两个属性有效，其余都是默认的undefined或者false

真实的DOM：
`<!-- 注释节点 -->`

生成的vnode如下：

```javascript
{
text: “注释节点”,
isComment: true
}
```

#### 1.3.2 文本节点

源代码：

```javascript
export function createTextVNode (val) {
  return new VNode(undefined, undefined, undefined, String(val))
}
```

也非常简单，比注释节点还要简单。

生成的vnode如下：

```javascript
{
  text: “Hello Berwin”
}
```

#### 1.3.3 元素节点

元素节点通常有4种有效属性：

* 1 tag：节点名称，例如p、ul、li和div等,
* 2 data： 节点上的属性数据，例如class和style等,
* 3 children： 当前节点的字节点列表,
* 4 context： 当前组建的Vue.js实例。

例如，一个真实的元素节点：
`<p><span>Hello</span><span>Berwin</span></p>`
对应的Vnode如下：

```javascript
{
  tag: p,
  data: {…},
  children: [VNode, VNode],
  context: {…}
}
```

#### 1.3.4 组件节点

组件节点和元素节点类似，但有两个独有的属性：

* componentOptions：组件节点的选项参数，其中包括propsData，tag和children等信息
* componentInstance： 组件的实例，也是Vue.js的实例

一个组件节点：
`<child></child>`

对应的vnode:

```javascript
{  
  tag: “vue-component-1-child”,
  data: {…},
  context: {…},
  componentOptions: {…},
  componentInstance: {…}
}
```

#### 1.3.5 函数式组件

函数式组件和组件节点类似，有两个独有的属性 functionalContext 和 functionalOptions

一个函数式组建的vnode如下：

```javascript
{
  tag:  “div”,
  data: {…},
  functionalContext: {…},
  functionalOptions: {…}
}
```

#### 1.3.6 克隆节点

克隆节点是将现有节点的属性复制到新节点中，让新创建的节点和被克隆节点的属性保持一致，从而实现克隆效果,优化静态节点。

当组件内的某个状态发生变化后，当前组件会通过虚拟DOM重新渲染试图，静态节点因为它的内容不会改变，所以除了首次渲染需要执行渲染函数获取vnode之外，后续更新不需要执行渲染函数重新生成vnode。因此，这时就会使用创建克隆节点的方法将vnode克隆一份，使用克隆节点进行渲染。这样就不需要重新执行渲染函数生成新的静态节点的vnode，从而提升异一定程度的性能。

克隆节点的代码：

```javascript
export function cloneVNode(vnode, deep) {
  const cloned = new VNode(
    vnode.tag,
    vnode.data,
    vnode.children,
    vnode.text,
    vnode.elm,
    vnode.context,
    vnode.componentOptions,
    vnode.asyncFactory
  )
  clone.ns = vnode.ns
  cloned.isStatic = vnode.isStatic
  clone.key = vnode.key
  cloned.isComment = vnode.isComment
  cloned.isCloned = true
  if (deep && vnode.children) {
    cloned.children = cloneVNodes(vnode.children)
  }
  return cloned
}
```

克隆节点只需要将现有节点的属性全部复制到新节点，然后将isCloned属性设置为true即可。

## 2、Patch

Patch算法是虚拟DOM中最核心的部分，它可以将vnode渲染成真实的DOM。而patch的过程，无非就是三种：

* 创建新增的节点
* 删除已经废弃的节点
* 修改需要更新的节点

### 2.1 新增节点

有两种情况会新增节点，第一种就是首次渲染的时候，因为oldVnode不存在，所以直接用vnode创建元素并渲染视图。另一种就是当vnode中有节点，但是oldVnode里面没有，那就需要创建节点。

### 2.2 删除节点

当一个节点值只在oldVnode中存在时，就证明vnode里面没有这个节点，那么就把这个这个点从真实DOM中删除掉。

### 2.3 更新节点

更新节点是patch过程中最重要也是相对最复杂的，因为大部分的操作都是更新节点。首先我们先要确认，只有是同一个节点才会进行更新，如果生成新的vnode树，然后和oldVnode不是同一个节点，那么将会删除然后再添加新的节点。如果是同一个节点，那么按照下面的顺序进行比对更新。

此章我们用vnode来代表新生成的虚拟节点，oldVnode代表原有的虚拟节点。

#### 2.3.1 静态节点

如果这新旧两个节点是静态节点，例如：
`<p>我是静态节点，我不需要发生变化</p>`
就是静态节点，也就是不会随着状态的变化而变化的节点。静态节点就不需要进行比对了，不需要更新操作，直接跳过更新过程。

#### 2.3.2 vnode有文本属性

如果vnode有文本属性，那么我们直接调用setTextContent方法将视图中的DOM节点的内容改为vnode的text属性所保存的文字。即使oldVnode有子元素或者文本是什么都不重要，把子元素清空或把文本替换，因为我们以vnode的text属性为准。

#### 2.3.3 vnode无文本属性

##### 2.3.3.1 无children的情况

当vnode既没有text属性，也没有children属性的时候，说明vnode是一个空节点，那么oldVnode中有有什么就删除什么，有节点就删除节点，有text就删除text，最后变成空节点就行。

##### 2.3.3.2 有children的情况

如果vnode有children，那么也有两种情况。如果oldVnode没有children，说明oldVnode要么是一个空节点，要不然就是一个文本节点，那么不需要进行比对，如果是文本节点就将文本删除，将vnode中的children挨个创建成真实的DOM插入到视图中的DOM节点下面。如果oldVnode有children，就是比较复杂的情况了，我们需要对新旧两个虚拟节点的children进行一个详细的对比并更新。更新children可能会移动子节点的位置，删除或者新增某个字节点。我们会在接下来的篇幅中都介绍这种比对。

### 2.4 更新字节点

更新子节点分为四种操作：

* 新增节点
* 更新节点
* 删除节点
* 移动节点

我们更新子节点的时候用newChildren来表示新的子节点数组，用oldChildren来表示旧的子节点数组。

#### 2.4.1 创建子节点

新旧两个子节点列表是通过循环进行对比的，newChildren中的一个节点，在oldChildren中去寻找，如果没有找到，说明这是一个新的节点，需要插入到真实的DOM中，那么就生成一个真实的节点然后插入到oldChildren中所有未处理节点（未处理就是没有进行任何更新操作的节点)的前面。
这里的创建子节点其实也是上面2.1的新增节点，因为这个patch过程也是一个深度优先的过程。

![我是图片](https://raw.githubusercontent.com/ZhengnanZhang/vue-document/master/image/7-11.jpg)

                                                    图片 1

为什么一定要放在oldChildren中所有未处理节点的前面，是因为如果放在oldChildren已处理节点的后面，如果下一个还是新插入的节点，那么顺序就会错。

创建真实节点时，事实上只有三种节点会被创建并插入到DOM中:

* 元素节点
* 注释节点
* 文本节点

如果是元素节点，就会有tag属性，那么利用createElement方法就可以创建，然后再利用appendChild方法就可以将一个元素插入到指定的福节点中。如果元素节点有子节点，那么利用递归将children中的子节点都执行一遍创建元素的逻辑，然后插入到这个节点下面。
如果没有tag属性，那么有可能是注释节点或者文本节点。通过isComment属性就可以判断是注释节点还是文本节点，如果是注释节点就利用createComment方法创建并插入到指定父节点中，如果是文本节点，就利用createTextNode方法创建。
既然已经插入了节点，节点越插入越多，肯定就需要删除不需要的节点，下一部分介绍删除子节点。

### 2.4.2 删除子节点

当newChildren中的所有节点都被循环了一遍后，如果oldChildren中还有剩余的没有处理的节点，那么这些节点就需要被删除。

删除vnodes数组中的节点的代码：

```javascript
function removeVnodes (vnodes, startIdx, endIdx) {
  for (; startIdx <= endIdx; ++startIdx) {
    const ch = vnodes[startIdx]
    if (isDef(ch)) {
      removeNode(ch.elm)
    }
  }
}
```

删除单个节点的代码：

```javascript
const nodeOps = {
  removeChild (node, child) {
    node.removeChild(child)
  }
}

function removeNode (el) {
  const parent = nodeOps.parentNode(el)
  if (isDef(parent)) {
    nodeOps.removeChild(parent, el)
  }
}
```

我们定义一个nodeOps里面封装了操作DOM的一些方法，这样做的目的是为了方便以后的跨平台。

#### 2.4.3  更新子节点

当newChildren和oldChildren中的两个节点是同一个节点并且位置相同时，进行更新节点的操作，按照2.3的更新节点的操作更新子节点。但如果oldChildren中子节点的位置和newChildren中的位置不同时，除了需要更新真实DOM还需要移动真实DOM节点位置。

#### 2.4.4  移动子节点

当oldChildren中找到和newChildren中相同的点，但是位置不同时候，利用Node.insertBefore()方法，就可以把一个已有的节点移动到指定位置。因为newChildren这个列表是从左向右循环的，所以newChildren中当前点的左边都是已经被处理过的点，这个点也就是未处理的节点的第一个，所以把需要移动的节点移动到所有未处理节点的最前面。
![我是图片](https://github.com/ZhengnanZhang/vue-document/tree/master/image/7-16.jpg)

                                                      图 2

### 2.5 优化策略

由于每次都需要循环整个oldChildren中所有节点，根据实际运行，我们也可以进行一定的优化，我们可以建立一个快捷查找的策略：新前与旧前、新后与旧后、新后与旧前、新前与旧后。

* 新前：newChildren中所有未处理的第一个节点，
* 新后：newChildren中所有未处理的最后一个节点，
* 旧前：oldChildren中所有未处理的第一个节点，
* 旧后：oldChildren中所有未处理的最后一个节点。

所以每当从newChildren中循环到一个点，我们都先按照这个快捷查找走一遍，如果没有找到，再进行创建节点的操作。进行快捷查找时，首先判断新前与旧前，如果找到，那就是位置相同也是同一个节点，进行普通的更新操作就行。如果新前与旧前不相同，那么我们接下来执行新后与旧后，同理相同就更新节点，不同的话再执行新后与旧前，如果新后与旧前是同一个节点，由于他们的位置不同，所以除了更新以外还需要移动他们的位置。这里我们移动和之前有些不同，我们要把点移动到oldChildren中未处理的点的最后，原因看下图：
![我是图片](https://github.com/ZhengnanZhang/vue-document/tree/master/image/7-22.jpg)

                                                      图 3

因为更新节点都是以新虚拟节点为基准，所有如图所示，当真实DOM子节点左右两侧都有节点更新了，只有中间这部分的时候，为了最后DOM位置的正确，只有将节点移动到oldChildren最后一个节点的后面才对。如果新后与旧前还是不一致，那么就执行新前与旧后。 新前与旧后也是需要更新然后进行移动节点的，移动的位置就是所有未处理节点的最前面，也就是oldChildren第一个节点的前面，有了上面的解释这个也比较容易理解了。如果这四种比对方式都没有找到相同的节点，这是再通过循环去oldChildren中查找然后进行移动和更新。一般情况下这四种都能找到，免去了循环。

### 2.6 针对快捷查找中未处理的节点的判断

在上面的讨论中我们一直对比的都是未处理的节点，插入的位置也是根据未处理的节点进行定位的，那这里我们就讲以下如果判断哪些节点时未处理的节点。因为我们也是利用循环来在newChildren和oldChildren中进行快捷查找，所有我们定义四个变量oldStartIndex，oldEndIndex，newStartIndex和newEndIndex。分别表示newChildren和oldChildren首尾未处理节点的位置。StartIndex只能向后走，EndIndex只能向前走，处理了一个节点以后，这四个值中的两个肯定更新，最后如果开始位置大于等于结束位置，就结束。如果oldChildren和newChildren有一个遍历完了，就结束整个循环。如果oldChildren中还有节点，那么就循环执行删除节点操作，如果newChildren中还有，就循环执行创建节点操作。StartIndex和EndIndex中间的节点就是剩余的节点。oldChildren中的虚拟节点，每处理一个就会将这个虚拟节点设置成undefined，防止后续重复处理同一个节点。
最后我们说一个key如何用，在模版中渲染列表时，Vue建议我们为节点设置属性key，这样做key与oldChildren中的index索引会建立对应关系，也就是说如果在oldChildren中有相同的节点，直接通过key就能拿到下标，不需要进行快捷查找和循环来查找。
