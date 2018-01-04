<!-- TOC -->

- [ReactCompositeComponent](#reactcompositecomponent)
    - [_processPendingState](#_processpendingstate)
    - [_renderValidatedComponentWithoutOwnerOrContext](#_rendervalidatedcomponentwithoutownerorcontext)
    - [_renderValidatedComponent](#_rendervalidatedcomponent)
    - [performUpdateIfNecessary](#performupdateifnecessary)
    - [receiveComponent](#receivecomponent)
    - [updateComponent](#updatecomponent)
    - [_performComponentUpdate](#_performcomponentupdate)
    - [_updateRenderedComponent](#_updaterenderedcomponent)
    - [mountComponent](#mountcomponent)
    - [performInitialMount](#performinitialmount)
    - [unmountComponent](#unmountcomponent)
    - [attachRef](#attachref)
    - [detachRef](#detachref)

<!-- /TOC -->
# ReactCompositeComponent

`ReactCompositeComponent`是React自定义类型组件。

```js
var ReactCompositeComponent = {
    /**
     * Base constructor for all composite component.
     *
     * @param {ReactElement} element
     * @final
     * @internal
     */
    construct: function(element) {
        this._currentElement = element;
        this._rootNodeID = 0;
        this._compositeType = null;
        this._instance = null;
        this._hostParent = null;
        this._hostContainerInfo = null;

        // See ReactUpdateQueue
        this._updateBatchNumber = null;
        this._pendingElement = null;
        this._pendingStateQueue = null;
        this._pendingReplaceState = false;
        this._pendingForceUpdate = false;

        this._renderedNodeType = null;
        this._renderedComponent = null;
        this._context = null;
        this._mountOrder = 0;
        this._topLevelWrapper = null;

        // See ReactUpdates and ReactUpdateQueue.
        this._pendingCallbacks = null;

        // ComponentWillUnmount shall only be called once
        this._calledComponentWillUnmount = false;
    },

    mountComponent: function() {},

    _constructComponent: function(
        doConstruct,
        publicProps,
        publicContext,
        updateQueue
    ) {
        return this._constructComponentWithoutOwner(
            doConstruct,
            publicProps,
            publicContext,
            updateQueue
        );
    },

    _constructComponentWithoutOwner: function(
        doConstruct,
        publicProps,
        publicContext,
        updateQueue
    ) {
        var Component = this._currentElement.type;

        if (doConstruct) {
            return new Component(publicProps, publicContext, updateQueue);
        }

        // This can still be an instance in case of factory components
        // but we'll count this as time spent rendering as the more common case.
        return Component(publicProps, publicContext, updateQueue);
    },

    performInitialMountWithErrorHandling: function() {},

    performInitialMount: function() {},  // 执行mountComponent的渲染阶段，会调用到instantiateReactComponent，从而进入初始化React组件的入口

    getHostNode: function() {
        return ReactReconciler.getHostNode(this._renderedComponent);
    },

    unmountComponent: function() {},  // 卸载组件，内存释放等工作

    /**
     * Filters the context object to only contain keys specified in
     * `contextTypes`
     *
     * @param {object} context
     * @return {?object}
     * @private
     */
    _maskContext: function(context) {
        var Component = this._currentElement.type;
        var contextTypes = Component.contextTypes;
        if (!contextTypes) {
            return emptyObject;
        }
        var maskedContext = {};
        for (var contextName in contextTypes) {
            maskedContext[contextName] = context[contextName];
        }
        return maskedContext;
    },

    /**
     * Filters the context object to only contain keys specified in
     * `contextTypes`, and asserts that they are valid.
     *
     * @param {object} context
     * @return {?object}
     * @private
     */
    _processContext: function(context) {
        var maskedContext = this._maskContext(context);
        return maskedContext;
    },

    /**
     * @param {object} currentContext
     * @return {object}
     * @private
     */
    _processChildContext: function(currentContext) {
        var Component = this._currentElement.type;
        var inst = this._instance;
        var childContext;

        if (inst.getChildContext) {
            childContext = inst.getChildContext();
        }

        if (childContext) {
            return Object.assign({}, currentContext, childContext);
        }
        return currentContext;
    },

    /**
     * Assert that the context types are valid
     *
     * @param {object} typeSpecs Map of context field to a ReactPropType
     * @param {object} values Runtime values that need to be type-checked
     * @param {string} location e.g. "prop", "context", "child context"
     * @private
     */
    _checkContextTypes: function(
        typeSpecs,
        values,
        location: ReactPropTypeLocations
    ) {
        if (__DEV__) {
            checkReactTypeSpec(
                typeSpecs,
                values,
                location,
                this.getName(),
                null,
                this._debugID
            );
        }
    },

    receiveComponent: function() {},

    performUpdateIfNecessary: function() {},

    updateComponent: function() {},  // setState后被调用，重新渲染组件

    _processPendingState: function() {},

    _performComponentUpdate: function() {},

    _updateRenderedComponent: function() {},

    /**
     * Overridden in shallow rendering.
     *
     * @protected
     */
    _replaceNodeWithMarkup: function(oldHostNode, nextMarkup, prevInstance) {
        ReactComponentEnvironment.replaceNodeWithMarkup(
            oldHostNode,
            nextMarkup,
            prevInstance
        );
    },

    /**
     * @protected
     */
    _renderValidatedComponentWithoutOwnerOrContext: function() {},

    /**
     * @private
     */
    _renderValidatedComponent: function() {},

    attachRef: function() {},  // 将ref指向组件对象

    detachRef: function() {}, // 将组件的引用从全局对象refs中删掉，

    /**
     * Get a text description of the component that can be used to identify it
     * in error messages.
     * @return {string} The name or null.
     * @internal
     */
    getName: function() {
        var type = this._currentElement.type;
        var constructor = this._instance && this._instance.constructor;
        return (
            type.displayName ||
            (constructor && constructor.displayName) ||
            type.name ||
            (constructor && constructor.name) ||
            null
        );
    },

    /**
     * Get the publicly accessible representation of this component - i.e. what
     * is exposed by refs and returned by render. Can be null for stateless
     * components.
     *
     * @return {ReactComponent} the public component instance.
     * @internal
     */
    getPublicInstance: function() {
        var inst = this._instance;
        if (this._compositeType === CompositeTypes.StatelessFunctional) {
            return null;
        }
        return inst;
    },

    // Stub
    _instantiateReactComponent: null
};
```

## _processPendingState

处理`_pendingStateQueue`队列中的数据，返回新的 state。

```js
_processPendingState = function(props, context) {
    var inst = this._instance;
    var queue = this._pendingStateQueue;
    var replace = this._pendingReplaceState;
    this._pendingReplaceState = false;
    this._pendingStateQueue = null;

    // 如果不存在需要更新的state，直接返回原有的state
    if (!queue) {
        return inst.state;
    }

    // 如果是替换操作，之间返回state队列第一个
    if (replace && queue.length === 1) {
        return queue[0];
    }

    // 遍历队列进行state合并
    var nextState = Object.assign({}, replace ? queue[0] : inst.state);
    for (var i = replace ? 1 : 0; i < queue.length; i++) {
        var partial = queue[i];
        Object.assign(
            nextState,
            typeof partial === "function"
                ? partial.call(inst, nextState, props, context)
                : partial
        );
    }

    return nextState;
};
```

## _renderValidatedComponentWithoutOwnerOrContext

调用 `render` 方法

```js
// 调用当前组件实例的render函数，获得ReactElement并返回
_renderValidatedComponentWithoutOwnerOrContext: function() {
    var inst = this._instance;
    var renderedElement;

    // 调用生命周期函数
    renderedElement = inst.render();

    return renderedElement;
}
```

## _renderValidatedComponent

```js
_renderValidatedComponent = function() {
    var renderedElement;
    renderedElement = this._renderValidatedComponentWithoutOwnerOrContext();

    return renderedElement;
};
```

## performUpdateIfNecessary

```js
/**
 * If any of `_pendingElement`, `_pendingStateQueue`, or `_pendingForceUpdate`
 * is set, update the component.
 *
 * @param {ReactReconcileTransaction} transaction
 * @internal
 */
performUpdateIfNecessary: function(transaction) {
    if (this._pendingElement != null) {
        ReactReconciler.receiveComponent(
            this,
            this._pendingElement,
            transaction,
            this._context
        );
    } else if (
        this._pendingStateQueue !== null ||
        this._pendingForceUpdate
    ) {
        this.updateComponent(
            transaction,
            this._currentElement,
            this._currentElement,
            this._context,
            this._context
        );
    } else {
        this._updateBatchNumber = null;
    }
}
```


## receiveComponent

```js
receiveComponent: function(nextElement, transaction, nextContext) {
    var prevElement = this._currentElement;
    var prevContext = this._context;

    this._pendingElement = null;

    this.updateComponent(
        transaction,
        prevElement,
        nextElement,
        prevContext,
        nextContext
    );
}
```

## updateComponent

调用`componentWillReceiveProps` 和 `shouldComponentUpdate` 方法。

```js
/**
 * Perform an update to a mounted component. The componentWillReceiveProps and
 * shouldComponentUpdate methods are called, then (assuming the update isn't
 * skipped) the remaining update lifecycle methods are called and the DOM
 * representation is updated.
 *
 * By default, this implements React's rendering and reconciliation algorithm.
 * Sophisticated clients may wish to override this.
 *
 * @param {ReactReconcileTransaction} transaction
 * @param {ReactElement} prevParentElement
 * @param {ReactElement} nextParentElement
 * @internal
 * @overridable
 */
updateComponent: function(
    transaction,
    prevParentElement,
    nextParentElement,
    prevUnmaskedContext,
    nextUnmaskedContext
) {
    var inst = this._instance;

    var willReceive = false;
    var nextContext;

    // 判断context是否已经改变
    if (this._context === nextUnmaskedContext) {
        nextContext = inst.context;
    } else {
        nextContext = this._processContext(nextUnmaskedContext);
        willReceive = true;
    }

    var prevProps = prevParentElement.props;
    var nextProps = nextParentElement.props;

    // 不是简单的state更新，而是props更新
    if (prevParentElement !== nextParentElement) {
        willReceive = true;
    }

    // An update here will schedule an update but immediately set
    // _pendingStateQueue which will ensure that any state updates gets
    // immediately reconciled instead of waiting for the next batch.

    // 调用生命周期函数
    if (willReceive && inst.componentWillReceiveProps) {
        inst.componentWillReceiveProps(nextProps, nextContext);
    }

    var nextState = this._processPendingState(nextProps, nextContext);
    var shouldUpdate = true;

    if (!this._pendingForceUpdate) {
        if (inst.shouldComponentUpdate) {
            // 调用生命周期函数
            shouldUpdate = inst.shouldComponentUpdate(
                nextProps,
                nextState,
                nextContext
            );
        } else {
            if (this._compositeType === CompositeTypes.PureClass) {
                shouldUpdate =
                    !shallowEqual(prevProps, nextProps) ||
                    !shallowEqual(inst.state, nextState);
            }
        }
    }

    this._updateBatchNumber = null;
    if (shouldUpdate) {
        this._pendingForceUpdate = false;
        // Will set `this.props`, `this.state` and `this.context`.
        this._performComponentUpdate(
            nextParentElement,
            nextProps,
            nextState,
            nextContext,
            transaction,
            nextUnmaskedContext
        );
    } else {
        // If it's determined that a component should not update, we still want
        // to set props and state but we shortcut the rest of the update.
        this._currentElement = nextParentElement;
        this._context = nextUnmaskedContext;
        inst.props = nextProps;
        inst.state = nextState;
        inst.context = nextContext;
    }
}
```


## _performComponentUpdate

会调用 `componentWillUpdate` 和 `componentDidUpdate`。

```js
/**
 * Merges new props and state, notifies delegate methods of update and
 * performs update.
 *
 * @param {ReactElement} nextElement Next element
 * @param {object} nextProps Next public object to set as properties.
 * @param {?object} nextState Next object to set as state.
 * @param {?object} nextContext Next public object to set as context.
 * @param {ReactReconcileTransaction} transaction
 * @param {?object} unmaskedContext
 * @private
 */
_performComponentUpdate: function(
    nextElement,
    nextProps,
    nextState,
    nextContext,
    transaction,
    unmaskedContext
) {
    var inst = this._instance;

    var hasComponentDidUpdate = Boolean(inst.componentDidUpdate);
    var prevProps;
    var prevState;
    var prevContext;
    if (hasComponentDidUpdate) {
        prevProps = inst.props;
        prevState = inst.state;
        prevContext = inst.context;
    }

    if (inst.componentWillUpdate) {
        // 调用生命周期函数
        inst.componentWillUpdate(nextProps, nextState, nextContext);
    }

    this._currentElement = nextElement;
    this._context = unmaskedContext;
    inst.props = nextProps;
    inst.state = nextState;
    inst.context = nextContext;

    // 此处会调用render
    this._updateRenderedComponent(transaction, unmaskedContext);

    if (hasComponentDidUpdate) {
        transaction
            .getReactMountReady()
            .enqueue(
                // 调用生命周期函数
                inst.componentDidUpdate.bind(
                    inst,
                    prevProps,
                    prevState,
                    prevContext
                ),
                inst
            );
    }
}
```

## _updateRenderedComponent

更新相应的DOM。this._renderValidatedComponent()会调用生命周期`render`方法

```js
/**
 * Call the component's `render` method and update the DOM accordingly.
 *
 * @param {ReactReconcileTransaction} transaction
 * @internal
 */
_updateRenderedComponent: function(transaction, context) {
    var prevComponentInstance = this._renderedComponent;
    var prevRenderedElement = prevComponentInstance._currentElement;
    var nextRenderedElement = this._renderValidatedComponent();

    var debugID = 0;

    if (
        shouldUpdateReactComponent(prevRenderedElement, nextRenderedElement)
    ) {
        ReactReconciler.receiveComponent(
            prevComponentInstance,
            nextRenderedElement,
            transaction,
            this._processChildContext(context)
        );
    } else {
        var oldHostNode = ReactReconciler.getHostNode(
            prevComponentInstance
        );
        ReactReconciler.unmountComponent(prevComponentInstance, false);

        var nodeType = ReactNodeTypes.getType(nextRenderedElement);
        this._renderedNodeType = nodeType;
        var child = this._instantiateReactComponent(
            nextRenderedElement,
            nodeType !== ReactNodeTypes.EMPTY /* shouldHaveDebugID */
        );
        this._renderedComponent = child;

        var nextMarkup = ReactReconciler.mountComponent(
            child,
            transaction,
            this._hostParent,
            this._hostContainerInfo,
            this._processChildContext(context),
            debugID
        );

        this._replaceNodeWithMarkup(
            oldHostNode,
            nextMarkup,
            prevComponentInstance
        );
    }
}
```

## mountComponent

```js
/**
 * Initializes the component, renders markup, and registers event listeners.
 *
 * @param {ReactReconcileTransaction|ReactServerRenderingTransaction} transaction
 * @param {?object} hostParent
 * @param {?object} hostContainerInfo
 * @param {?object} context
 * @return {?string} Rendered markup to be inserted into the DOM.
 * @final
 * @internal
 */
mountComponent: function(
    transaction,
    hostParent,
    hostContainerInfo,
    context
) {
    this._context = context;
    this._mountOrder = nextMountID++;
    this._hostParent = hostParent;
    this._hostContainerInfo = hostContainerInfo;

    var publicProps = this._currentElement.props;
    var publicContext = this._processContext(context);

    var Component = this._currentElement.type;

    var updateQueue = transaction.getUpdateQueue();

    // Initialize the public class
    var doConstruct = shouldConstruct(Component);
    var inst = this._constructComponent(
        doConstruct,
        publicProps,
        publicContext,
        updateQueue
    );
    var renderedElement;

    // Support functional components
    if (!doConstruct && (inst == null || inst.render == null)) {
        renderedElement = inst;

        inst = new StatelessComponent(Component);
        this._compositeType = CompositeTypes.StatelessFunctional;
    } else {
        if (isPureComponent(Component)) {
            this._compositeType = CompositeTypes.PureClass;
        } else {
            this._compositeType = CompositeTypes.ImpureClass;
        }
    }

    // These should be set up in the constructor, but as a convenience for
    // simpler class abstractions, we set them up after the fact.
    inst.props = publicProps;
    inst.context = publicContext;
    inst.refs = emptyObject;
    inst.updater = updateQueue;

    this._instance = inst;

    // Store a reference from the instance back to the internal representation
    ReactInstanceMap.set(inst, this);

    var initialState = inst.state;
    if (initialState === undefined) {
        inst.state = initialState = null;
    }

    this._pendingStateQueue = null;
    this._pendingReplaceState = false;
    this._pendingForceUpdate = false;

    var markup;
    if (inst.unstable_handleError) {
        markup = this.performInitialMountWithErrorHandling(
            renderedElement,
            hostParent,
            hostContainerInfo,
            transaction,
            context
        );
    } else {
        markup = this.performInitialMount(
            renderedElement,
            hostParent,
            hostContainerInfo,
            transaction,
            context
        );
    }

    if (inst.componentDidMount) {
        transaction
            .getReactMountReady()
            .enqueue(inst.componentDidMount, inst);
    }

    return markup;
}

```

## performInitialMount

```js
performInitialMount: function(
    renderedElement,
    hostParent,
    hostContainerInfo,
    transaction,
    context
) {
    var inst = this._instance;

    var debugID = 0;

    if (inst.componentWillMount) {
        // 调用生命周期函数
        inst.componentWillMount();
        // When mounting, calls to `setState` by `componentWillMount` will set
        // `this._pendingStateQueue` without triggering a re-render.
        if (this._pendingStateQueue) {
            inst.state = this._processPendingState(
                inst.props,
                inst.context
            );
        }
    }

    // If not a stateless component, we now render
    if (renderedElement === undefined) {
        renderedElement = this._renderValidatedComponent();
    }

    var nodeType = ReactNodeTypes.getType(renderedElement);
    this._renderedNodeType = nodeType;
    var child = this._instantiateReactComponent(
        renderedElement,
        nodeType !== ReactNodeTypes.EMPTY /* shouldHaveDebugID */
    );
    this._renderedComponent = child;

    var markup = ReactReconciler.mountComponent(
        child,
        transaction,
        hostParent,
        hostContainerInfo,
        this._processChildContext(context),
        debugID
    );

    return markup;
}
```






## unmountComponent

```js
/**
 * Releases any resources allocated by `mountComponent`.
 *
 * @final
 * @internal
 */
unmountComponent: function(safely) {
    if (!this._renderedComponent) {
        return;
    }

    var inst = this._instance;

    if (inst.componentWillUnmount && !inst._calledComponentWillUnmount) {
        inst._calledComponentWillUnmount = true;

        if (safely) {
            var name = this.getName() + ".componentWillUnmount()";
            ReactErrorUtils.invokeGuardedCallback(
                name,
                inst.componentWillUnmount.bind(inst)
            );
        } else {
            inst.componentWillUnmount();
        }
    }

    if (this._renderedComponent) {
        ReactReconciler.unmountComponent(this._renderedComponent, safely);
        this._renderedNodeType = null;
        this._renderedComponent = null;
        this._instance = null;
    }

    // Reset pending fields
    // Even if this component is scheduled for another update in ReactUpdates,
    // it would still be ignored because these fields are reset.
    this._pendingStateQueue = null;
    this._pendingReplaceState = false;
    this._pendingForceUpdate = false;
    this._pendingCallbacks = null;
    this._pendingElement = null;

    // These fields do not really need to be reset since this object is no
    // longer accessible.
    this._context = null;
    this._rootNodeID = 0;
    this._topLevelWrapper = null;

    // Delete the reference from the instance to this internal representation
    // which allow the internals to be properly cleaned up even if the user
    // leaks a reference to the public instance.
    ReactInstanceMap.remove(inst);◊

    // Some existing components rely on inst.props even after they've been
    // destroyed (in event handlers).
    // TODO: inst.props = null;
    // TODO: inst.state = null;
    // TODO: inst.context = null;
}
```


## attachRef

```js
/**
 * Lazily allocates the refs object and stores `component` as `ref`.
 *
 * @param {string} ref Reference name.
 * @param {component} component Component to store as `ref`.
 * @final
 * @private
 */
attachRef: function(ref, component) {
var inst = this.getPublicInstance();

var publicComponentInstance = component.getPublicInstance();

var refs = inst.refs === emptyObject ? (inst.refs = {}) : inst.refs;
refs[ref] = publicComponentInstance;
}

```


## detachRef

```js
/**
 * Detaches a reference name.
 *
 * @param {string} ref Name to dereference.
 * @final
 * @private
 */
detachRef: function(ref) {
    var refs = this.getPublicInstance().refs;
    delete refs[ref];
}
```