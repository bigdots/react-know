
## 第一步：声明组件并继承 Component 类

### Component 类

```js
function Component(props, context, updater) {
    this.props = props;
    this.context = context;
    this.refs = emptyObject;
    this.updater = updater || ReactNoopUpdateQueue;
}
```

## 第三步：执行 ReactDOM.render，渲染虚拟 DOM

```js
render(element,container,callback) {
    return renderSubtreeIntoContainer(
        null, //父节点
        element, // 要渲染的React节点
        container, // 容器节点，必须是DOM节点: document.getElementById("root")
        false,
        callback // 回调函数
    );
}


/***
 *
 * */
renderSubtreeIntoContainer = (parentComponent,children,container,forceHydrate,callback)=>{
    let root = container._reactRootContainer;
    if (!root) {
        const shouldHydrate = forceHydrate || shouldHydrateDueToLegacyHeuristic(container);
        const newRoot = DOMRenderer.createContainer(container, shouldHydrate);
        root = container._reactRootContainer = newRoot;
        DOMRenderer.unbatchedUpdates(() => {
            DOMRenderer.updateContainer(children, newRoot, parentComponent, callback);
        });
    }else{
        DOMRenderer.updateContainer(children, root, parentComponent, callback);
    }
    //返回一个引用， 指向 ReactComponent的根实例
    return DOMRenderer.getPublicRootInstance(root);
}
```

```js
getPublicRootInstance(container){
    const containerFiber = container.current;
    if (!containerFiber.child) {
    return null;
    }
    switch (containerFiber.child.tag) {
    case HostComponent:
        return getPublicInstance(containerFiber.child.stateNode);
    default:
        return containerFiber.child.stateNode;
    }
}
```
