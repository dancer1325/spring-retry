* == `RetryPolicy` /
  * based on latest exception's value -- dynamically adapts to -- one of a set of injected policies
    * set of exceptions -- are configured through -- `exceptionClassifier`
  * flexible implementation

* `exceptionClassifier`
  * responsible for
    * exceptions - are translated to -- concrete retry policies

* `policyMap` 
  * | ANY time, configure it or `exceptionClassifier` 
    * == NOT BOTH | SAME time