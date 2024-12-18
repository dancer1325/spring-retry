* == implementation of `BackOffPolicy` /
  * back off period increases / EACH retry attempt | given set up / UP to limit
    * Reason: 🧠 avoid 2 retries getting into lock step & both failing 🧠
  * thread-safe
  * suitable for concurrent access

* if you modify the configuration -> NOT affect any retry sets / ALREADY in progress

* `setInitialInterval(long)`
  * controls the initial delay value -- for the -- FIRST retry

* `setMultiplier(double)`
  * controls by how much the delay is increased / EACH subsequent attempt
   
* `setMaxInterval(long)`
  * controls the delay interval 