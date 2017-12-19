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

## transaction的获取

```js
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  },
};

var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];


// 在ReactDefaultBatchingStrategyTransaction的原型上加上Transaction的属性和getTransactionWrappers方法
Object.assign(ReactDefaultBatchingStrategyTransaction.prototype, Transaction, {
  getTransactionWrappers: function() {
    return TRANSACTION_WRAPPERS;
  }
});
var transaction = new ReactDefaultBatchingStrategyTransaction();

```
