## transcation

```js
mountComponent: function(
    internalInstance,
    transaction,
    hostParent,
    hostContainerInfo,
    context,
    parentDebugID // 0 in production and for roots
) {
    var markup = internalInstance.mountComponent(
        transaction,
        hostParent,
        hostContainerInfo,
        context,
        parentDebugID
    );
    if (
        internalInstance._currentElement &&
        internalInstance._currentElement.ref != null
    ) {
        transaction.getReactMountReady().enqueue(attachRefs, internalInstance);
    }

    return markup;
};
```
