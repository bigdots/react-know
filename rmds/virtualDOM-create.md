
# 创建虚拟DOM: JSX

> 从本质上讲，JSX 只是为 React.createElement函数提供的语法糖。

实例：

```js
// 创建组件
class Welcome extends React.Component {
    render() {
        return <h1>Hello, react</h1>;
    }
}
```

上面的`<h1>Hello, react</h1>`就是jsx，它在编译后，会转化成如下代码：

```js
React.createElement(
    "h1",
    null,
    "Hello, react"
);
```

`createElement` 是React提供的一个创建虚拟DOM的方法。它接收三个参数`type`、 `config`以及`children`，并且返回一个`element`( 虚拟DOM )。我们可以看下下面的源码：

```js
// src/isomorphic/React.js
React.createElement = function(type, config, children){

    var props = {};
    var key = null;
    var ref = null;
    var self = null;
    var source = null;


    //从config中取出key、ref、self、source以及props的各个键值

    return ReactElement
};
```

**所以，使用jsx并不是必须的，你也可以通过直接使用`createElement`来构建应用，但显而易见地，这样操作的成本更高。**


## 虚拟DOM

> 虚拟DOM：就是将页面的 DOM 树的信息使用 JavaScript 对象来表示。

react的虚拟DOM结构：

```js
var ReactElement = {
    $$typeof: REACT_ELEMENT_TYPE,
    type: type,
    key: key,
    ref: ref,
    props: {
        children: ReactElement || [ ReactElement... ],
        ...
    },
    _owner: owner
};
```


