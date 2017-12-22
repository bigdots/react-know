
# 组件
<!-- TOC -->

- [组件](#组件)
    - [ReactComponent](#reactcomponent)
        - [setState](#setstate)
        - [forceUpdate](#forceupdate)
    - [PureComponent](#purecomponent)

<!-- /TOC -->

## ReactComponent

```js
/**
 * Base class helpers for the updating state of a component.
 */
function ReactComponent(props, context, updater) {
    this.props = props;
    this.context = context;
    this.refs = emptyObject;
    // We initialize the default updater but the real one gets injected by the
    // renderer.
    this.updater = updater || ReactNoopUpdateQueue;
}

ReactComponent.prototype.isReactComponent = {};

ReactComponent.prototype.setState = function() {};

ReactComponent.prototype.forceUpdate = function() {};
```

### setState

```js
ReactComponent.prototype.setState = function(partialState, callback) {
    this.updater.enqueueSetState(this, partialState);
    if (callback) {
        this.updater.enqueueCallback(this, callback, "setState");
    }
};
```

### forceUpdate

```js
ReactComponent.prototype.forceUpdate = function(callback) {
    this.updater.enqueueForceUpdate(this);
    if (callback) {
        this.updater.enqueueCallback(this, callback, "forceUpdate");
    }
};
```

## PureComponent

React.PureComponent 与 React.Component 几乎完全相同，但 React.PureComponent 通过prop和state的浅对比来实现 shouldComponentUpate()。

如果React组件的 render() 函数在给定相同的props和state下渲染为相同的结果，在某些场景下你可以使用 React.PureComponent 来提升性能。

```js
/**
 * Base class helpers for the updating state of a component.
 */
function ReactPureComponent(props, context, updater) {
    // Duplicated from ReactComponent.
    this.props = props;
    this.context = context;
    this.refs = emptyObject;
    // We initialize the default updater but the real one gets injected by the
    // renderer.
    this.updater = updater || ReactNoopUpdateQueue;
}

function ComponentDummy() {}
ComponentDummy.prototype = ReactComponent.prototype;
ReactPureComponent.prototype = new ComponentDummy();
ReactPureComponent.prototype.constructor = ReactPureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(ReactPureComponent.prototype, ReactComponent.prototype);
ReactPureComponent.prototype.isPureReactComponent = true;
```
