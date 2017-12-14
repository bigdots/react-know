## Transaction

Transaction就是给需要执行的方法fn用wrapper封装了 initialize 和 close 方法。且支持多次封装。再通过 Transaction 提供的 perform 方法执行。 perform执行后，先调用所有initialize 方法。最后调用所有close方法。


整个生命周期就是一个Transaction，在Transaction执行期间，componentDidUpdate方法被推入一个队列中。DOM reconciliation后，再调用队列中的所有componentDidUpdate。



```js
// src/renderers/shared/utils/Transaction.js
var TransactionImpl = {
    reinitializeTransaction: function() {},

    _isInTransaction: false,

    /**
     * @abstract
     * @return {Array<TransactionWrapper>} Array of transaction wrappers.
     */

    // 会在ReactDefaultBatchingStrategy中提供
    getTransactionWrappers: null,

    isInTransaction: function() {
        return !!this._isInTransaction;
    },

    perform: function(method, scope, a, b, c, d, e, f) {},

    initializeAll: function(startIndex) {},

    closeAll: function(startIndex) {}
};
```

### reinitializeTransaction

```js
/**
 * Sets up this instance so that it is prepared for collecting metrics. Does
 * so such that this setup method may be used on an instance that is already
 * initialized, in a way that does not consume additional memory upon reuse.
 * That can be useful if you decide to make your subclass of this mixin a
 * "PooledClass".
 */
reinitializeTransaction: function(): void {
    // 返回一个对象数组，格式为
    // [
    //  {
    //     initialize: fn,
    //     close: fn
    //   }
    // ...
    // ]
    this.transactionWrappers = this.getTransactionWrappers();
    if (this.wrapperInitData) {
        // 清空 this.wrapperInitData
        this.wrapperInitData.length = 0;
    } else {
        this.wrapperInitData = [];
    }
    this._isInTransaction = false;
};
```

## perform

```js
/* eslint-disable space-before-function-paren */

/**
 * Executes the function within a safety window. Use this for the top level
 * methods that result in large amounts of computation/mutations that would
 * need to be safety checked. The optional arguments helps prevent the need
 * to bind in many cases.
 *
 * @param {function} method Member of scope to call.
 * @param {Object} scope Scope to invoke from.
 * @param {Object?=} a Argument to pass to the method.
 * @param {Object?=} b Argument to pass to the method.
 * @param {Object?=} c Argument to pass to the method.
 * @param {Object?=} d Argument to pass to the method.
 * @param {Object?=} e Argument to pass to the method.
 * @param {Object?=} f Argument to pass to the method.
 *
 * @return {*} Return value from `method`.
 */
perform = function(method, scope, a, b, c, d, e, f) {
    /* eslint-enable space-before-function-paren */

    var errorThrown;
    var ret;
    try {
        this._isInTransaction = true;
        // Catching errors makes debugging more difficult, so we start with
        // errorThrown set to true before setting it to false after calling
        // close -- if it's still set to true in the finally block, it means
        // one of these calls threw.
        errorThrown = true;
        this.initializeAll(0);
        ret = method.call(scope, a, b, c, d, e, f);
        errorThrown = false; // 执行完，表示没有错误抛出
    } finally {
        try {
            if (errorThrown) {
                // If `method` throws, prefer to show that stack trace over any thrown
                // by invoking `closeAll`.
                try {
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

## initializeAll

```js
initializeAll = function(startIndex: number): void {
    // 获得transaction wrapper数组
    var transactionWrappers = this.transactionWrappers;
    for (var i = startIndex; i < transactionWrappers.length; i++) {
        var wrapper = transactionWrappers[i];
        try {
            // Catching errors makes debugging more difficult, so we start with the
            // OBSERVED_ERROR state before overwriting it with the real return value
            // of initialize -- if it's still set to OBSERVED_ERROR in the finally
            // block, it means wrapper.initialize threw.
            this.wrapperInitData[i] = OBSERVED_ERROR;
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

## closeAll

```js
/**
 * Invokes each of `this.transactionWrappers.close[i]` functions, passing into
 * them the respective return values of `this.transactionWrappers.init[i]`
 * (`close`rs that correspond to initializers that failed will not be
 * invoked).
 */
closeAll: function(startIndex: number): void {
    
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
