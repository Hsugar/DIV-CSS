# Vue实现virtual-dom的原理简析



virtual-dom(后文简称vdom)的概念大规模的推广还是得益于react出现，virtual-dom也是react这个框架的非常重要的特性之一。相比于频繁的手动去操作dom而带来性能问题，vdom很好的将dom做了一层映射关系，进而将在我们本需要直接进行dom的一系列操作，映射到了操作vdom，而vdom上定义了关于真实dom的一些关键的信息，vdom完全是用js去实现，和宿主浏览器没有任何联系，此外得益于js的执行速度，将原本需要在真实dom进行的创建节点,删除节点,添加节点等一系列复杂的dom操作全部放到vdom中进行，这样就通过操作vdom来提高直接操作的dom的效率和性能。

Vue在2.0版本也引入了vdom。其vdom算法是基于[snabbdom算法](https://github.com/snabbdom/snabbdom)所做的修改。

在Vue的整个应用生命周期当中，每次需要更新视图的时候便会使用vdom。那么在Vue当中，vdom是如何和Vue这个框架融合在一起工作的呢？以及大家常常提到的vdom的diff算法又是怎样的呢？接下来就通过这篇文章简单的向大家介绍下Vue当中的vdom是如何去工作的。

首先，我们还是来看下Vue生命周期当中初始化的最后阶段：将vm实例挂载到dom上，源码在src/core/instance

```javascript
`Vue.prototype._init = ``function` `() {``  ``...``  ``vm.$mount(vm.$options.el) ``// 实际上是调用了mountComponent方法``  ``...``}  `
```

mountComponent函数的定义是：

```javascript
`export ``function` `mountComponent (`` ``vm: Component,`` ``el: ?Element,`` ``hydrating?: boolean``): Component {`` ``// vm.$el为真实的node`` ``vm.$el = el`` ``// 如果vm上没有挂载render函数`` ``if` `(!vm.$options.render) {``  ``// 空节点``  ``vm.$options.render = createEmptyVNode`` ``}`` ``// 钩子函数`` ``callHook(vm, ``'beforeMount'``)` ` ``let updateComponent`` ``/* istanbul ignore if */`` ``if` `(process.env.NODE_ENV !== ``'production'` `&& config.performance && mark) {``  ``...`` ``} ``else` `{``  ``// updateComponent为监听函数, new Watcher(vm, updateComponent, noop)``  ``updateComponent = () => {``   ``// Vue.prototype._render 渲染函数``   ``// vm._render() 返回一个VNode``   ``// 更新dom``   ``// vm._render()调用render函数，会返回一个VNode，在生成VNode的过程中，会动态计算getter,同时推入到dep里面``   ``vm._update(vm._render(), hydrating)``  ``}`` ``}` ` ``// 新建一个_watcher对象`` ``// vm实例上挂载的_watcher主要是为了更新DOM`` ``// vm/expression/cb`` ``vm._watcher = ``new` `Watcher(vm, updateComponent, noop)`` ``hydrating = ``false` ` ``// manually mounted instance, call mounted on self`` ``// mounted is called for render-created child components in its inserted hook`` ``if` `(vm.$vnode == ``null``) {``  ``vm._isMounted = ``true``  ``callHook(vm, ``'mounted'``)`` ``}`` ``return` `vm``}`
```

注意上面的代码中定义了一个updateComponent函数，这个函数执行的时候内部会调用vm._update(vm._render(), hyddrating)方法，其中vm._render方法会返回一个新的vnode，(关于vm_render是如何生成vnode的建议大家看看vue的关于compile阶段的代码)，然后传入vm._update方法后，就用这个新的vnode和老的vnode进行diff，最后完成dom的更新工作。那么updateComponent都是在什么时候去进行调用呢？

```javascript
`vm._watcher = ``new` `Watcher(vm, updateComponent, noop)`
```

实例化一个watcher，在求值的过程中this.value = this.lazy ? undefined : this.get()，会调用this.get()方法，因此在实例化的过程当中Dep.target会被设为这个watcher，通过调用vm._render()方法生成新的Vnode并进行diff的过程中完成了模板当中变量依赖收集工作。即这个watcher被添加到了在模板当中所绑定变量的依赖当中。一旦model中的响应式的数据发生了变化，这些响应式的数据所维护的dep数组便会调用dep.notify()方法完成所有依赖遍历执行的工作，这里面就包括了视图的更新即updateComponent方法的调用。

updateComponent方法的定义是：

```javascript
`updateComponent = () => {`` ``vm._update(vm._render(), hydrating)``}`
```

完成视图的更新工作事实上就是调用了vm._update方法，这个方法接收的第一个参数是刚生成的Vnode，调用的vm._update方法的定义是

```javascript
`Vue.prototype._update = ``function` `(vnode: VNode, hydrating?: boolean) {``  ``const vm: Component = ``this``  ``if` `(vm._isMounted) {``   ``callHook(vm, ``'beforeUpdate'``)``  ``}``  ``const prevEl = vm.$el``  ``const prevVnode = vm._vnode``  ``const prevActiveInstance = activeInstance``  ``activeInstance = vm``  ``// 新的vnode``  ``vm._vnode = vnode``  ``// Vue.prototype.__patch__ is injected in entry points``  ``// based on the rendering backend used.``  ``// 如果需要diff的prevVnode不存在，那么就用新的vnode创建一个真实dom节点``  ``if` `(!prevVnode) {``   ``// initial render``   ``// 第一个参数为真实的node节点``   ``vm.$el = vm.__patch__(``    ``vm.$el, vnode, hydrating, ``false` `/* removeOnly */``,``    ``vm.$options._parentElm,``    ``vm.$options._refElm``   ``)``  ``} ``else` `{``   ``// updates``   ``// 如果需要diff的prevVnode存在，那么首先对prevVnode和vnode进行diff,并将需要的更新的dom操作已patch的形式打到prevVnode上，并完成真实dom的更新工作``   ``vm.$el = vm.__patch__(prevVnode, vnode)``  ``}``  ``activeInstance = prevActiveInstance``  ``// update __vue__ reference``  ``if` `(prevEl) {``   ``prevEl.__vue__ = ``null``  ``}``  ``if` `(vm.$el) {``   ``vm.$el.__vue__ = vm``  ``}``  ``// if parent is an HOC, update its $el as well``  ``if` `(vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {``   ``vm.$parent.$el = vm.$el``  ``}``}`
```

在这个方法当中最为关键的就是vm.__patch__方法，这也是整个virtaul-dom当中最为核心的方法，主要完成了prevVnode和vnode的diff过程并根据需要操作的vdom节点打patch，最后生成新的真实dom节点并完成视图的更新工作。

接下来就让我们看下vm.__patch__里面到底发生了什么：

```javascript
`function` `patch (oldVnode, vnode, hydrating, removeOnly, parentElm, refElm) {``  ``// 当oldVnode不存在时``  ``if` `(isUndef(oldVnode)) {``    ``// 创建新的节点``    ``createElm(vnode, insertedVnodeQueue, parentElm, refElm)``  ``} ``else` `{``    ``const isRealElement = isDef(oldVnode.nodeType)``    ``if` `(!isRealElement && sameVnode(oldVnode, vnode)) {``    ``// patch existing root node``    ``// 对oldVnode和vnode进行diff，并对oldVnode打patch``    ``patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly)`` ``} ``  ``}``}`
```

在对oldVnode和vnode类型判断中有个sameVnode方法，这个方法决定了是否需要对oldVnode和vnode进行diff及patch的过程。

```javascript
`function` `sameVnode (a, b) {`` ``return` `(``  ``a.key === b.key &&``  ``a.tag === b.tag &&``  ``a.isComment === b.isComment &&``  ``isDef(a.data) === isDef(b.data) &&``  ``sameInputType(a, b)`` ``)``}`
```

sameVnode会对传入的2个vnode进行基本属性的比较，只有当基本属性相同的情况下才认为这个2个vnode只是局部发生了更新，然后才会对这2个vnode进行diff，如果2个vnode的基本属性存在不一致的情况，那么就会直接跳过diff的过程，进而依据vnode新建一个真实的dom，同时删除老的dom节点。

vnode基本属性的定义可以参见源码:src/vdom/vnode.js里面对于vnode的定义。

```javascript
`constructor (``  ``tag?: string,``  ``data?: VNodeData,     ``// 关于这个节点的data值，包括attrs,style,hook等``  ``children?: ?Array<VNode>, ``// 子vdom节点``  ``text?: string,    ``// 文本内容``  ``elm?: Node,      ``// 真实的dom节点``  ``context?: Component, ``// 创建这个vdom的上下文``  ``componentOptions?: VNodeComponentOptions`` ``) {``  ``this``.tag = tag``  ``this``.data = data``  ``this``.children = children``  ``this``.text = text``  ``this``.elm = elm``  ``this``.ns = undefined``  ``this``.context = context``  ``this``.functionalContext = undefined``  ``this``.key = data && data.key``  ``this``.componentOptions = componentOptions``  ``this``.componentInstance = undefined``  ``this``.parent = undefined``  ``this``.raw = ``false``  ``this``.isStatic = ``false``  ``this``.isRootInsert = ``true``  ``this``.isComment = ``false``  ``this``.isCloned = ``false``  ``this``.isOnce = ``false`` ``}` ` ``// DEPRECATED: alias for componentInstance for backwards compat.`` ``/* istanbul ignore next */`` ``get child (): Component | void {``  ``return` `this``.componentInstance`` ``}``}`
```

每一个vnode都映射到一个真实的dom节点上。其中几个比较重要的属性:

1. tag 属性即这个vnode的标签属性
2. data 属性包含了最后渲染成真实dom节点后，节点上的class,attribute,style以及绑定的事件
3. children 属性是vnode的子节点
4. text 属性是文本属性
5. elm 属性为这个vnode对应的真实dom节点
6. key 属性是vnode的标记，在diff过程中可以提高diff的效率，后文有讲解

比如，我定义了一个vnode，它的数据结构是:

```javascript
`{``  ``tag: ``'div'``  ``data: {``    ``id: ``'app'``,``    ``class: ``'page-box'``  ``},``  ``children: [``    ``{``      ``tag: ``'p'``,``      ``text: ``'this is demo'``    ``}``  ``]``}`
```

最后渲染出的实际的dom结构就是:

```html
`<``div` `id``=``"app"` `class``=``"page-box"``>``  ``<``p``>this is demo</``p``>``</``div``>`
```

让我们再回到patch函数当中，在当oldVnode不存在的时候，这个时候是root节点初始化的过程，因此调用了createElm(vnode, insertedVnodeQueue, parentElm, refElm)方法去创建一个新的节点。而当oldVnode是vnode且sameVnode(oldVnode, vnode)2个节点的基本属性相同，那么就进入了2个节点的diff过程。

diff的过程主要是通过调用patchVnode方法进行的:

```javascript
`function` `patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly) {``  ``...``}`
```

```
`if` `(isDef(data) && isPatchable(vnode)) {``   ``// cbs保存了hooks钩子函数: 'create', 'activate', 'update', 'remove', 'destroy'``   ``// 取出cbs保存的update钩子函数，依次调用，更新attrs/style/class/events/directives/refs等属性``   ``for` `(i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)``   ``if` `(isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)``  ``}`
```

更新真实dom节点的data属性，相当于对dom节点进行了预处理的操作

接下来:

```javascript
`...``const elm = vnode.elm = oldVnode.elm``const oldCh = oldVnode.children``const ch = vnode.children``// 如果vnode没有文本节点``if` `(isUndef(vnode.text)) {`` ``// 如果oldVnode的children属性存在且vnode的属性也存在`` ``if` `(isDef(oldCh) && isDef(ch)) {``  ``// updateChildren，对子节点进行diff``  ``if` `(oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)`` ``} ``else` `if` `(isDef(ch)) {``  ``// 如果oldVnode的text存在，那么首先清空text的内容``  ``if` `(isDef(oldVnode.text)) nodeOps.setTextContent(elm, ``''``)``  ``// 然后将vnode的children添加进去``  ``addVnodes(elm, ``null``, ch, 0, ch.length - 1, insertedVnodeQueue)`` ``} ``else` `if` `(isDef(oldCh)) {``  ``// 删除elm下的oldchildren``  ``removeVnodes(elm, oldCh, 0, oldCh.length - 1)`` ``} ``else` `if` `(isDef(oldVnode.text)) {``  ``// oldVnode有子节点，而vnode没有，那么就清空这个节点``  ``nodeOps.setTextContent(elm, ``''``)`` ``}``} ``else` `if` `(oldVnode.text !== vnode.text) {`` ``// 如果oldVnode和vnode文本属性不同，那么直接更新真是dom节点的文本元素`` ``nodeOps.setTextContent(elm, vnode.text)``}`
```

这其中的diff过程中又分了好几种情况，oldCh为oldVnode的子节点，ch为Vnode的子节点：

1. 首先进行文本节点的判断，若oldVnode.text !== vnode.text，那么就会直接进行文本节点的替换；
2. 在vnode没有文本节点的情况下，进入子节点的diff；
3. 当oldCh和ch都存在且不相同的情况下，调用updateChildren对子节点进行diff；
4. 若oldCh不存在，ch存在，首先清空oldVnode的文本节点，同时调用addVnodes方法将ch添加到elm真实dom节点当中；
5. 若oldCh存在，ch不存在，则删除elm真实节点下的oldCh子节点；
6. 若oldVnode有文本节点，而vnode没有，那么就清空这个文本节点。

这里着重分析下updateChildren方法，它也是整个diff过程中最重要的环节:

[?](https://www.jb51.net/article/118263.htm#)

```javascript
`function` `updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {``  ``// 为oldCh和newCh分别建立索引，为之后遍历的依据``  ``let oldStartIdx = 0``  ``let newStartIdx = 0``  ``let oldEndIdx = oldCh.length - 1``  ``let oldStartVnode = oldCh[0]``  ``let oldEndVnode = oldCh[oldEndIdx]``  ``let newEndIdx = newCh.length - 1``  ``let newStartVnode = newCh[0]``  ``let newEndVnode = newCh[newEndIdx]``  ``let oldKeyToIdx, idxInOld, elmToMove, refElm``  ` `  ``// 直到oldCh或者newCh被遍历完后跳出循环``  ``while` `(oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {``   ``if` `(isUndef(oldStartVnode)) {``    ``oldStartVnode = oldCh[++oldStartIdx] ``// Vnode has been moved left``   ``} ``else` `if` `(isUndef(oldEndVnode)) {``    ``oldEndVnode = oldCh[--oldEndIdx]``   ``} ``else` `if` `(sameVnode(oldStartVnode, newStartVnode)) {``    ``patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue)``    ``oldStartVnode = oldCh[++oldStartIdx]``    ``newStartVnode = newCh[++newStartIdx]``   ``} ``else` `if` `(sameVnode(oldEndVnode, newEndVnode)) {``    ``patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue)``    ``oldEndVnode = oldCh[--oldEndIdx]``    ``newEndVnode = newCh[--newEndIdx]``   ``} ``else` `if` `(sameVnode(oldStartVnode, newEndVnode)) { ``// Vnode moved right``    ``patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue)``    ``canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))``    ``oldStartVnode = oldCh[++oldStartIdx]``    ``newEndVnode = newCh[--newEndIdx]``   ``} ``else` `if` `(sameVnode(oldEndVnode, newStartVnode)) { ``// Vnode moved left``    ``patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue)``    ``// 插入到老的开始节点的前面``    ``canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)``    ``oldEndVnode = oldCh[--oldEndIdx]``    ``newStartVnode = newCh[++newStartIdx]``   ``} ``else` `{``    ``// 如果以上条件都不满足，那么这个时候开始比较key值，首先建立key和index索引的对应关系``    ``if` `(isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)``    ``idxInOld = isDef(newStartVnode.key) ? oldKeyToIdx[newStartVnode.key] : ``null``    ``// 如果idxInOld不存在``    ``// 1. newStartVnode上存在这个key,但是oldKeyToIdx中不存在``    ``// 2. newStartVnode上并没有设置key属性``    ``if` `(isUndef(idxInOld)) { ``// New element``     ``// 创建新的dom节点``     ``// 插入到oldStartVnode.elm前面``     ``// 参见createElm方法``     ``createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)``     ``newStartVnode = newCh[++newStartIdx]``    ``} ``else` `{``     ``elmToMove = oldCh[idxInOld]``     ``/* istanbul ignore if */``     ``if` `(process.env.NODE_ENV !== ``'production'` `&& !elmToMove) {``      ``warn(``       ``'It seems there are duplicate keys that is causing an update error. '` `+``       ``'Make sure each v-for item has a unique key.'``      ``)``     ` `     ``// 将找到的key一致的oldVnode再和newStartVnode进行diff``     ``if` `(sameVnode(elmToMove, newStartVnode)) {``      ``patchVnode(elmToMove, newStartVnode, insertedVnodeQueue)``      ``oldCh[idxInOld] = undefined``      ``// 移动node节点``      ``canMove && nodeOps.insertBefore(parentElm, newStartVnode.elm, oldStartVnode.elm)``      ``newStartVnode = newCh[++newStartIdx]``     ``} ``else` `{``      ``// same key but different element. treat as new element``      ``// 创建新的dom节点``      ``createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)``      ``newStartVnode = newCh[++newStartIdx]``     ``}``    ``}``   ``}``  ``}``  ``// 如果最后遍历的oldStartIdx大于oldEndIdx的话``  ``if` `(oldStartIdx > oldEndIdx) {    ``// 如果是老的vdom先被遍历完``   ``refElm = isUndef(newCh[newEndIdx + 1]) ? ``null` `: newCh[newEndIdx + 1].elm``   ``// 添加newVnode中剩余的节点到parentElm中``   ``addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)``  ``} ``else` `if` `(newStartIdx > newEndIdx) { ``// 如果是新的vdom先被遍历完，则删除oldVnode里面所有的节点``   ``// 删除剩余的节点``   ``removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx)``  ``}``}`
```

在开始遍历diff前，首先给oldCh和newCh分别分配一个startIndex和endIndex来作为遍历的索引，当oldCh或者newCh遍历完后(遍历完的条件就是oldCh或者newCh的startIndex >= endIndex)，就停止oldCh和newCh的diff过程。接下来通过实例来看下整个diff的过程(节点属性中不带key的情况):

首先从第一个节点开始比较，不管是oldCh还是newCh的起始或者终止节点都不存在sameVnode，同时节点属性中是不带key标记的，因此第一轮的diff完后，newCh的startVnode被添加到oldStartVnode的前面，同时newStartIndex前移一位；



[![img](https://files.jb51.net/file_images/article/201707/201707100945207.jpg)](https://files.jb51.net/file_images/article/201707/201707100945207.jpg)

第二轮的diff中，满足sameVnode(oldStartVnode, newStartVnode)，因此对这2个vnode进行diff，最后将patch打到oldStartVnode上，同时oldStartVnode和newStartIndex都向前移动一位

[![img](https://files.jb51.net/file_images/article/201707/201707100945218.jpg)](https://files.jb51.net/file_images/article/201707/201707100945218.jpg)

第三轮的diff中，满足sameVnode(oldEndVnode, newStartVnode)，那么首先对oldEndVnode和newStartVnode进行diff，并对oldEndVnode进行patch，并完成oldEndVnode移位的操作，最后newStartIndex前移一位，oldStartVnode后移一位；

[![img](https://files.jb51.net/file_images/article/201707/201707100945219.jpg)](https://files.jb51.net/file_images/article/201707/201707100945219.jpg)

第四轮的diff中，过程同步骤3；

[![img](https://files.jb51.net/file_images/article/201707/2017071009452110.jpg)](https://files.jb51.net/file_images/article/201707/2017071009452110.jpg)

第五轮的diff中，同过程1；

[![img](https://files.jb51.net/file_images/article/201707/2017071009452111.jpg)](https://files.jb51.net/file_images/article/201707/2017071009452111.jpg)

遍历的过程结束后，newStartIdx > newEndIdx，说明此时oldCh存在多余的节点，那么最后就需要将这些多余的节点删除。

[![img](https://files.jb51.net/file_images/article/201707/2017071009452212.jpg)](https://files.jb51.net/file_images/article/201707/2017071009452212.jpg)

在vnode不带key的情况下，每一轮的diff过程当中都是起始和结束节点进行比较，直到oldCh或者newCh被遍历完。而当为vnode引入key属性后，在每一轮的diff过程中，当起始和结束节点都没有找到sameVnode时，首先对oldCh中进行key值与索引的映射:

```javascript
`if` `(isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)``idxInOld = isDef(newStartVnode.key) ? oldKeyToIdx[newStartVnode.key] : ``null`
```

createKeyToOldIdx方法，用以将oldCh中的key属性作为键，而对应的节点的索引作为值。然后再判断在newStartVnode的属性中是否有key，且是否在oldKeyToIndx中找到对应的节点。

如果不存在这个key，那么就将这个newStartVnode作为新的节点创建且插入到原有的root的子节点中:

```
`if` `(isUndef(idxInOld)) { ``// New element``  ``// 创建新的dom节点``  ``// 插入到oldStartVnode.elm前面``  ``// 参见createElm方法``  ``createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)``     ``newStartVnode = newCh[++newStartIdx]``    ``} `
```

如果存在这个key，那么就取出oldCh中的存在这个key的vnode，然后再进行diff的过程:

```javascript
`elmToMove = oldCh[idxInOld]``     ``/* istanbul ignore if */``     ``if` `(process.env.NODE_ENV !== ``'production'` `&& !elmToMove) {``     ` `     ``// 将找到的key一致的oldVnode再和newStartVnode进行diff``     ``if` `(sameVnode(elmToMove, newStartVnode)) {``      ``patchVnode(elmToMove, newStartVnode, insertedVnodeQueue)``      ``// 清空这个节点``      ``oldCh[idxInOld] = undefined``      ``// 移动node节点``      ``canMove && nodeOps.insertBefore(parentElm, newStartVnode.elm, oldStartVnode.elm)``      ``newStartVnode = newCh[++newStartIdx]``     ``} ``else` `{``      ``// same key but different element. treat as new element``      ``// 创建新的dom节点``      ``createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm)``      ``newStartVnode = newCh[++newStartIdx]``     ``}`
```

通过以上分析，给vdom上添加key属性后，遍历diff的过程中，当起始点, 结束点的搜寻及diff出现还是无法匹配的情况下时，就会用key来作为唯一标识，来进行diff，这样就可以提高diff效率。

带有Key属性的vnode的diff过程可见下图：

注意在第一轮的diff过后oldCh上的B节点被删除了，但是newCh上的B节点上elm属性保持对oldCh上B节点的elm引用。

