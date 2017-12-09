## update

```js
ReactComponent.prototype.setState = function(partialState, callback) {
    // 放入事物队列
    this.updater.enqueueSetState(this, partialState);
    if (callback) {
        this.updater.enqueueCallback(this, callback, "setState");
    }
};
```

```js
/**
 * Sets a subset of the state. This only exists because _pendingState is
 * internal. This provides a merging strategy that is not available to deep
 * properties which is confusing. TODO: Expose pendingState or don't use it
 * during the merge.
 *
 * @param {ReactClass} publicInstance 组件实例.
 * @param {object} partialState 要被合并到state的数据.
 * @internal
 */
enqueueSetState : function(publicInstance, partialState) {

    // 根据实例this获取组件
    var internalInstance = getInternalInstanceReadyForUpdate(
        publicInstance,
        "setState"
    );

    if (!internalInstance) {
        return;
    }

    // 如果_pendingStateQueue为空,则创建它。队列是数组形式的
    var queue =
        internalInstance._pendingStateQueue ||
        (internalInstance._pendingStateQueue = []);
    queue.push(partialState);

    // 将要更新的ReactComponent放入数组中
    enqueueUpdate(internalInstance);
};
```

```js
function getInternalInstanceReadyForUpdate(publicInstance, callerName) {
    // 从ReactInstanceMap取出ReactComponent组件， 这个组件是 react在 mount 的时候存入ReactInstanceMap的
    var internalInstance = ReactInstanceMap.get(publicInstance);
    if (!internalInstance) {
        return null;
    }
    return internalInstance;
}
```

```js
/**
 * Mark a component as needing a rerender, adding an optional callback to a
 * list of functions which will be executed once the rerender occurs.
 */
function enqueueUpdate(component) {
    ensureInjected();

    // Various parts of our code (such as ReactCompositeComponent's
    // _renderValidatedComponent) assume that calls to render aren't nested;
    // verify that that's the case. (This is called by each top-level update
    // function, like setState, forceUpdate, etc.; creation and
    // destruction of top-level components is guarded in ReactMount.)

    // 如果不是正处于创建或更新组件阶段,则处理update事务
    if (!batchingStrategy.isBatchingUpdates) {
        batchingStrategy.batchedUpdates(enqueueUpdate, component);
        return;
    }

    // 如果正在创建或更新组件,则暂且先不处理update,只是将组件放在dirtyComponents数组中
    dirtyComponents.push(component);
    if (component._updateBatchNumber == null) {
        component._updateBatchNumber = updateBatchNumber + 1;
    }
}
```

```js
var ReactDefaultBatchingStrategy = {
    isBatchingUpdates: false,

    /**
     * Call the provided function in a context within which calls to `setState`
     * and friends are batched such that components aren't updated unnecessarily.
     */
    batchedUpdates: function(callback, a, b, c, d, e) {
        var alreadyBatchingUpdates =
            ReactDefaultBatchingStrategy.isBatchingUpdates;

        // 批处理最开始时，将isBatchingUpdates设为true，表示正在更新
        ReactDefaultBatchingStrategy.isBatchingUpdates = true;

        // The code is written this way to avoid extra allocations
        // 避免额外的配置
        if (alreadyBatchingUpdates) {
            return callback(a, b, c, d, e);
        } else {
            return transaction.perform(callback, null, a, b, c, d, e);
        }
    }
};
```

## transaction

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
    invariant(
        this.isInTransaction(),
        "Transaction.closeAll(): Cannot close transaction when none are open."
    );
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
