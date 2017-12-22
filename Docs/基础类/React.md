<!-- TOC -->

- [React](#react)
    - [Component & PureComponent](#component--purecomponent)
    - [createElement/cloneElement/createFactory](#createelementcloneelementcreatefactory)
    - [createReactClass](#createreactclass)

<!-- /TOC -->

# React

```js
var React = {
    // Modern
    Children: {
        map: ReactChildren.map,
        forEach: ReactChildren.forEach,
        count: ReactChildren.count,
        toArray: ReactChildren.toArray,
        only: onlyChild
    },

    Component: ReactBaseClasses.Component,
    PureComponent: ReactBaseClasses.PureComponent,

    createElement: createElement,
    cloneElement: cloneElement,
    isValidElement: ReactElement.isValidElement,

    // Classic

    PropTypes: ReactPropTypes,
    createClass: createReactClass,
    createFactory: createFactory,
    createMixin: createMixin,

    // This looks DOM specific but these are actually isomorphic helpers
    // since they are just generating DOM strings.
    DOM: ReactDOMFactories,

    version: ReactVersion,

    // Deprecated hook for JSX spread, don't use this for anything.
    __spread: __spread
};
```

## Component & PureComponent

详见[ReactComponent&PureComponent](ReactComponent&PureComponent.md)

## createElement/cloneElement/createFactory

详见[ReactElement](ReactElement.md)

## createReactClass
`create-react-class` 模块。

