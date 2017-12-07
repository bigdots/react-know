## 渲染虚拟 DOM: ReactDOM.render

上一节，我们已经创建了一个组件，现在我们将这个组件渲染到页面中。

```js
// 渲染到页面
ReactDOM.render(<Welcome />, document.getElementById("root"));
```

ReactDOM.render 是 React 的最基本方法用于将 react 元素转化为 HTML 语言并插入指定
的 DOM 节点。

```js
ReactDOM.render = function(element, container, callback) {
    return renderSubtreeIntoContainer(
        null, //父节点
        element, // 要渲染的React节点
        container, // 容器节点，必须是DOM节点: document.getElementById("root")
        false,
        callback // 回调函数
    );
};
```

render 接收三个参数：

* element （ react 元素 ）
* container （ 容器节点 ）
* callback （ 回调函数 ）。

并且返回 renderSubtreeIntoContainer 函数的调用。

`renderSubtreeIntoContainer`顾名思义，它的功能是将子树注入到指定的 container 中
。

```js
// path: /src/renderers/dom/client/ReactMount.js
renderSubtreeIntoContainer: function(
    parentComponent,
    nextElement,
    container,
    callback
) {
    // ReactUpdateQueue 实现了在下一次的调和算法中更新state
    // validateCallback 检验回调函数的合法性
    ReactUpdateQueue.validateCallback(callback, "ReactDOM.render");


    //  TopLevelWrapper是一个构造函数，this.rootID唯一
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

    var prevComponent = getTopLevelWrapperInContainer(container);

    if (prevComponent) {
        var prevWrappedElement = prevComponent._currentElement;
        var prevElement = prevWrappedElement.props.child;
        if (shouldUpdateReactComponent(prevElement, nextElement)) {
            var publicInst = prevComponent._renderedComponent.getPublicInstance();
            var updatedCallback =
                callback &&
                function() {
                    callback.call(publicInst);
                };
            ReactMount._updateRootComponent(
                prevComponent,
                nextWrappedElement,
                nextContext,
                container,
                updatedCallback
            );
            return publicInst;
        } else {
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
    var component = ReactMount._renderNewRootComponent(
        nextWrappedElement,
        container,
        shouldReuseMarkup,
        nextContext
    )._renderedComponent.getPublicInstance();
    if (callback) {
        callback.call(component);
    }
    return component;
};
```

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
    invariant(false, 'Unable to find element with ID %s.', childID);
  }
  inst._flags |= Flags.hasCachedChildNodes;
}
```