## 渲染虚拟 DOM: ReactDOM.render

上一节，我吗已经创建了一个组件，现在我们将这个组件渲染到页面中。

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

```js
renderSubtreeIntoContainer = function(
    parentComponent,
    children,
    container,
    forceHydrate,
    callback
) {
    let root = container._reactRootContainer;
    // 如果container._reactRootContainer （=DOMRenderer.createContainer
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
DOMRenderer.createContainer = function() {};
```

```js
ReactFiberReconciler = function(config){
    return {
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
