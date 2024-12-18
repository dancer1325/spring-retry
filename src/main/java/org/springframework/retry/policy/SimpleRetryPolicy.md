* == simple retry policy /
  * retries
    * fixed number of times | set of named exceptions + 's subclasses
* _Example:_ 
  ```
  retryTemplate = new RetryTemplate(new SimpleRetryPolicy(3));
  retryTemplate.execute(callback);
  ```
* | v1.3
  * NOT necessary to use it
    * Reason: 🧠 used by default 🧠
      * _Example1:_ 
        ```
         RetryTemplate.builder()
            .maxAttempts(3)
            .retryOn(Exception.class)
            .build();
        ``` 
      * _Example2:_ `RetryTemplate.defaultInstance()`

* `SimpleRetryPolicy(boolean traverseCauses){}`
  * `traverseCauses`
    * if `true` -> exception causes -- will be traversed UNTIL it's found a match or the root cause
  