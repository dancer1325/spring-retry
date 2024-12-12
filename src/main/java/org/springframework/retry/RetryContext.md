* 👀== low-level access -- to -- ongoing retry operation 👀 

* uses
  * NOT by clients
  * alter the course of the retry
    * _Example:_ force an early termination

* TODO:

* `int getRetryCount();`
  * 👀counts the # of retry attempts 👀
    * BEFORE the FIRST attempt == 0,
