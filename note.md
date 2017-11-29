## React

## UI

## 虚拟 DOM

## 数据流 flux

## JSX

### ReactDOM.render()

将元素渲染到 DOM 中

```js
const element = <h1>Hello, world</h1>;
ReactDOM.render(element, document.getElementById("root"));
```

1. ReactDOM

2. render

### class AddSubject extends React.Component

## Component 类

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
// 将isReactComponent／setState／forceUpdate添加到Component的原型对象中，让所有实例共享这些实例以及方法

Component.prototype.isReactComponent = {};

Component.prototype.setState = function(partialState, callback) {})

Component.prototype.forceUpdate = function(callback) {}
```

## state

## 更新机制

```js
/***
 * partialState  需要更新的state／function
 * callback 回调函数
 *
 * 调用实例中updater的enqueueSetState方法
 ***/
setState = function(partialState, callback) {
    invariant(
        typeof partialState === "object" ||
            typeof partialState === "function" ||
            partialState == null,
        "setState(...): takes an object of state variables to update or a " +
            "function which returns an object of state variables."
    );
    this.updater.enqueueSetState(this, partialState, callback, "setState");
};
```

### updater/ReactNoopUpdateQueue

updater 是一个 ReactNoopUpdateQueue；

```js
/**
 * 抽象出的更新队列.
 */
var ReactNoopUpdateQueue = {
    /**
     * 检查组件是否已经挂载（mounted）
     * @param {ReactClass} publicInstance The instance we want to test.
     * @return {boolean} 返回布尔值，true表示已挂载.
     * @protected
     * @final
     */
    isMounted: function(publicInstance) {
        return false;
    },

    /**
     * Forces an update. This should only be invoked when it is known with
     * certainty that we are **not** in a DOM transaction.
     *
     * You may want to call this when you know that some deeper aspect of the
     * component's state has changed but `setState` was not called.
     *
     * This will not invoke `shouldComponentUpdate`, but it will invoke
     * `componentWillUpdate` and `componentDidUpdate`.
     *
     * @param {ReactClass} publicInstance The instance that should rerender.
     * @param {?function} callback Called after component is updated.
     * @param {?string} callerName name of the calling function in the public API.
     * @internal
     */
    enqueueForceUpdate: function(publicInstance, callback, callerName) {
        warnNoop(publicInstance, "forceUpdate");
    },

    /**
     * 替换所有的state。通常使用`setState`来改变state；应该把`this.state`当做是不可变的（immutable）。
     * There is no guarantee that `this.state` will be immediately updated, so
     * accessing `this.state` after calling this method may return the old value.
     *
     * @param {ReactClass} publicInstance 重新render的组件实例.
     * @param {object} completeState 需要更新的stat.
     * @param {?function} callback 可选。组件更新完成（updated）后执行.
     * @param {?string} callerName name of the calling function in the public API.
     * @internal
     */
    enqueueReplaceState: function(
        publicInstance,
        completeState,
        callback,
        callerName
    ) {
        warnNoop(publicInstance, "replaceState");
    },

    /**
     * 更新state的子集
     * Sets a subset of the state. This only exists because _pendingState is
     * internal. This provides a merging strategy that is not available to deep
     * properties which is confusing. TODO: Expose pendingState or don't use it
     * during the merge.
     *
     * @param {ReactClass} publicInstance 需要重新render的组件实例.
     * @param {object} partialState 需要更新的state.
     * @param {?function} callback 可选。组件更新完成（updated）后执行.
     * @param {?string} Name of the calling function in the public API.
     * @internal
     */
    enqueueSetState: function(
        publicInstance,
        partialState,
        callback,
        callerName
    ) {
        warnNoop(publicInstance, "setState");
    }
};
```

```js
var didWarnStateUpdateForUnmountedComponent = {};
function warnNoop(publicInstance, callerName) {
    if (__DEV__) {
        var constructor = publicInstance.constructor;
        const componentName =
            (constructor && (constructor.displayName || constructor.name)) ||
            "ReactClass";
        const warningKey = `${componentName}.${callerName}`;
        if (didWarnStateUpdateForUnmountedComponent[warningKey]) {
            return;
        }
        warning(
            false,
            "%s(...): Can only update a mounted or mounting component. " +
                "This usually means you called %s() on an unmounted component. " +
                "This is a no-op.\n\nPlease check the code for the %s component.",
            callerName,
            callerName,
            componentName
        );
        didWarnStateUpdateForUnmountedComponent[warningKey] = true;
    }
}
```



## render

检查this.props 和 this.state并返回以下类型中的一个

JSX会编译成下面这种形式

```js
return React.createElement("div", null);
```


## createElement





props

组件声明周期

合成事件

虚拟 DOM
