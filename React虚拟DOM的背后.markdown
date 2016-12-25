# React虚拟DOM的背后

###作者：唐金州
---

相信打开这篇文章的读者对React已经不在陌生，甚至已经将其运用到项目中，打开[react官网](http://facebook.github.io/react/index.html)，不难发现react的三大特点（JUST THE UI， VIRTUAL DOM，DATA FLOW），虚拟DOM更可谓React的核心思想，本编文章就是介绍虚拟DOM是如何进行比较，又是如何更新最终的DOM节点。

我们可以简单的理解React原理：不同的数据状态对应着不同的DOM树结构，当数据状态变化后，React构建出对应的虚拟DOM树，并将其和前一种虚拟DOM树进行比较，最后仅对不同的节点进行更新，其中最关键的就是如何查找到不同的DOM节点，查找到后如何对其更新，这也是我们将要介绍的。

我们先看下虚拟DOM的构建及存储结构：

	ReactElement.createElement = function(type, config, children) {
  	...
	  return new ReactElement(
	    type,
	    key,
	    ref,
	    ReactCurrentOwner.current,
	    ReactContext.current,
	    props
	  );
	};
	结构类型：
	{
	    "type": "h1",
	    "key": null,
	    "ref": null,
	    "_owner": null,
	    "_context": {},
	    "_store": {
	        "validated": false,
	        "props": {
	            "className": "header",
	            "children": ["Hello, world!", {
	                "type": "span",
	                ...
	            }]
	        }
	    }
	}

---
	React.render(ReactElement.createElement(...), container)
	构建虚拟DOM：
	new ReactDOMComponent(reactElement,...)
	结构类型：
	{
	    _currentElement: Object，//ReactElement类型,也是参与比较的属性
	    _lifeCycleState: "MOUNTED"，
	    _mountDepth: 3，//相对组件根节点的深度
	    _mountIndex: 3，//此节点在兄弟DOM中所在的顺序（从0计数）
	    _owner: MyComponent，//节点所属组件
	    _pendingCallbacks: null，
	    _pendingElement: null，
	    _rootNodeID: ".0.0.3"，//唯一reactid
	    _tag: "li"，
	    props: Object，//className, children等属性
	    tagName: "LI"，
	    __proto__: ReactDOMComponent
	}


构建后的虚拟DOM结构如上，他们会被存储在全局变量instancesByReactRootID中，每一个虚拟DOM节点已key-value形式存储，这样在之后的比较虚拟DOM时就可以快速的取到需要比较的节点，已达到O(n)的时间复杂度,如下：

	//通过React.createClass()构建的组件结构如下，最外层包含了组件生命周期中的方法，如自定义的addLi，deleteLi等，_renderedComponent存储的就是虚拟节点（ReactDOMComponent类型）
	instancesByReactRootID = {
	    ".0" : {
	        _compositeLifeCycleState: null,
	        _currentElement: ReactElement,
	        _lifeCycleState: "MOUNTED",
	        _mountDepth: 0,
	        _owner: null,
	        _pendingCallbacks: null,
	        _pendingElement: null,
	        _pendingForceUpdate: false,
	        _pendingState: null,
	        _renderedComponent: ReactDOMComponent,
	        _rootNodeID: ".0",
	        addLi: function () { [native code] },
	        context: null,
	        deleteLi: function () { [native code] },
	        getDOMNode: function () { [native code] },
	        props: Object,
	        refs: Object,
	        state: Object,
	        updateLi: function () { [native code] },
	        __proto__: MyComponent,
	        __proto__: Object
	    }
	}


当数据状态改变后，将再次执行this.render构建renderedComponent虚拟节点(_renderValidatedComponent)，构建完成后就是递归的比较所有节点，源代码如下：

	var prevComponentInstance = this._renderedComponent;
	var prevElement = prevComponentInstance._currentElement;
	var nextElement = this._renderValidatedComponent();
	prevComponentInstance.receiveComponent(nextElement, transaction);

递归比较prevElement._renderedChildren 和 nextElement._renderedChildren是否相等，不相等则更新。

更新过程：
节点操作类型：

	INSERT_MARKUP: "INSERT_MARKUP" //添加节点
	MOVE_EXISTING: "MOVE_EXISTING" //移动节点（指定key时）
	REMOVE_NODE: "REMOVE_NODE"    //删除节点
	TEXT_CONTENT: "TEXT_CONTENT" //文本节点更新

对于以上4种节点操作类型，react不会发现一个节点不同，就更新一个，而是将需要更新的节点以及如何更新存储在一个更新队列中，等全部比较完后，统一更新，减少DOM重绘次数，提高性能。
对于节点的属性，事件等更新，将会是一边比较一边更新。

源代码如下：

	for (var k = 0; update = updates[k]; k++) {
	  	switch (update.type) {
		    case ReactMultiChildUpdateTypes.INSERT_MARKUP:
		      insertChildAt(
		        update.parentNode,
		        renderedMarkup[update.markupIndex],
		        update.toIndex
		      );
		      break;
		    case ReactMultiChildUpdateTypes.MOVE_EXISTING:
		      insertChildAt(
		        update.parentNode,
		        initialChildren[update.parentID][update.fromIndex],
		        update.toIndex
		      );
		      break;
		    case ReactMultiChildUpdateTypes.TEXT_CONTENT:
		      updateTextContent(
		        update.parentNode,
		        update.textContent
		      );
		      break;
		    case ReactMultiChildUpdateTypes.REMOVE_NODE:
		      // Already removed by the for-loop above.
		      break;
	  	}
	}

所谓的移动节点只会在手动指定了key的情况下才会发生，下面我们分别对增、删、改以及移动举例[(参考官网)](http://facebook.github.io/react/docs/reconciliation.html)：

	//添加节点
	renderA: <div><span>first</span></div>
	renderB: <div><span>first</span><span>second</span></div>
	=> [insertNode <span>second</span>]

	//删除节点
	renderA: <div><span>first</span><span>second</span></div>
	renderB: <div><span>first</span></div>
	=> [removeNode <span>second</span>]

	//类型不一样，将首先删除节点，然后插入新的节点，对于自定义组件也是同样的逻辑
	renderA: <div />
	renderB: <span />
	=> [removeNode <div />], [insertNode <span />]


	//属性改变
	renderA: <div id="before" />
	renderB: <div id="after" />
	=> [replaceAttribute id "after"]

	//改变文本节点，插入新的节点
	//如果手动指定将仅仅执行一次节点插入，后面会介绍
	renderA: <div><span>first</span></div>
	renderB: <div><span>second</span><span>first</span></div>
	=> [replaceAttribute textContent 'second'], [insertNode <span>first</span>]

对于增删改，react如何给节点标记操作类型的呢，有心的朋友可以看下ReactMultiChild.js文件中的 \_updateChildren方法，每一个(非组件根节点)虚拟DOM节点都有一个"\_mountIndex"属性，它的值为节点在同级兄弟节点中的顺序索引（从0计数），每一个节点同样也会有一个唯一的reactid。对于没有手动指定key值的节点，react会顺序生成reactid,相同reactid的节点在一起比较。上面那个例子真实的DOM节点结构如下：

	//renderA
	<div data-reactid=".0">
	    <span data-reactid=".0.0">first</span> //_mountIndex = 0>
	</div>
	
	//renderB
	<div data-reactid=".0">
	    <span data-reactid=".0.0">second</span> //_mountIndex = 0
	    <span data-reactid=".0.1">first</span> //_mountIndex = 1
	</div>

这样就不难理解为什么renderA->renderB,不是简单的插入<span>second</span>,而是先改变第一个子节点span的文本子节点，然后再插入<span>first</span>。
是否觉得上面的节点改变有点多余呢？为了解决以上问题，改变更少的DOM来完成节点的更新，react允许手动指定key，如果手动指定了key,reactid将根据key来生成，看下面的例子：

	//renderA: 
	<div>
	    <span key="test-2">2</span>
	    <span key="test-1">1</span>
	    <span key="test-3">3</span>
	    <span key="test-4">4</span>
	</div>
	//renderB: 
	<div>
	    <span key="test-5">5</span>
	    <span key="test-2">2</span>
	    <span key="test-3">3</span>
	    <span key="test-1">1</span>
	</div>

真实的DOM结构如下：

	//renderA
	<div data-reactid=".0">
	    <span data-reactid=".0.0.$test-2:0">2</span> //_mountIndex = 0>
	    <span data-reactid=".0.0.$test-1:0">1</span> //_mountIndex = 1>
	    <span data-reactid=".0.0.$test-3:0">3</span> //_mountIndex = 2>
	    <span data-reactid=".0.0.$test-4:0">4</span> //_mountIndex = 3>
	</div>
	
	//renderB
	<div data-reactid=".0">
	    <span data-reactid=".0.0.$test-5:0">5</span> //_mountIndex = 0>
	    <span data-reactid=".0.0.$test-2:0">2</span> //_mountIndex = 1>
	    <span data-reactid=".0.0.$test-3:0">3</span> //_mountIndex = 2>
	    <span data-reactid=".0.0.$test-1:0">1</span> //_mountIndex = 3>
	</div>

 切记相同reactid的节点才进行比较，并非按照_mountIndex比较，上述例子涉及到增删移动: 

	1. [insertNode <span key="test-5">5</span>],
	2. [moveNode <span key="test-1">1</span> fromIndex: 1, toIndex:3], 
	3. [removeNode <span key="test-4">4</span>]

对于本例如果使用key，从renderA->renderB则需要4次的文本节点的替换，至于在项目中是否使用key，也要跟据具体的业务场景来选择，并不是说用key一定就比不用key效率高。




