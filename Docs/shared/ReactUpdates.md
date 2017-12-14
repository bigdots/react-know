# ReactUpdates

```js
var ReactUpdates = {
    /**
     * React references `ReactReconcileTransaction` using this property in order
     * to allow dependency injection.
     *
     * @internal
     */
    ReactReconcileTransaction: null,

    batchedUpdates: batchedUpdates,
    enqueueUpdate: enqueueUpdate,
    flushBatchedUpdates: flushBatchedUpdates,
    injection: ReactUpdatesInjection,
    asap: asap
};
```

## enqueueUpdate

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

## flushBatchedUpdates

```js
var flushBatchedUpdates = function() {
    // ReactUpdatesFlushTransaction's wrappers will clear the dirtyComponents
    // array and perform any updates enqueued by mount-ready handlers (i.e.,
    // componentDidUpdate) but we need to check here too in order to catch
    // updates enqueued by setState callbacks and asap calls.

    // 会清空dirtyComponents 数组，
    while (dirtyComponents.length || asapEnqueued) {
        if (dirtyComponents.length) {
            var transaction = ReactUpdatesFlushTransaction.getPooled();
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
```

## ReactUpdatesFlushTransaction

```js
function ReactUpdatesFlushTransaction() {
    this.reinitializeTransaction();
    this.dirtyComponentsLength = null;
    this.callbackQueue = CallbackQueue.getPooled();
    this.reconcileTransaction = ReactUpdates.ReactReconcileTransaction.getPooled(
        /* useCreateElement */ true
    );
}

Object.assign(ReactUpdatesFlushTransaction.prototype, Transaction, {
    getTransactionWrappers: function() {
        return TRANSACTION_WRAPPERS;
    },

    destructor: function() {
        this.dirtyComponentsLength = null;
        CallbackQueue.release(this.callbackQueue);
        this.callbackQueue = null;
        ReactUpdates.ReactReconcileTransaction.release(
            this.reconcileTransaction
        );
        this.reconcileTransaction = null;
    },

    perform: function(method, scope, a) {
        // Essentially calls `this.reconcileTransaction.perform(method, scope, a)`
        // with this transaction's wrappers around it.
        return Transaction.perform.call(
            this,
            this.reconcileTransaction.perform,
            this.reconcileTransaction,
            method,
            scope,
            a
        );
    }
});
```
