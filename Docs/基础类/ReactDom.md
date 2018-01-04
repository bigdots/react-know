<!-- TOC -->

- [ReactDOM](#reactdom)
    - [findDOMNode()](#finddomnode)
    - [render()](#render)
    - [unmountComponentAtNode()](#unmountcomponentatnode)

<!-- /TOC -->

# ReactDOM

```js
var ReactDOM = {
    findDOMNode: findDOMNode,
    render: ReactMount.render,
    unmountComponentAtNode: ReactMount.unmountComponentAtNode,
    version: ReactVersion,

    /* eslint-disable camelcase */
    unstable_batchedUpdates: ReactUpdates.batchedUpdates,
    unstable_renderSubtreeIntoContainer: renderSubtreeIntoContainer
    /* eslint-enable camelcase */
};
```

## findDOMNode()

```js
/**
 * Returns the DOM node rendered by this element.
 *
 * See https://facebook.github.io/react/docs/top-level-api.html#reactdom.finddomnode
 *
 * @param {ReactComponent|DOMElement} componentOrElement
 * @return {?DOMElement} The root node of this element.
 */
function findDOMNode(componentOrElement) {
    if (componentOrElement == null) {
        return null;
    }
    if (componentOrElement.nodeType === 1) {
        return componentOrElement;
    }

    var inst = ReactInstanceMap.get(componentOrElement);
    if (inst) {
        inst = getHostComponentFromComposite(inst);
        return inst ? ReactDOMComponentTree.getNodeFromInstance(inst) : null;
    }
}
```

## render()

```js
/**
 * nextElement ReactElement
 * container 容器
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
```

## unmountComponentAtNode()
