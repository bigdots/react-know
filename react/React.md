# React 类

源码：

```js
var React = {
    Children: {},

    Component,
    PureComponent,
    unstable_AsyncComponent,

    Fragment,

    createElement,
    cloneElement,
    createFactory,
    isValidElement,

    version,

    __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED: {}
};
```

React 类的代码很简单，只是将一些属性（比如：Children ，
Component，createElement... ）注入。

## Component

详情请参阅[React.Children](sections/React.Children.md)

## createElement: functon

这个方法平常写 react 代码的时候可能不怎么需要用到，但其实它被使用地十分频繁。我
们用 `JSX` 编写的代码会被转换成用这个方法来实现。

```jsx
// jsx
return (
<div className="div" num={23}>
    <div>1</div>
    <div>2</div>
</div>
）

// 编译后
return React.createElement(
    "div",
    { className: "div", num: 23 },
    React.createElement("div", null, "1"),
    React.createElement("div", null, "2")
);
```

```js
createElement: __DEV__ ? createElementWithValidation : createElement,
```

源码：

```js
/**
 * type  html标签名称字符串 或者 组件类名。
 * config 当前节点的props
 * children 子节点（形式：React.createElement()），可以是多个
 **/
function createElement(type, config, children) {
    var propName;

    // Reserved names are extracted
    var props = {};

    var key = null;
    var ref = null;
    var self = null;
    var source = null;

    // 如果节点存在props（属性）
    if (config != null) {
        // 取ref
        if (hasValidRef(config)) {
            ref = config.ref;
        }
        // 取key
        if (hasValidKey(config)) {
            key = "" + config.key;
        }

        self = config.__self === undefined ? null : config.__self;
        source = config.__source === undefined ? null : config.__source;
        // 将节点的properties重新用一个新的对象（props）接收
        for (propName in config) {
            if (
                hasOwnProperty.call(config, propName) &&
                !RESERVED_PROPS.hasOwnProperty(propName)
            ) {
                props[propName] = config[propName];
            }
        }
    }

    // 分情况取children值，分1个子节点和多个子节点
    var childrenLength = arguments.length - 2;
    if (childrenLength === 1) {
        props.children = children;
    } else if (childrenLength > 1) {
        var childArray = Array(childrenLength);
        for (var i = 0; i < childrenLength; i++) {
            childArray[i] = arguments[i + 2];
        }
        if (__DEV__) {
            if (Object.freeze) {
                Object.freeze(childArray);
            }
        }
        props.children = childArray;
    }

    // 节点默认属性赋值逻辑
    if (type && type.defaultProps) {
        var defaultProps = type.defaultProps;
        for (propName in defaultProps) {
            if (props[propName] === undefined) {
                props[propName] = defaultProps[propName];
            }
        }
    }
    if (__DEV__) {
        if (key || ref) {
            if (
                typeof props.$$typeof === "undefined" ||
                props.$$typeof !== REACT_ELEMENT_TYPE
            ) {
                var displayName =
                    typeof type === "function"
                        ? type.displayName || type.name || "Unknown"
                        : type;
                if (key) {
                    defineKeyPropWarningGetter(props, displayName);
                }
                if (ref) {
                    defineRefPropWarningGetter(props, displayName);
                }
            }
        }
    }
    return ReactElement(
        type,
        key,
        ref,
        self,
        source,
        ReactCurrentOwner.current,
        props
    );
}
```

## isValidElement

验证对象是否是一个 React 元素。返回 true 或 false 。

## Children

这个属性提供了处理 this.props.children 的工具。

```js
Children: {
    map,
    forEach,
    count,
    toArray,
    only,
}
```
详情请参阅[React.Children](sections/React.Children.md)


## version

当前React的版本号