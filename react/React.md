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

## Component ： Class

详情请参阅[React.Children](sections/React.Children.md)

## PureComponent： Class

与 React.Component 几乎完全相同，但 PureComponent 通过 prop 和 state 的浅对比来
实现 shouldComponentUpate()。

它会忽略整个组件的子级。请确保所有的子级组件  也是 ”Pure” 的。

## unstable_AsyncComponent: Class

实验特性

```js
unstable_AsyncComponent: AsyncComponent;
```

## createElement: functon

这个方法平常写 react 代码的时候可能不怎么需要用到，但其实它被使用地十分频繁。我
们用 `JSX` 编写的代码会被转换成用这个方法来实现。这个方法会创建**虚拟 DOM**。

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

```js
/**
 * 创建react节点的工厂函数. This no longer adheres to
 * the class pattern, so do not use new to call it. Also, no instanceof check
 * will work. Instead test $$typeof field against Symbol.for('react.element') to check
 * if something is a React Element.
 *
 * @param {*} type
 * @param {*} key
 * @param {string|object} ref
 * @param {*} self 一个临时解决方案：当调用React.createElement时，this的指向，
 * 使用箭头函数
 * A *temporary* helper to detect places where `this` is
 * different from the `owner` when React.createElement is called, so that we
 * can warn. We want to get rid of owner and replace string `ref`s with arrow
 * functions, and as long as `this` and owner are the same, there will be no
 * change in behavior.
 * @param {*} source 辅助性说明对象
 * An annotation object (added by a transpiler or otherwise)
 * indicating filename, line number, and/or other information.
 * @param {*} owner
 * @param {*} props
 * @internal
 */
var ReactElement = function(type, key, ref, self, source, owner, props) {
    var element = {
        // This tag allow us to uniquely identify this as a React Element
        $$typeof: REACT_ELEMENT_TYPE,

        // Built-in properties that belong on the element
        type: type,
        key: key,
        ref: ref,
        props: props,

        // Record the component responsible for creating this element.
        _owner: owner
    };

    if (__DEV__) {
        // The validation flag is currently mutative. We put it on
        // an external backing store so that we can freeze the whole object.
        // This can be replaced with a WeakMap once they are implemented in
        // commonly used development environments.
        element._store = {};

        // To make comparing ReactElements easier for testing purposes, we make
        // the validation flag non-enumerable (where possible, which should
        // include every environment we run tests in), so the test framework
        // ignores it.
        Object.defineProperty(element._store, "validated", {
            configurable: false,
            enumerable: false,
            writable: true,
            value: false
        });
        // self and source are DEV only properties.
        Object.defineProperty(element, "_self", {
            configurable: false,
            enumerable: false,
            writable: false,
            value: self
        });
        // Two elements created in two different places should be considered
        // equal for testing purposes and therefore we hide it from enumeration.
        Object.defineProperty(element, "_source", {
            configurable: false,
            enumerable: false,
            writable: false,
            value: source
        });
        if (Object.freeze) {
            Object.freeze(element.props);
            Object.freeze(element);
        }
    }

    return element;
};
```

```js
const hasSymbol = typeof Symbol === "function" && Symbol.for;

export const REACT_ELEMENT_TYPE = hasSymbol
    ? Symbol.for("react.element")
    : 0xeac7;
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

当前 React 的版本号
