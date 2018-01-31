# 组件 render

## react.render

\_renderSubtreeIntoContainer->createElement->shouldUpdateReactComponent(diff):

* true （更新）
  \_updateRootComponent-> enqueueUpdate (-> batchedUpdates)
* false (重新挂载)
  unmountComponentAtNode(卸载)->\_renderNewRootComponent->instantiateReactComponent(生成不同类型的组件)(->batchedUpdates)

    isBatchingUpdates 表示是否正在更新

    * true
      dirtyComponents.push
    * fals
      [[[batchedUpdates-> transaction.perform(mountComponentIntoNode) -> mountComponentIntoNode -> mountComponent(解析的 HTML)]]] -> \_mountImageIntoNode(插入到 DOM)

```js
/**
 * nextElement ReactElement
 * container 容器真实DOM
 * callback 回调函数
 **/
render: function(nextElement, container, callback) {
    return ReactMount._renderSubtreeIntoContainer(
      null,
      nextElement,
      container,
      callback,
    );
}


ReactMount._renderSubtreeIntoContainer = function(
    parentComponent,
    nextElement,
    container,
    callback
) {
    //  TopLevelWrapper是一个构造函数，this.rootID唯一
    //  封装ReactElement，将nextElement挂载到wrapper的props属性下
    var nextWrappedElement = React.createElement(TopLevelWrapper, {
        child: nextElement
    });

    // 对于ReactDOM.render()调用，parentComponent为null
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
            // 不做update，直接先卸载再挂载。即unmountComponent,再mountComponent mountComponent在后面代码中进行
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

## component.render

定义组件时自定义，在 xx 中调用，生成并返回一个虚拟 DOM。

react.render ／ component.render 区别

component.render 虚拟 DOM，DOM diff

1. ReactElement

2. DOM diff： 更新 or 挂载

挂载组件：

2. 根据新的 VD，生成不同类别的组件类。

    instantiateReactComponent 方法，根据 ReactElement 中不同的 type 字段，创建不同类型的组件对象，即 ReactComponent（所有生命周期函数挂载在这里）。

3. 转成 html

    mountComponent(), 调用 React 生命周期方法解析组件，得到它的 HTML。

4. 插入 DOM

    \_mountImageIntoNode(), 通过设置 DOM 父节点的 innerHTML 属性将 HTML 插入到 DOM 父节点中。

## transaction

```js
var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];

var FLUSH_BATCHED_UPDATES = {
    initialize: emptyFunction,
    close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates)
};
```

flushBatchedUpdates -> runBatchedUpdates

递归渲染DOM树：

```js
function mountComponent(transaction, hostParent, hostContainerInfo, context) {
    //...
    markup = this.performInitialMount(
        renderedElement,
        hostParent,
        hostContainerInfo,
        transaction,
        context
    );
    //...
}

function performInitialMount(
    renderedElement,
    hostParent,
    hostContainerInfo,
    transaction,
    context
) {
    // ...
    var markup = ReactReconciler.mountComponent(
        child,
        transaction,
        hostParent,
        hostContainerInfo,
        this._processChildContext(context),
        debugID
    );

    //...
}
```
