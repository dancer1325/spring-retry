* `RetryListener`
  * allows
    * adding behaviour | a retry
  * implementations of `RetryOperations`
    * can issue callbacks -- to an -- interceptor | retry lifecycle
  * `boolean open(){}`
    * 👀called | BEFORE the first attempt in a retry 👀
    * implementers -- can set up -- state
      * state -- is needed by the -- policies | `RetryOperations`
    * 👀if it returns `false` -> the whole retry is stopped -> throw a `TerminatedRetryException` 👀
  * `void close(){}`
    * 👀called | AFTER the FINAL attempt (successful or not)  👀
    * uses
      * clean up ANY resource / it is held | BEFORE control returns to the retry caller
  * `void onSuccess(){}`
    * 👀called | AFTER a SUCCESSFUL attempt 👀
    * uses
      * if it throws a NEW exception & based on retry policy -> can cause ANOTHER retry 
  * `void onError(){}`
    * 👀called | AFTER ALL UNSUCESSFUL attempt at a retry 👀
