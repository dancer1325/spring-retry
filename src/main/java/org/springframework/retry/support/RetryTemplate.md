* == template class / 
  * simplifies the execution of operations -- via -- retry semantics
  * | executing operations & performing configuration changes, 
    * thread-safe
    * suitable for concurrent access
 
* Retryable operations are
  * encapsulated | implementations of `RetryCallback`
  * executed -- via -- one of the supplied `execute(){}`
  * retried
    * 👀by default, 👀
      * if the operation throws any `Exception` or subclass of `Exception` 
      * 3 maximum attempts & NO backoff
    * if you want to customize 
      * ways
        * `setRetryPolicy(RetryPolicy)`
        * `setBackOffPolicy(BackOffPolicy)`
      * 👀& done on the fly -> in progress retryable operations will NOT be affected 👀

* create a NEW instance
  ```
  RetryTemplate.builder()
    .maxAttempts(10)
    .fixedBackoff(1000)
    .build();
  ```
