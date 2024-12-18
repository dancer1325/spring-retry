* `RetryPolicy`
  * responsible for, about resources / needed by `RetryOperations`
    * allocate
    * manage 
  * allows
    * retry operations / is aware of their context (_Example:_ `RetryContext`)
      * types of context
        * internal (to the retry framework)
        * external 
          * Reason: 🧠 thanks to RetryPolicy API 🧠