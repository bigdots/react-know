
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
