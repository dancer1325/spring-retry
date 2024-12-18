* == strategy interface /
  * allows
    * 👀controlling the pause between retryWithFailure -- nextRetryAttempt 👀
      * == | 1! RetryTemplate retry operation

* implementations of it
  * expectations
    * be thread-safe
    * designed for concurrent access

* configuration / EACH implementation
  * expectations
    * be thread-safe
    * NOT need to -- be suitable for -- HIGH load concurrent access

* `start(){}`
  * called / EACH block of retry operations
  * implementations -- can return an -- implementation-specific `BackOffContext` 
  * uses
    * track state -- through -- subsequent back off invocations

* `backOff(){}`
  * handle EACH back off process
