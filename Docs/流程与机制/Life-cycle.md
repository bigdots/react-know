
## transaction


```
 *
 *                       wrappers (injected at creation time)
 *                                      +        +
 *                                      |        |
 *                    +-----------------|--------|--------------+
 *                    |                 v        |              |
 *                    |      +---------------+   |              |
 *                    |   +--|    wrapper1   |---|----+         |
 *                    |   |  +---------------+   v    |         |
 *                    |   |          +-------------+  |         |
 *                    |   |     +----|   wrapper2  |--------+   |
 *                    |   |     |    +-------------+  |     |   |
 *                    |   |     |                     |     |   |
 *                    |   v     v                     v     v   | wrapper
 *                    | +---+ +---+   +---------+   +---+ +---+ | invariants
 * perform(anyMethod) | |   | |   |   |         |   |   | |   | | maintained
 * +----------------->|-|---|-|---|-->|anyMethod|---|---|-|---|-|-------->
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | |   | |   |   |         |   |   | |   | |
 *                    | +---+ +---+   +---------+   +---+ +---+ |
 *                    |  initialize                    close    |
 *                    +-----------------------------------------+
 *
```


```js
var TransactionImpl = {
    reinitializeTransaction: function(){

    },

    // 返回一个transaction wrappers数组
    getTransactionWrappers,

    perform: function(){

    },

    _isInTransaction,

    initializeAll: function(){

    },

    closeAll: function(){

    }
}
```




```js
var transaction = new ReactDefaultBatchingStrategyTransaction();

function ReactDefaultBatchingStrategyTransaction() {
    this.reinitializeTransaction();
}

var RESET_BATCHED_UPDATES = {
    initialize: emptyFunction,
    close: function() {
        ReactDefaultBatchingStrategy.isBatchingUpdates = false;
    }
};

var FLUSH_BATCHED_UPDATES = {
    initialize: emptyFunction,
    close: ReactUpdates.flushBatchedUpdates.bind(ReactUpdates)
};

var TRANSACTION_WRAPPERS = [FLUSH_BATCHED_UPDATES, RESET_BATCHED_UPDATES];
```

```js
perform : function(method, scope, a, b, c, d, e, f) {
    var errorThrown;
    var ret;
    try {
        this._isInTransaction = true;
        // Catching errors makes debugging more difficult, so we start with
        // errorThrown set to true before setting it to false after calling
        // close -- if it's still set to true in the finally block, it means
        // one of these calls threw.
        errorThrown = true;

        // 运行所有wrapper中的initialize方法
        this.initializeAll(0);
        // 执行perform方法传入的callback
        ret = method.call(scope, a, b, c, d, e, f);
        errorThrown = false;
    } finally {
        try {
            if (errorThrown) {
                // If `method` throws, prefer to show that stack trace over any thrown
                // by invoking `closeAll`.
                try {
                    //运行wrapper中的所有close方法
                    this.closeAll(0);
                } catch (err) {}
            } else {
                // Since `method` didn't throw, we don't want to silence the exception
                // here.
                this.closeAll(0);
            }
        } finally {
            this._isInTransaction = false;
        }
    }
    return ret;
};
```

```js
initializeAll : function(startIndex: number): void {
    var transactionWrappers = this.transactionWrappers;
     // 遍历所有注册的wrapper
    for (var i = startIndex; i < transactionWrappers.length; i++) {
        var wrapper = transactionWrappers[i];
        try {
            // Catching errors makes debugging more difficult, so we start with the
            // OBSERVED_ERROR state before overwriting it with the real return value
            // of initialize -- if it's still set to OBSERVED_ERROR in the finally
            // block, it means wrapper.initialize threw.
            this.wrapperInitData[i] = OBSERVED_ERROR;
            // 调用wrapper的initialize方法
            this.wrapperInitData[i] = wrapper.initialize
                ? wrapper.initialize.call(this)
                : null;
        } finally {
            if (this.wrapperInitData[i] === OBSERVED_ERROR) {
                // The initializer for wrapper i threw an error; initialize the
                // remaining wrappers but silence any exceptions from them to ensure
                // that the first error is the one to bubble up.
                try {
                    this.initializeAll(i + 1);
                } catch (err) {}
            }
        }
    }
};
```

```js
/**
 * Invokes each of `this.transactionWrappers.close[i]` functions, passing into
 * them the respective return values of `this.transactionWrappers.init[i]`
 * (`close`rs that correspond to initializers that failed will not be
 * invoked).
 */
closeAll : function(startIndex: number): void {
    // invariant(
    //     this.isInTransaction(),
    //     "Transaction.closeAll(): Cannot close transaction when none are open."
    // );
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
        var wrapper = transactionWrappers[i];
        var initData = this.wrapperInitData[i];
        var errorThrown;
        try {
            // Catching errors makes debugging more difficult, so we start with
            // errorThrown set to true before setting it to false after calling
            // close -- if it's still set to true in the finally block, it means
            // wrapper.close threw.
            errorThrown = true;
            if (initData !== OBSERVED_ERROR && wrapper.close) {
                // 调用wrapper的close方法
                wrapper.close.call(this, initData);
            }
            errorThrown = false;
        } finally {
            if (errorThrown) {
                // The closer for wrapper i threw an error; close the remaining
                // wrappers but silence any exceptions from them to ensure that the
                // first error is the one to bubble up.
                try {
                    this.closeAll(i + 1);
                } catch (e) {}
            }
        }
    }
    this.wrapperInitData.length = 0;
};
```



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

























