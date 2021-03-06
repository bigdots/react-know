# ReactUpdateQueue

<!-- TOC -->

- [ReactUpdateQueue](#reactupdatequeue)
    - [getInternalInstanceReadyForUpdate](#getinternalinstancereadyforupdate)

<!-- /TOC -->

为组件提供了一个updater对象。处理upfate事宜。

```js
// /src/renderers/shared/stack/reconciler/ReactUpdateQueue.js

var ReactUpdateQueue = {
    /**
     * Checks whether or not this composite component is mounted.
     * @param {ReactClass} publicInstance The instance we want to test.
     * @return {boolean} True if mounted, false otherwise.
     * @protected
     * @final
     */
    isMounted: function(publicInstance) {
        var internalInstance = ReactInstanceMap.get(publicInstance);
        if (internalInstance) {
            // During componentWillMount and render this will still be null but after
            // that will always render to something. At least for now. So we can use
            // this hack.
            return !!internalInstance._renderedComponent;
        } else {
            return false;
        }
    },

    /**
     * Enqueue a callback that will be executed after all the pending updates
     * have processed.
     *
     * @param {ReactClass} publicInstance The instance to use as `this` context.
     * @param {?function} callback Called after state is updated.
     * @param {string} callerName Name of the calling function in the public API.
     * @internal
     */
    enqueueCallback: function(publicInstance, callback, callerName) {
        ReactUpdateQueue.validateCallback(callback, callerName);
        var internalInstance = getInternalInstanceReadyForUpdate(
            publicInstance
        );

        // Previously we would throw an error if we didn't have an internal
        // instance. Since we want to make it a no-op instead, we mirror the same
        // behavior we have in other enqueue* methods.
        // We also need to ignore callbacks in componentWillMount. See
        // enqueueUpdates.
        if (!internalInstance) {
            return null;
        }

        // 将回调函数推入栈
        if (internalInstance._pendingCallbacks) {
            internalInstance._pendingCallbacks.push(callback);
        } else {
            internalInstance._pendingCallbacks = [callback];
        }
        // TODO: The callback here is ignored when setState is called from
        // componentWillMount. Either fix it or disallow doing so completely in
        // favor of getInitialState. Alternatively, we can disallow
        // componentWillMount during server-side rendering.
        enqueueUpdate(internalInstance);
    },

    enqueueCallbackInternal: function(internalInstance, callback) {
        if (internalInstance._pendingCallbacks) {
            internalInstance._pendingCallbacks.push(callback);
        } else {
            internalInstance._pendingCallbacks = [callback];
        }
        enqueueUpdate(internalInstance);
    },

    /**
     * Forces an update. This should only be invoked when it is known with
     * certainty that we are **not** in a DOM transaction.
     *
     * You may want to call this when you know that some deeper aspect of the
     * component's state has changed but `setState` was not called.
     *
     * This will not invoke `shouldComponentUpdate`, but it will invoke
     * `componentWillUpdate` and `componentDidUpdate`.
     *
     * @param {ReactClass} publicInstance The instance that should rerender.
     * @internal
     */
    enqueueForceUpdate: function(publicInstance) {
        var internalInstance = getInternalInstanceReadyForUpdate(
            publicInstance,
            "forceUpdate"
        );

        if (!internalInstance) {
            return;
        }

        internalInstance._pendingForceUpdate = true;

        enqueueUpdate(internalInstance);
    },

    /**
     * Replaces all of the state. Always use this or `setState` to mutate state.
     * You should treat `this.state` as immutable.
     *
     * There is no guarantee that `this.state` will be immediately updated, so
     * accessing `this.state` after calling this method may return the old value.
     *
     * @param {ReactClass} publicInstance The instance that should rerender.
     * @param {object} completeState Next state.
     * @internal
     */
    enqueueReplaceState: function(publicInstance, completeState, callback) {
        var internalInstance = getInternalInstanceReadyForUpdate(
            publicInstance,
            "replaceState"
        );

        if (!internalInstance) {
            return;
        }

        internalInstance._pendingStateQueue = [completeState];
        internalInstance._pendingReplaceState = true;

        // Future-proof 15.5
        if (callback !== undefined && callback !== null) {
            ReactUpdateQueue.validateCallback(callback, "replaceState");
            if (internalInstance._pendingCallbacks) {
                internalInstance._pendingCallbacks.push(callback);
            } else {
                internalInstance._pendingCallbacks = [callback];
            }
        }

        enqueueUpdate(internalInstance);
    },

    /**
     * Sets a subset of the state. This only exists because _pendingState is
     * internal. This provides a merging strategy that is not available to deep
     * properties which is confusing. TODO: Expose pendingState or don't use it
     * during the merge.
     *
     * @param {ReactClass} publicInstance The instance that should rerender.
     * @param {object} partialState Next partial state to be merged with state.
     * @internal
     */
    enqueueSetState: function(publicInstance, partialState) {
        if (__DEV__) {
            ReactInstrumentation.debugTool.onSetState();
            warning(
                partialState != null,
                "setState(...): You passed an undefined or null state object; " +
                    "instead, use forceUpdate()."
            );
        }

        var internalInstance = getInternalInstanceReadyForUpdate(
            publicInstance,
            "setState"
        );

        if (!internalInstance) {
            return;
        }

        var queue =
            internalInstance._pendingStateQueue ||
            (internalInstance._pendingStateQueue = []);
        queue.push(partialState);

        enqueueUpdate(internalInstance);
    },

    enqueueElementInternal: function(
        internalInstance,
        nextElement,
        nextContext
    ) {
        internalInstance._pendingElement = nextElement;
        // TODO: introduce _pendingContext instead of setting it directly.
        internalInstance._context = nextContext;
        enqueueUpdate(internalInstance);
    },

    validateCallback: function(callback, callerName) {
        invariant(
            !callback || typeof callback === "function",
            "%s(...): Expected the last optional `callback` argument to be a " +
                "function. Instead received: %s.",
            callerName,
            formatUnexpectedArgument(callback)
        );
    }
};

module.exports = ReactUpdateQueue;


```

## getInternalInstanceReadyForUpdate

```js
// 从ReactInstanceMap取出实例
function getInternalInstanceReadyForUpdate(publicInstance, callerName) {
    var internalInstance = ReactInstanceMap.get(publicInstance);

    // 实例不存在则返回null
    if (!internalInstance) {
        return null;
    }

    return internalInstance;
}
```
