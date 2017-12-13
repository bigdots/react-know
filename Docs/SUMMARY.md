* [API]()
    * [React](./API/React/React.md)
        * [Children]()
        * [Component]()
        * [PureComponent]()
        * [createElement]()
        * [cloneElement]()
        * [isValidElement]()
        * [PropTypes]()
        * [createClass]()
        * [createClass]()
        * [createMixin]()
        * [DOM]()
        * [version]()
        * [__spread]()
    * [ReactDOM]()
        * [render()]()
        * [unmountComponentAtNode()]()
        * [findDOMNode()]()

* [内部重要类]()
    * [ReactMount]()



```js
var React = {
  // Modern

  Children: {
    map: ReactChildren.map,
    forEach: ReactChildren.forEach,
    count: ReactChildren.count,
    toArray: ReactChildren.toArray,
    only: onlyChild,
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
  __spread: __spread,
};
```