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


<!-- todo.md -->

## runBatchedUpdates

会在perform中调用

```js
var updateBatchNumber = 0;

function runBatchedUpdates(transaction) {
    var len = transaction.dirtyComponentsLength;

    // Since reconciling a component higher in the owner hierarchy usually (not
    // always -- see shouldComponentUpdate()) will reconcile children, reconcile
    // them before their children by sorting the array.
    dirtyComponents.sort(mountOrderComparator);

    // Any updates enqueued while reconciling must be performed after this entire
    // batch. Otherwise, if dirtyComponents is [A, B] where A has children B and
    // C, B could update twice in a single batch if C's render enqueues an update
    // to B (since B would have already updated, we should skip it, and the only
    // way we can know to do so is by checking the batch counter).


    // 现在有个dirtyComponents [A,B], A又有子元素A和B，B可能在一次batch中被更新俩次

    // 检查更新的次数，防止一个组件被更新多次
    updateBatchNumber++;

    for (var i = 0; i < len; i++) {
        // If a component is unmounted before pending changes apply, it will still
        // be here, but we assume that it has cleared its _pendingCallbacks and
        // that performUpdateIfNecessary is a noop.
        var component = dirtyComponents[i];

        // If performUpdateIfNecessary happens to enqueue any new updates, we
        // shouldn't execute the callbacks until the next render happens, so
        // stash the callbacks first

        // performUpdateIfNecessary可能会调用callbacks，但是在下一次reander发生之前，我们不应该执行callbacks，所以这里先暂存callbacks
        var callbacks = component._pendingCallbacks;
        component._pendingCallbacks = null;

        var markerName;
        if (ReactFeatureFlags.logTopLevelRenders) {
            var namedComponent = component;
            // Duck type TopLevelWrapper. This is probably always true.
            if (component._currentElement.type.isReactTopLevelWrapper) {
                namedComponent = component._renderedComponent;
            }
            markerName = "React update: " + namedComponent.getName();
            console.time(markerName);
        }

        ReactReconciler.performUpdateIfNecessary(
            component,
            transaction.reconcileTransaction,
            updateBatchNumber
        );

        if (markerName) {
            console.timeEnd(markerName);
        }

        if (callbacks) {
            for (var j = 0; j < callbacks.length; j++) {
                transaction.callbackQueue.enqueue(
                    callbacks[j],
                    component.getPublicInstance()
                );
            }
        }
    }
}
```
