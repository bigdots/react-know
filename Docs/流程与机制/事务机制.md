<!-- TOC -->

- [transaction 机制](#transaction-机制)
    - [transactionWrapper](#transactionwrapper)
    - [transactionWrappers](#transactionwrappers)
    - [运行机制](#运行机制)
    - [react 中的使用](#react-中的使用)

<!-- /TOC -->

# transaction 机制

## transactionWrapper

react 中大量采用了 transaction（事务）机制，它将事务通过 wrapper 进行封装。一个 wrapper 包含一对 initialize 和 close 方法。其结构如下：

```js
transactionWrapper = {
    initialize: function() {},
    close: function() {}
};
```

## transactionWrappers

transactionWrappers 就是多个 transactionWrapper 组成的数组，结构如下：

```js
const transactionWrappers = [
    {
        initialize: function(){},
        close: function(){}
    }
    ....
];
```

## 运行机制

1. 通过调用 transaction.perform 进入
2. 执行 initializeAll 方法，调用所有 initialize 方法
3. 执行 perform 方法中的 callback，最后调用所有 close 方法。
4. 调用 closeAll 方法

initializeAll／closeAll 方法会遍历 transactionWrappers，执行当中每个 transactionWrapper 的 initialize／close 方法。

## react 中的使用

```js
var RESET_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: function() {
    // 将 isBatchingUpdates状态 置为false
    ReactDefaultBatchingStrategy.isBatchingUpdates = false;
  },
};

var FLUSH_BATCHED_UPDATES = {
  initialize: emptyFunction,
  close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates),
};

var flushBatchedUpdates = function() {
    // 循环遍历处理完所有dirtyComponents
    while (dirtyComponents.length || asapEnqueued) {
        if (dirtyComponents.length) {
            var transaction = ReactUpdatesFlushTransaction.getPooled();
            // close前执行完runBatchedUpdates方法，这是关键
            transaction.perform(runBatchedUpdates, null, transaction);
            ReactUpdatesFlushTransaction.release(transaction);
        }

        if (asapEnqueued) {
            asapEnqueued = false;
            var queue = asapCallbackQueue;
            asapCallbackQueue = CallbackQueue.getPooled();
            queue.notifyAll();
            CallbackQueue.release(queue);
        }
    }
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```

初始时注入俩个transaction。

FLUSH_BATCHED_UPDATES： 会在事务开始，处理dirtyComponents，
RESET_BATCHED_UPDATES： 会在事务结束将 isBatchingUpdates状态 置为false。




整个生命周期就是一个 Transaction，在 Transaction 执行期间，componentDidUpdate 方法被推入一个队列中。DOM reconciliation 后，再调用队列中的所有
componentDidUpdate。
