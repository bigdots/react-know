## React



### ReactDOM.render()

1. ReactDOM

2. render

### class AddSubject extends React.Component

1. React

```js
var React = {
  Children: {...},

  Component,
  PureComponent,

  createElement: __DEV__ ? createElementWithValidation : createElement,
};
```

2. Component

 赋予组件更新state能力的基类；
```js
function Component(props, context, updater) {
  this.props = props;
  this.context = context;
  this.refs = emptyObject;
  // We initialize the default updater but the real one gets injected by the
  // renderer.
  this.updater = updater || ReactNoopUpdateQueue;
}
```

### setState

```js
Component.prototype.setState = function(partialState, callback) {
  invariant(
    typeof partialState === 'object' ||
      typeof partialState === 'function' ||
      partialState == null,
    'setState(...): takes an object of state variables to update or a ' +
      'function which returns an object of state variables.',
  );
  this.updater.enqueueSetState(this, partialState, callback, 'setState');
};
```

### forceUpdate
shouldComponentUpdate返回true或者调用forceUpdate之后，componentWillUpdate会被调用。

```js
Component.prototype.forceUpdate = function(callback) {
  this.updater.enqueueForceUpdate(this, callback, 'forceUpdate');
};
```

