# ReactMount

```js
var ReactMount = {
    TopLevelWrapper: TopLevelWrapper,

    /**
     * Used by devtools. The keys are not important.
     */
    _instancesByReactRootID: instancesByReactRootID,

    scrollMonitor: function() {},

    _updateRootComponent: function(){},

    _renderNewRootComponent: function() {},

    renderSubtreeIntoContainer: function() {},

    _renderSubtreeIntoContainer: function() {}

    render: function() {},

    unmountComponentAtNode: function() {},

    _mountImageIntoNode: function(){}
};
```

## scrollMonitor

```js
/**
 * This is a hook provided to support rendering React components while
 * ensuring that the apparent scroll position of its `container` does not
 * change.
 *
 * @param {DOMElement} container 已经被渲染到页面的组件
 * @param {function} renderCallback This must be called once to do the render.
 */
scrollMonitor = function(container, renderCallback) {
    renderCallback();
};
```

## _updateRootComponent

```js
/**
 * Take a component that's already mounted into the DOM and replace its props
 * @param {ReactComponent} prevComponent 已经渲染完成的组件
 * @param {ReactElement} nextElement 即将被渲染的组件实例
 * @param {DOMElement} container 容器
 * @param {?function} callback 完成后的回调函数
 */
_updateRootComponent = function(
    prevComponent,
    nextElement,
    nextContext,
    container,
    callback
) {
    ReactMount.scrollMonitor(container, function() {
        ReactUpdateQueue.enqueueElementInternal(
            prevComponent,
            nextElement,
            nextContext
        );
        // 调用回调函数
        if (callback) {
            ReactUpdateQueue.enqueueCallbackInternal(prevComponent, callback);
        }
    });

    return prevComponent;
};
```

## _renderNewRootComponent

```js
/**
 * 渲染一个新组件到DOM中. Hooked by hooks!
 *
 * @param {ReactElement} nextElement 即将被渲染的Element
 * @param {DOMElement} container 容器
 * @param {boolean} shouldReuseMarkup if we should skip the markup insertion
 * @return {ReactComponent} 返回新组件
 */
_renderNewRootComponent = function(
    nextElement,
    container,
    shouldReuseMarkup,
    context
) {
    ReactBrowserEventEmitter.ensureScrollValueMonitoring();
    var componentInstance = instantiateReactComponent(nextElement, false);

    // The initial render is synchronous but any updates that happen during
    // rendering, in componentWillMount or componentDidMount, will be batched
    // according to the current batching strategy.

    ReactUpdates.batchedUpdates(
        batchedMountComponentIntoNode,
        componentInstance,
        container,
        shouldReuseMarkup,
        context
    );

    var wrapperID = componentInstance._instance.rootID;
    instancesByReactRootID[wrapperID] = componentInstance;

    return componentInstance;
};
```

## _renderSubtreeIntoContainer

```js
/**
 * 渲染一个组件到DOM中
 * Renders a React component into the DOM in the supplied `container`.
 *
 * If the React component was previously rendered into `container`, this will
 * perform an update on it and only mutate the DOM as necessary to reflect the
 * latest React component.
 *
 * @param {ReactComponent} parentComponent The conceptual parent of this render tree.
 * @param {ReactElement} nextElement Component element to render.
 * @param {DOMElement} container DOM element to render into.
 * @param {?function} callback function triggered on completion
 * @return {ReactComponent} Component instance rendered in `container`.
 */
_renderSubtreeIntoContainer = function(
    parentComponent,
    nextElement,
    container,
    callback
) {
    ReactUpdateQueue.validateCallback(callback, "ReactDOM.render");

    var nextWrappedElement = React.createElement(TopLevelWrapper, {
        child: nextElement
    });

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

    // 调用回调
    if (callback) {
        callback.call(component);
    }
    return component;
};
```

## renderSubtreeIntoContainer

```js
renderSubtreeIntoContainer = function(
    parentComponent,
    nextElement,
    container,
    callback
) {
    return ReactMount._renderSubtreeIntoContainer(
        parentComponent,
        nextElement,
        container,
        callback
    );
};
```

## render

```js
/**
 * Renders a React component into the DOM in the supplied `container`.
 * See https://facebook.github.io/react/docs/top-level-api.html#reactdom.render
 *
 * If the React component was previously rendered into `container`, this will
 * perform an update on it and only mutate the DOM as necessary to reflect the
 * latest React component.
 *
 * @param {ReactElement} nextElement Component element to render.
 * @param {DOMElement} container DOM element to render into.
 * @param {?function} callback function triggered on completion
 * @return {ReactComponent} Component instance rendered in `container`.
 */
render = function(nextElement, container, callback) {
    return ReactMount._renderSubtreeIntoContainer(
        null,
        nextElement,
        container,
        callback
    );
};
```

## unmountComponentAtNode

```js
/**
 * 销毁一个已经挂载的React组件
 * Unmounts and destroys the React component rendered in the `container`.
 * See https://facebook.github.io/react/docs/top-level-api.html#reactdom.unmountcomponentatnode
 *
 * @param {DOMElement} container DOM element containing a React component.
 * @return {boolean} True if a component was found in and unmounted from
 *                   `container`
 */
unmountComponentAtNode = function(container) {
    var prevComponent = getTopLevelWrapperInContainer(container);
    if (!prevComponent) {
        // Check if the node being unmounted was rendered by React, but isn't a
        // root node.
        var containerHasNonRootReactChild = hasNonRootReactChild(container);

        // Check if the container itself is a React root node.
        var isContainerReactRoot =
            container.nodeType === 1 && container.hasAttribute(ROOT_ATTR_NAME);

        return false;
    }
    delete instancesByReactRootID[prevComponent._instance.rootID];
    ReactUpdates.batchedUpdates(
        unmountComponentFromNode,
        prevComponent,
        container,
        false
    );
    return true;
};
```

## _mountImageIntoNode

```js
_mountImageIntoNode = function(
    markup,
    container,
    instance,
    shouldReuseMarkup,
    transaction
) {
    if (shouldReuseMarkup) {
        var rootElement = getReactRootElementInContainer(container);
        if (ReactMarkupChecksum.canReuseMarkup(markup, rootElement)) {
            ReactDOMComponentTree.precacheNode(instance, rootElement);
            return;
        } else {
            var checksum = rootElement.getAttribute(
                ReactMarkupChecksum.CHECKSUM_ATTR_NAME
            );
            rootElement.removeAttribute(ReactMarkupChecksum.CHECKSUM_ATTR_NAME);

            var rootMarkup = rootElement.outerHTML;
            rootElement.setAttribute(
                ReactMarkupChecksum.CHECKSUM_ATTR_NAME,
                checksum
            );

            var normalizedMarkup = markup;

            var diffIndex = firstDifferenceIndex(normalizedMarkup, rootMarkup);
            var difference =
                " (client) " +
                normalizedMarkup.substring(diffIndex - 20, diffIndex + 20) +
                "\n (server) " +
                rootMarkup.substring(diffIndex - 20, diffIndex + 20);
        }
    }

    if (transaction.useCreateElement) {
        while (container.lastChild) {
            container.removeChild(container.lastChild);
        }
        DOMLazyTree.insertTreeBefore(container, markup, null);
    } else {
        setInnerHTML(container, markup);
        ReactDOMComponentTree.precacheNode(instance, container.firstChild);
    }
};
```
