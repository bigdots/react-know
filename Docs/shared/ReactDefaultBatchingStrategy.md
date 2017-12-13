# ReactDefaultBatchingStrategy

```js
var ReactDefaultBatchingStrategy = {
    isBatchingUpdates: false,

    batchedUpdates: function(callback, a, b, c, d, e) {}
};
```

## batchedUpdates

```js
/**
 * Call the provided function in a context within which calls to `setState`
 * and friends are batched such that components aren't updated unnecessarily.
 */
batchedUpdates = function(callback, a, b, c, d, e) {
    // 获取当前的isBatchingUpdates值
    var alreadyBatchingUpdates = ReactDefaultBatchingStrategy.isBatchingUpdates;

    // 将isBatchingUpdates设为true，表明正在更新
    ReactDefaultBatchingStrategy.isBatchingUpdates = true;

    // 避免重复更新
    if (alreadyBatchingUpdates) {
        return callback(a, b, c, d, e);
    } else {
        // 执行更新
        return transaction.perform(callback, null, a, b, c, d, e);
    }
};
```
