* `RetryCallback`
  * == callback interface -- for an -- operation / can be retried -- via -- `RetryOperations`
  * `T doWithRetry(RetryContext context) {}`
    * execute an operation -- with -- retry semantics / operations
      * should generally be idempotent,
      * if operation is retried -> implementations -- may choose to implement -- compensation semantics
  * `String getLabel() {}`
    * logical identifier for this callback /
      * distinguish retries around business operations
