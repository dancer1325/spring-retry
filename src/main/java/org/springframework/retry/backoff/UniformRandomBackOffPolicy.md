* == implementation of `BackOffPolicy` /
  * back off period is random -- via -- `Sleeper.sleep(long)`
    * Reason: 🧠 avoid resonating BETWEEN related-failures | complex system 🧠

* `setMinBackOffPeriod(long)`
  * thread-safe
  * safe to call
  
* `setMaxBackOffPeriod(long)`
  * uses
    * | execution from multiple threads
  * side-effects
    * 1! retry operation could have pauses / different intervals

