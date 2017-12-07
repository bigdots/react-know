# 渲染虚拟 DOM: ReactDOM.render

上一节，我们已经创建了一个组件，现在我们将这个组件渲染到页面中。

```js
// 渲染到页面
ReactDOM.render(<Welcome />, document.getElementById("root"));
```

ReactDOM.render 是 React 的最基本方法用于将 react 元素转化为 HTML 语言并插入指定
的 DOM 节点。

```js
render: function(nextElement, container, callback) {
    return ReactMount._renderSubtreeIntoContainer(
      null,
      nextElement,
      container,
      callback,
    );
}
```

render 接收三个参数：

* element （ react 元素 ）
* container （ 容器节点 ）
* callback （ 回调函数 ）。

并且返回 renderSubtreeIntoContainer 函数的调用。

## renderSubtreeIntoContainer 渲染组件树到容器节点

`renderSubtreeIntoContainer`顾名思义，它的功能是将子树注入到指定的 container 中
。

```js
// path: /src/renderers/dom/client/ReactMount.js
/**
 * @param {parentComponent} 父组件，对于第一次渲染，为null
 * @param {nextElement} 要插入到DOM中的ReactComponent
 * @param {container} 要插入到的容器
 * @param {callback} 回调函数
 *
 * @return {component}  返回ReactComponent，对于ReactDOM.render()调用，不用管返回值。
 */
_renderSubtreeIntoContainer: function(
    parentComponent,
    nextElement,
    container,
    callback
) {
    // ReactUpdateQueue 实现了在下一次的调和算法中更新state
    // validateCallback 检验回调函数的合法性
    ReactUpdateQueue.validateCallback(callback, "ReactDOM.render");

    //  TopLevelWrapper是一个构造函数，this.rootID唯一
    //  封装ReactElement，将nextElement挂载到wrapper的props属性下
    var nextWrappedElement = React.createElement(TopLevelWrapper, {
        child: nextElement
    });

    // parentComponent为null，nextContext={}
    var nextContext;
    if (parentComponent) {
        var parentInst = ReactInstanceMap.get(parentComponent);
        nextContext = parentInst._processChildContext(parentInst._context);
    } else {
        nextContext = emptyObject;
    }

    // 获取要插入到的容器的前一次的ReactComponent，这是为了做DOM diff
    // 对于ReactDOM.render()调用，prevComponent为null
    var prevComponent = getTopLevelWrapperInContainer(container);

    if (prevComponent) {
        // 从prevComponent中获取到prevElement这个数据对象
        var prevWrappedElement = prevComponent._currentElement;
        var prevElement = prevWrappedElement.props.child;

        // shouldUpdateReactComponent做diff运算，判断组件是更新还是卸载后重新挂载
        if (shouldUpdateReactComponent(prevElement, nextElement)) {
            var publicInst = prevComponent._renderedComponent.getPublicInstance();
            var updatedCallback =
                callback &&
                function() {
                    callback.call(publicInst);
                };
            // 只需要update，调用_updateRootComponent，然后直接return了
            ReactMount._updateRootComponent(
                prevComponent,
                nextWrappedElement,
                nextContext,
                container,
                updatedCallback
            );
            return publicInst;
        } else {
            // 不做update，直接先卸载再挂载。即unmountComponent,再mountComponent。mountComponent在后面代码中进行
            ReactMount.unmountComponentAtNode(container);
        }
    }

    var reactRootElement = getReactRootElementInContainer(container);
    var containerHasReactMarkup =
        reactRootElement && !!internalGetID(reactRootElement);
    var containerHasNonRootReactChild = hasNonRootReactChild(container);

    var shouldReuseMarkup =
        containerHasReactMarkup &&
        !prevComponent &&
        !containerHasNonRootReactChild;

    //将ReactElement元素，转化为DOM元素并且插入到对应的Container元素中去；
    var component = ReactMount._renderNewRootComponent(
        nextWrappedElement,
        container,
        shouldReuseMarkup,
        nextContext
    )._renderedComponent.getPublicInstance();

    // 调用callback
    if (callback) {
        callback.call(component);
    }
    return component;
};
```

### shouldUpdateReactComponent 组件更新机制 —— React DOM diff 算法

`shouldUpdateReactComponent` 传入前后两次 ReactElement: `prevElement` 和
`nextElement`。返回 :

* `true` : 更新 `prevElement`
* `false` : 销毁 `prevElement`， 挂载 `nextElement`

```js
/**
 * Given a `prevElement` and `nextElement`, determines if the existing
 * instance should be updated as opposed to being destroyed or replaced by a new
 * instance. Both arguments are elements. This ensures that this logic can
 * operate on stateless trees without any backing instance.
 *
 * @param {?object} prevElement
 * @param {?object} nextElement
 * @return {boolean} True if the existing instance should be updated.
 * @protected
 */
function shouldUpdateReactComponent(prevElement, nextElement) {
    // 两次ReactElement都为null或者false中的任意一项，返回true。
    var prevEmpty = prevElement === null || prevElement === false;
    var nextEmpty = nextElement === null || nextElement === false;
    if (prevEmpty || nextEmpty) {
        return prevEmpty === nextEmpty;
    }

    var prevType = typeof prevElement;
    var nextType = typeof nextElement;

    /**
     * React DOM diff算法
     *
     * prevElement 和 nextElement 任意一个的类型为数字或者字符串，直接返回true；
     * 比较前后俩次 ReactElement 的type 和 key属性是否相等。进行返回；
     */
    if (prevType === "string" || prevType === "number") {
        return nextType === "string" || nextType === "number";
    } else {
        return (
            nextType === "object" &&
            prevElement.type === nextElement.type &&
            prevElement.key === nextElement.key
        );
    }
}
```

### _renderNewRootComponent 渲染新的根组件

```js
_renderNewRootComponent: function(
    nextElement,
    container,
    shouldReuseMarkup,
    context,
  ) {

    ReactBrowserEventEmitter.ensureScrollValueMonitoring();

    // 根据ReactElement中不同的type字段，创建不同类型的组件对象。
    var componentInstance = instantiateReactComponent(nextElement, false);

    // 处理batchedMountComponentIntoNode方法调用，将ReactComponent插入DOM中，
    ReactUpdates.batchedUpdates(
      batchedMountComponentIntoNode,
      componentInstance,
      container,
      shouldReuseMarkup,
      context,
    );

    var wrapperID = componentInstance._instance.rootID;
    instancesByReactRootID[wrapperID] = componentInstance;

    return componentInstance;
  },
```

#### instantiateReactComponent 创建将被挂载的组件实例

`instantiateReactComponent` 创建一个将被挂载的实例并返回。这里根据传入的 node 会
有四种处理情况：

* node 为 `null/fasle`
* node 为 原生 `html`
* node 为 `ReactComponent`
* node 为 `string/number`

```js
//path: /src/renderers/shared/stack/reconciler/instantiateReactComponent.js
/**
 * Given a ReactNode, create an instance that will actually be mounted.
 *
 * @param {ReactNode} node
 * @param {boolean} shouldHaveDebugID
 * @return {object} A new instance of the element's constructor.
 * @protected
 */
function instantiateReactComponent(node, shouldHaveDebugID) {
    var instance;

    if (node === null || node === false) {
        instance = ReactEmptyComponent.create(instantiateReactComponent);
    } else if (typeof node === "object") {
        var element = node;
        var type = element.type;

        if (typeof element.type === "string") {
            // type为string则表示DOM原生对象，比如div span等
            instance = ReactHostComponent.createInternalComponent(element);
        } else if (isInternalComponentType(element.type)) {
            // 忽略
        } else {
            // React自定义组件
            instance = new ReactCompositeComponentWrapper(element);
        }
    } else if (typeof node === "string" || typeof node === "number") {
        // node本身是string或者number类型时
        instance = ReactHostComponent.createInstanceForText(node);
    } else {
        // 忽略
    }

    // These two fields are used by the DOM and ART diffing algorithms
    // respectively. Instead of using expandos on components, we should be
    // storing the state needed by the diffing algorithms elsewhere.

    // 这两个参数是分别为DOM和ART做`diff算法`用的。
    instance._mountIndex = 0;
    instance._mountImage = null;

    return instance;
}
```

#### batchedMountComponentIntoNode  以transaction事务的形式调用mountComponentIntoNode

```js
/**
 * Batched mount.
 *
 * @param {ReactComponent} componentInstance The instance to mount.
 * @param {DOMElement} container DOM element to mount into.
 * @param {boolean} shouldReuseMarkup If true, do not insert markup
 */
function batchedMountComponentIntoNode(
    componentInstance,
    container,
    shouldReuseMarkup,
    context
) {
    var transaction = ReactUpdates.ReactReconcileTransaction.getPooled(
        /* useCreateElement */
        !shouldReuseMarkup && ReactDOMFeatureFlags.useCreateElement
    );
    transaction.perform(
        mountComponentIntoNode,
        null,
        componentInstance,
        container,
        transaction,
        shouldReuseMarkup,
        context
    );
    ReactUpdates.ReactReconcileTransaction.release(transaction);
}
```

##### mountComponentIntoNode 将虚拟DOM解析成html

```js
/**
 * Mounts this component and inserts it into the DOM.
 *
 * @param {ReactComponent} componentInstance The instance to mount.
 * @param {DOMElement} container DOM element to mount into.
 * @param {ReactReconcileTransaction} transaction
 * @param {boolean} shouldReuseMarkup If true, do not insert markup
 */
function mountComponentIntoNode(
    wrapperInstance,
    container,
    transaction,
    shouldReuseMarkup,
    context
) {
    var markerName;
    if (ReactFeatureFlags.logTopLevelRenders) {
        var wrappedElement = wrapperInstance._currentElement.props.child;
        var type = wrappedElement.type;
        markerName =
            "React mount: " +
            (typeof type === "string" ? type : type.displayName || type.name);
        console.time(markerName);
    }

    // 调用mountComponent方法来渲染组件, 返回React组件解析的HTML
    var markup = ReactReconciler.mountComponent(
        wrapperInstance,
        transaction,
        null,
        ReactDOMContainerInfo(wrapperInstance, container),
        context,
        0 /* parentDebugID */
    );

    if (markerName) {
        console.timeEnd(markerName);
    }

    wrapperInstance._renderedComponent._topLevelWrapper = wrapperInstance;
    ReactMount._mountImageIntoNode(
        markup,
        container,
        wrapperInstance,
        shouldReuseMarkup,
        transaction
    );
}
```

###### _mountImageIntoNode 将转化后的html插入到页面中

```js
_mountImageIntoNode: function(
    markup,
    container,
    instance,
    shouldReuseMarkup,
    transaction,
  ) {
    // 对于ReactDOM.render()调用，shouldReuseMarkup为false
    if (shouldReuseMarkup) {
      var rootElement = getReactRootElementInContainer(container);
      if (ReactMarkupChecksum.canReuseMarkup(markup, rootElement)) {
        ReactDOMComponentTree.precacheNode(instance, rootElement);
        return;
      } else {
        var checksum = rootElement.getAttribute(
          ReactMarkupChecksum.CHECKSUM_ATTR_NAME,
        );
        rootElement.removeAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);

        var rootMarkup = rootElement.outerHTML;
        rootElement.setAttribute(
          ReactMarkupChecksum.CHECKSUM_ATTR_NAME,
          checksum,
        );

        var normalizedMarkup = markup;

        var diffIndex = firstDifferenceIndex(normalizedMarkup, rootMarkup);
        var difference =
          ' (client) ' +
          normalizedMarkup.substring(diffIndex - 20, diffIndex + 20) +
          '\n (server) ' +
          rootMarkup.substring(diffIndex - 20, diffIndex + 20);
      }
    }

    if (transaction.useCreateElement) {
        // 清空container的子节点
        while (container.lastChild) {
            container.removeChild(container.lastChild);
        }
        DOMLazyTree.insertTreeBefore(container, markup, null);
    } else {
        // 将markup (HTML)设置到container这个DOM元素的innerHTML属性上，这样就插入到了DOM中了
        setInnerHTML(container, markup);
        // 将本次虚拟DOM保存到container这个DOM元素的firstChild这个原生节点上。简单理解就是将Virtual DOM保存到内存中，这样可以大大提高交互效率
        ReactDOMComponentTree.precacheNode(instance, container.firstChild);
    }
  }
```

<!-- =========================== ignore ====================================== -->

```js
var topLevelRootCounter = 1;
var TopLevelWrapper = function() {
    this.rootID = topLevelRootCounter++;
};
```

```js
getTopLevelWrapperInContainer: function(container){
    var root = getHostRootInstanceInContainer(container);
    return root ? root._hostContainerInfo._topLevelWrapper : null;
}
```

```js
// reactDOM.render 返回rootEl即container中的根元素
getHostRootInstanceInContainer: function(container){
    // 返回container中的根元素
    var rootEl = getReactRootElementInContainer(container);
    var prevHostInstance =
    rootEl && ReactDOMComponentTree.getInstanceFromNode(rootEl);
    return prevHostInstance && !prevHostInstance._hostParent
            ? prevHostInstance
            : null;
}
```

```js
getReactRootElementInContainer: function(container){
    if (!container) {
    return null;
  }

  if (container.nodeType === DOC_NODE_TYPE) {
    // container = document
    return container.documentElement;
  } else {
    return container.firstChild;
  }
}
```

```js
// 传入一个DOM节点，返回一个 ReactDOMComponent／ReactDOMTextComponent的实例 或 null
ReactDOMComponentTree.getInstanceFromNode = function(node) {
    var inst = getClosestInstanceFromNode(node);
    if (inst != null && inst._hostNode === node) {
        return inst;
    } else {
        return null;
    }
};
```

```js
// path: /src/renderers/dom/client/ReactDOMComponentTree.js
var internalInstanceKey =
    "__reactInternalInstance$" +
    Math.random()
        .toString(36)
        .slice(2);
/**
 * Given a DOM node, return the closest ReactDOMComponent or
 * ReactDOMTextComponent instance ancestor.
 */
// 传入一个DOM节点，返回一个最近的 ReactDOMComponent／ReactDOMTextComponent实例的祖先元素 或者null
function getClosestInstanceFromNode(node) {
    if (node[internalInstanceKey]) {
        return node[internalInstanceKey];
    }

    // Walk up the tree until we find an ancestor whose instance we have cached.
    // 遍历nodeTree，找到缓存过的实例的祖先元素;
    var parents = [];
    while (!node[internalInstanceKey]) {
        // 当node[internalInstanceKey]不存在，则node = node.parentNode;
        parents.push(node);
        if (node.parentNode) {
            node = node.parentNode;
        } else {
            // Top of the tree. This node must not be part of a React tree (or is
            // unmounted, potentially).
            return null;
        }
    }

    var closest;
    var inst;
    for (; node && (inst = node[internalInstanceKey]); node = parents.pop()) {
        closest = inst;
        if (parents.length) {
            precacheChildNodes(inst, node);
        }
    }

    return closest;
}
```

```js
// path: /src/renderers/dom/client/ReactDOMComponentTree.js
/**
 * Populate `_hostNode` on each child of `inst`, assuming that the children
 * match up with the DOM (element) children of `node`.
 *
 * We cache entire levels at once to avoid an n^2 problem where we access the
 * children of a node sequentially and have to walk from the start to our target
 * node every time.
 *
 * Since we update `_renderedChildren` and the actual DOM at (slightly)
 * different times, we could race here and see a newer `_renderedChildren` than
 * the DOM nodes we see. To avoid this, ReactMultiChild calls
 * `prepareToManageChildren` before we change `_renderedChildren`, at which
 * time the container's child nodes are always cached (until it unmounts).
 */

/**
 * 为实例的每一个 child 添加 _hostNode
 * 预缓存子节点
 */

function precacheChildNodes(inst, node) {
    if (inst._flags & Flags.hasCachedChildNodes) {
        return;
    }
    var children = inst._renderedChildren;
    var childNode = node.firstChild;
    outer: for (var name in children) {
        if (!children.hasOwnProperty(name)) {
            continue;
        }
        var childInst = children[name];
        var childID = getRenderedHostOrTextFromComponent(childInst)._domID;
        if (childID === 0) {
            // We're currently unmounting this child in ReactMultiChild; skip it.
            continue;
        }
        // We assume the child nodes are in the same order as the child instances.
        for (; childNode !== null; childNode = childNode.nextSibling) {
            if (shouldPrecacheNode(childNode, childID)) {
                precacheNode(childInst, childNode);
                continue outer;
            }
        }
        // We reached the end of the DOM children without finding an ID match.
        invariant(false, "Unable to find element with ID %s.", childID);
    }
    inst._flags |= Flags.hasCachedChildNodes;
}
```
