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
renderSubtreeIntoContainer = function(
    parentComponent,
    children,
    container,
    forceHydrate,
    callback
) {
    let root = container._reactRootContainer;
    // 如果container._reactRootContainer 存在
    if (!root) {
        const shouldHydrate =
            forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
        const newRoot = DOMRenderer.createContainer(container, shouldHydrate);
        root = container._reactRootContainer = newRoot;
        DOMRenderer.unbatchedUpdates(() => {
            DOMRenderer.updateContainer(
                children,
                newRoot,
                parentComponent,
                callback
            );
        });
    } else {
        DOMRenderer.updateContainer(children, root, parentComponent, callback);
    }
    //返回一个引用， 指向 ReactComponent 的根实例
    return DOMRenderer.getPublicRootInstance(root);
};
```

```js
function getPublicRootInstance(container){
      const containerFiber = container.current;
      if (!containerFiber.child) {
        return null;
      }
      switch (containerFiber.child.tag) {
        case HostComponent:
          return getPublicInstance(containerFiber.child.stateNode);
        default:
          return containerFiber.child.stateNode;
      }
    },
```

```js
//判断跟节点是否为元素节点并且拥有'data-reactroot'属性
function shouldHydrateDueToLegacyHeuristic(container) {
    const rootElement = getReactRootElementInContainer(container);
    return !!(
        rootElement &&
        rootElement.nodeType === ELEMENT_NODE && //ELEMENT_NODE = 1  即元素节点
        rootElement.hasAttribute(ROOT_ATTRIBUTE_NAME)
    ); // ROOT_ATTRIBUTE_NAME='data-reactroot'
}
```

```js
DOMRenderer = ReactFiberReconciler({});
```

```js
DOMRenderer.createContainer = function() {
    return {};
};
```

```js
ReactFiberReconciler = function(config){
    return {
        // container,
        // shouldHydrate
        createContainer(containerInfo, hydrate) {
            return createFiberRoot(containerInfo, hydrate);
        },
        ...
    }
}
```

```js
function createFiberRoot(containerInfo, hydrate) {
    const uninitializedFiber = createHostRootFiber();
    const root = {
        current: uninitializedFiber,
        containerInfo: containerInfo,
        pendingChildren: null,
        remainingExpirationTime: NoWork,
        isReadyForCommit: false,
        finishedWork: null,
        context: null,
        pendingContext: null,
        hydrate,
        nextScheduledRoot: null
    };
    uninitializedFiber.stateNode = root;
    return root;
}
```

```js
function unbatchedUpdates(fn) {
    if (isBatchingUpdates && !isUnbatchingUpdates) {
        isUnbatchingUpdates = true;
        try {
            return fn();
        } finally {
            isUnbatchingUpdates = false;
        }
    }
    return fn();
}
```

```js
function updateContainer(element, container, parentComponent, callback) {
    // TODO: If this is a nested container, this won't be the root.
    const current = container.current;

    const context = getContextForSubtree(parentComponent);
    if (container.context === null) {
        container.context = context;
    } else {
        container.pendingContext = context;
    }

    scheduleTopLevelUpdate(current, element, callback);
}
```

```js
function getContextForSubtree(parentComponent) {
    // 渲染根结点时parentComponent = null
    if (!parentComponent) {
        return emptyObject; //{}
    }

    // 返回parentComponent的_reactInternalFiber
    const fiber = ReactInstanceMap.get(parentComponent);
    const parentContext = findCurrentUnmaskedContext(fiber);
    return isContextProvider(fiber)
        ? processChildContext(fiber, parentContext)
        : parentContext;
}
```

```js
ReactInstanceMap.get = function(key) {
    return key._reactInternalFiber;
};
```

```js
function scheduleTopLevelUpdate(current, element, callback) {
    callback = callback === undefined ? null : callback;
    let expirationTime;
    // Check if the top-level element is an async wrapper component. If so,
    // treat updates to the root as async. This is a bit weird but lets us
    // avoid a separate `renderAsync` API.
    if (
        enableAsyncSubtreeAPI &&
        element != null &&
        (element: any).type != null &&
        (element: any).type.prototype != null &&
        (element: any).type.prototype.unstable_isAsyncReactComponent === true
    ) {
        expirationTime = computeAsyncExpiration();
    } else {
        expirationTime = computeExpirationForFiber(current);
    }

    const update = {
        expirationTime,
        partialState: { element },
        callback,
        isReplace: false,
        isForced: false,
        nextCallback: null,
        next: null
    };
    insertUpdateIntoFiber(current, update);
    scheduleWork(current, expirationTime);
}
```
