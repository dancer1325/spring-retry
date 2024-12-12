* `MethodInvocationRetryListenerSupport`
  * == empty method implementation of `RetryListener` /
    * focus on AOP reflective method invocations
    * convenience retry listener type-safe
    * specific methods -- with -- `MethodInvocationRetryCallback` callback parameter 
    * requirements
      * callbacks == instance of `MethodInvocationRetryCallback`
        * ⚠️otherwise -> NO action performed ⚠️
