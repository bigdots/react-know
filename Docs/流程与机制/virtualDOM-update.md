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




1. enqueueSetState将state放入队列中，并调用enqueueUpdate处理要更新的Component
2. 如果组件当前正处于update事务中，则先将Component存入dirtyComponent中。否则调用batchedUpdates处理。
3. batchedUpdates发起一次transaction.perform()事务
4. 开始执行事务初始化，运行，结束三个阶段
    初始化：事务初始化阶段没有注册方法，故无方法要执行
    运行：执行setSate时传入的callback方法，一般不会传callback参数
    结束：更新isBatchingUpdates为false，并执行FLUSH_BATCHED_UPDATES这个wrapper中的close方法
5. FLUSH_BATCHED_UPDATES在close阶段，会循环遍历所有的dirtyComponents，调用updateComponent刷新组件，并执行它的pendingCallbacks, 也就是setState中设置的callback。