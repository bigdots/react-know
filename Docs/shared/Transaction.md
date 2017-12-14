## Transaction

```js
// src/renderers/shared/utils/Transaction.js
var TransactionImpl = {
    /**
     * Sets up this instance so that it is prepared for collecting metrics. Does
     * so such that this setup method may be used on an instance that is already
     * initialized, in a way that does not consume additional memory upon reuse.
     * That can be useful if you decide to make your subclass of this mixin a
     * "PooledClass".
     */
    reinitializeTransaction: function(): void {
        this.transactionWrappers = this.getTransactionWrappers();
        if (this.wrapperInitData) {
            // 清空 this.wrapperInitData
            this.wrapperInitData.length = 0;
        } else {
            this.wrapperInitData = [];
        }
        this._isInTransaction = false;
    },

    _isInTransaction: false,

    /**
     * @abstract
     * @return {Array<TransactionWrapper>} Array of transaction wrappers.
     */
    getTransactionWrappers: null,

    isInTransaction: function(): boolean {
        return !!this._isInTransaction;
    },

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
    perform: function(method, scope, a, b, c, d, e, f) {
        /* eslint-enable space-before-function-paren */
        invariant(
            !this.isInTransaction(),
            "Transaction.perform(...): Cannot initialize a transaction when there " +
                "is already an outstanding transaction."
        );
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
            errorThrown = false;
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
    },

    initializeAll: function(startIndex: number): void {
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
    },

    /**
     * Invokes each of `this.transactionWrappers.close[i]` functions, passing into
     * them the respective return values of `this.transactionWrappers.init[i]`
     * (`close`rs that correspond to initializers that failed will not be
     * invoked).
     */
    closeAll: function(startIndex: number): void {
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
    }
};
```


