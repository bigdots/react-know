# React.Component

<!-- AsyncComponent,PureComponent 暂不讨论后期补加 -->

```js
function Component(props, context, updater) {
  // 将传入的props，context，updater赋给this对象，即实例
  // 给this添加一个refs属性

  this.props = props;
  this.context = context;
  this.refs = emptyObject;

  // We initialize the default updater but the real one gets injected by the
  // renderer.

  //updater：注入的render函数
  // ReactNoopUpdateQueue: 默认的更新机制
  this.updater = updater || ReactNoopUpdateQueue;
}
// 将isReactComponent／setState／forceUpdate添加到Component的原型对象中

Component.prototype.isReactComponent = {};

Component.prototype.setState = function(partialState, callback) {})

Component.prototype.forceUpdate = function(callback) {}
```

## props

## setState

## forceUpdate



























# React.PureComponent | React.AsyncComponent

类的声明同Component一致

```js
function ComponentDummy() {}
ComponentDummy.prototype = Component.prototype;

```

```js
var pureComponentPrototype = (PureComponent.prototype = new ComponentDummy());
pureComponentPrototype.constructor = PureComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(pureComponentPrototype, Component.prototype);
pureComponentPrototype.isPureReactComponent = true;

```

```js
var asyncComponentPrototype = (AsyncComponent.prototype = new ComponentDummy());
asyncComponentPrototype.constructor = AsyncComponent;
// Avoid an extra prototype jump for these methods.
Object.assign(asyncComponentPrototype, Component.prototype);
asyncComponentPrototype.unstable_isAsyncReactComponent = true;
asyncComponentPrototype.render = function() {
    return this.props.children;
};
```
