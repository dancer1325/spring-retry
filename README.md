[![Build Status](https://github.com/spring-projects/spring-retry/actions/workflows/build-and-deploy-snapshot.yml/badge.svg?branch=main)](https://github.com/spring-projects/spring-retry/actions/workflows/build-and-deploy-snapshot.yml?query=branch%3Amain) [![Javadocs](https://www.javadoc.io/badge/org.springframework.retry/spring-retry.svg)](https://www.javadoc.io/doc/org.springframework.retry/spring-retry)

* goal
  * declarative retry support -- for -- Spring applications
* uses
  * Spring Batch,
  * Spring Integration,
  * other Spring projects
  * explicit usage

* How to add?
  * via Maven

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
```

## Quick Start

### Declarative
* `@Retryable` & `@Recover`
  * 👁️ has -- an additional runtime dependency on -- AOP classes 👁️
    * Check ['Java Configuration for Retry Proxies'](#javaConfigForRetryProxies)
* _Example:_

```java
@Configuration
@EnableRetry
public class Application {

}

@Service
class Service {
    
    // if invocation to this method, hits "RemoteAccessException" -> retries, by default, 3 times 
    @Retryable(retryFor = RemoteAccessException.class)
    public void service() {
        // ... do something
    }
    
    // if after 3 times, same error -> this method is invoked
    @Recover
    public void recover(RemoteAccessException e) {
       // ... panic
    }
}
```

### Imperative 

* -- based on the -- "spring-retry" version 
  * spring-retry v1.3+
    * _Example:_ 

        ```java
        RetryTemplate template = RetryTemplate.builder()
                        .maxAttempts(3)
                        .fixedBackoff(1000)
                        .retryOn(RemoteAccessException.class)
                        .build();
        
        template.execute(ctx -> {
            // ... do something
        });
        ```

  * spring-retry < v1.3
    * Check [RetryTemplate](#using-retrytemplate)

## Building

* requirements
  * Java v17+
  * Maven v3.0.5+
* `mvn install`
  * to build

## Features and API

* goal
  * features of Spring Retry
  * how to use its API

### Using `RetryTemplate`
* TODO:
To make processing more robust and less prone to failure, it sometimes helps to
automatically retry a failed operation, in case it might succeed on a subsequent attempt.
Errors that are susceptible to this kind of treatment are transient in nature. For
example, a remote call to a web service or an RMI service that fails because of a network
glitch or a `DeadLockLoserException` in a database update may resolve itself after a
short wait. To automate the retry of such operations, Spring Retry has the
`RetryOperations` strategy. The `RetryOperations` interface definition follows:

```java
public interface RetryOperations {

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback) throws E;

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RecoveryCallback<T> recoveryCallback) throws E;

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RetryState retryState) throws E, ExhaustedRetryException;

	<T, E extends Throwable> T execute(RetryCallback<T, E> retryCallback, RecoveryCallback<T> recoveryCallback, RetryState retryState) throws E;

}
```

The basic callback is a simple interface that lets you insert some business logic to be retried:

```java
public interface RetryCallback<T, E extends Throwable> {

    T doWithRetry(RetryContext context) throws E;

}
```

The callback is tried, and, if it fails (by throwing an `Exception`), it is retried
until either it is successful or the implementation decides to abort. There are a number
of overloaded `execute` methods in the `RetryOperations` interface, to deal with various
use cases for recovery when all retry attempts are exhausted and to deal with retry state, which
lets clients and implementations store information between calls (more on this later).

The simplest general purpose implementation of `RetryOperations` is `RetryTemplate`.
The following example shows how to use it:

```java
RetryTemplate template = new RetryTemplate();

TimeoutRetryPolicy policy = new TimeoutRetryPolicy();
policy.setTimeout(30000L);

template.setRetryPolicy(policy);

MyObject result = template.execute(new RetryCallback<MyObject, Exception>() {

    public MyObject doWithRetry(RetryContext context) {
        // Do stuff that might fail, e.g. webservice operation
        return result;
    }

});
```

In the preceding example, we execute a web service call and return the result to the user.
If that call fails, it is retried until a timeout is reached.

Since version 1.3, fluent configuration of `RetryTemplate` is also available, as follows:

```java
RetryTemplate.builder()
      .maxAttempts(10)
      .exponentialBackoff(100, 2, 10000)
      .retryOn(IOException.class)
      .traversingCauses()
      .build();

RetryTemplate.builder()
      .fixedBackoff(10)
      .withinMillis(3000)
      .build();

RetryTemplate.builder()
      .infiniteRetry()
      .retryOn(IOException.class)
      .uniformRandomBackoff(1000, 3000)
      .build();
```

### Using `RetryContext`

* uses
  * `RetryCallback`'s method parameter
    * MANY callbacks -- ignore the -- context
    * 👀-> store data | duration of the iteration 👀

* parent context of the `RetryContext`
  * requirements
    * there is a nested retry in progress | SAME thread 
  * uses
    * store data / shared between calls -- to -- execute

* if you do NOT have access to the context directly -- via calling `RetrySynchronizationManager.getContext()` -- you can obtain the scope of the retries' current context 

* stored, by default, | `ThreadLocal`
  * if you are using virtual threads (== Java v21+) ->
    * NOT use `ThreadLocal` -- see [JEP 444](https://openjdk.org/jeps/444)
    * store | `Map`
    * call `RetrySynchronizationManager.setUseThreadLocal(false)`

### Using `RecoveryCallback`

When a retry is exhausted, the `RetryOperations` can pass control to a different
callback: `RecoveryCallback`. To use this feature, clients can pass in the callbacks
together to the same method, as the following example shows:

```
MyObject myObject = template.execute(new RetryCallback<MyObject, Exception>() {
    public MyObject doWithRetry(RetryContext context) {
        // business logic here
    },
  new RecoveryCallback<MyObject>() {
    MyObject recover(RetryContext context) throws Exception {
          // recover logic here
    }
});
```

If the business logic does not succeed before the template decides to abort, the client is
given the chance to do some alternate processing through the recovery callback.

## Stateless Retry

In the simplest case, a retry is just a while loop: the `RetryTemplate` can keep trying
until it either succeeds or fails. The `RetryContext` contains some state to determine
whether to retry or abort. However, this state is on the stack, and there is no need to
store it anywhere globally. Consequently, we call this "stateless retry". The distinction
between stateless and stateful retry is contained in the implementation of `RetryPolicy`
(`RetryTemplate` can handle both). In a stateless retry, the callback is always executed
in the same thread as when it failed on retry.

## Stateful Retry

Where the failure has caused a transactional resource to become invalid, there are some
special considerations. This does not apply to a simple remote call, because there is (usually) no
transactional resource, but it does sometimes apply to a database update,
especially when using Hibernate. In this case, it only makes sense to rethrow the
exception that called the failure immediately so that the transaction can roll back and
we can start a new (and valid) one.

In these cases, a stateless retry is not good enough, because the re-throw and roll back
necessarily involve leaving the `RetryOperations.execute()` method and potentially losing
the context that was on the stack. To avoid losing the context, we have to introduce a
storage strategy to lift it off the stack and put it (at a minimum) in heap storage. For
this purpose, Spring Retry provides a storage strategy called `RetryContextCache`, which
you can inject into the `RetryTemplate`. The default implementation of the
`RetryContextCache` is in-memory, using a simple `Map`. It has a strictly enforced maximum
capacity, to avoid memory leaks, but it does not have any advanced cache features (such as
time to live). You should consider injecting a `Map` that has those features if you need
them. For advanced usage with multiple processes in a clustered environment, you might
also consider implementing the `RetryContextCache` with a cluster cache of some sort
(though, even in a clustered environment, this might be overkill).

Part of the responsibility of the `RetryOperations` is to recognize the failed operations
when they come back in a new execution (and usually wrapped in a new transaction). To
facilitate this, Spring Retry provides the `RetryState` abstraction. This works in
conjunction with special `execute` methods in the `RetryOperations`.

The failed operations are recognized by identifying the state across multiple invocations
of the retry. To identify the state, you can provide a `RetryState` object that is
responsible for returning a unique key that identifies the item. The identifier is used as
a key in the `RetryContextCache`.

> *Warning:*
Be very careful with the implementation of `Object.equals()` and `Object.hashCode()` in
the key returned by `RetryState`. The best advice is to use a business key to identify the
items. In the case of a JMS message, you can use the message ID.

When the retry is exhausted, you also have the option to handle the failed item in a
different way, instead of calling the `RetryCallback` (which is now presumed to be likely
to fail). As in the stateless case, this option is provided by the `RecoveryCallback`,
which you can provide by passing it in to the `execute` method of `RetryOperations`.

The decision to retry or not is actually delegated to a regular `RetryPolicy`, so the
usual concerns about limits and timeouts can be injected there (see the [Additional Dependencies](#Additional_Dependencies) section).

## Retry Policies

* `RetryPolicy`
  * uses
    * | `RetryTemplate`, determine to retry or fail   
  * built-in implementations
    * `SimpleRetryPolicy`
      * _Example:_
        ```java
        // Set the max attempts including the initial attempt before retrying
        // and retry on all exceptions (this is the default):
        SimpleRetryPolicy policy = new SimpleRetryPolicy(5, Collections.singletonMap(Exception.class, true));
        
        // Use the policy...
        RetryTemplate template = new RetryTemplate();
        template.setRetryPolicy(policy);
        template.execute(new RetryCallback<MyObject, Exception>() {
            public MyObject doWithRetry(RetryContext context) {
                // business logic here
            }
        });
        ```
    * `TimeoutRetryPolicy`
    * `ExceptionClassifierRetryPolicy`

* `RetryTemplate`
  * `.execute(currentPolicy)`
    * 👀creates a `RetryContext` 👀
    * 👀pass the `RetryContext` | `RetryCallback` / EVERY attempt 👀 
    * if a callback fails ->
      * `RetryTemplate` -- has to make a call to the -- `RetryPolicy`
        * Reason: 🧠 ask it to 🧠
          * update its state | `RetryContext` &
          * validate if ANOTHER attempt can be made
            * if another attempt NOT allowed 
              * -> the policy identifies the exhausted state (!= handling the exception)
              * `RetryTemplate`
                * | ALL cases, throws the original exception
                * | stateful case, throws `RetryExhaustedException`

* if a failure is deterministic (== always fail | ANY # of retries) -> NOT retry it

## Backoff Policies

When retrying after a transient failure, it often helps to wait a bit before trying again,
because (usually) the failure is caused by some problem that can be resolved only by
waiting. If a `RetryCallback` fails, the `RetryTemplate` can pause execution according to
the `BackoffPolicy`. The following listing shows the definition of the `BackoffPolicy`
interface:

```java
public interface BackoffPolicy {

    BackOffContext start(RetryContext context);

    void backOff(BackOffContext backOffContext)
        throws BackOffInterruptedException;

}
```

A `BackoffPolicy` is free to implement the backoff in any way it chooses. The policies
provided by Spring Retry all use `Object.wait()`. A common use case is to
back off with an exponentially increasing wait period, to avoid two retries getting into
lock step and both failing (a lesson learned from Ethernet). For this purpose, Spring
Retry provides `ExponentialBackoffPolicy`. Spring Retry also provides randomized versions
of delay policies that are quite useful to avoid resonating between related failures in a
complex system, by adding jitter.

## Listeners

* `RetryListener`
  * goal
    * receive additional callbacks -- for -- cross-cutting concerns ACROSS a number of DIFFERENT retries 
  * `open` & `close` callbacks
    * BEFORE & AFTER the entire retry
    * `close(..., Throwable)`
  * `onSuccess`, `onError`
    * apply | individual `RetryCallback` calls
    * `onSuccess`
      * | v2.0+, called AFTER a successful -- call to the -- callback
        * -> listener can examine the result & throw an exception -- based on -- your business logic
          * based on type of exception thrown -> call will be or NOT retried
  * if there are >=1 listener -> there is an order
    * `open` called | SAME order
    * `onSuccess`, `onError`, and `close` called | reverse order 

* `RetryContext`
  * allows
    * obtaining the current retry count 
 
* `RetryTemplate`
  * allows
    * registering `RetryListener` instances, and they are given callbacks with the `RetryContext` and `Throwable` (where available during the iteration).

### Listeners for Reflective Method Invocations

* if methods are annotated with `@Retryable` or Spring AOP intercepted methods -> Spring Retry allows a detailed inspection of the method invocation | `RetryListener` implementation
  * use cases
    * monitor how often a certain method call has been retried & exposed / detailed tagging information (_Example:_ class name, method name, or parameter values)
      * _Example:_ 
        ```java
        
        template.registerListener(new MethodInvocationRetryListenerSupport() {
              @Override
              protected <T, E extends Throwable> void doClose(RetryContext context,
                  MethodInvocationRetryCallback<T, E> callback, Throwable throwable) {
                monitoringTags.put(labelTagName, callback.getLabel());
                Method method = callback.getInvocation()
                    .getMethod();
                monitoringTags.put(classTagName,
                    method.getDeclaringClass().getSimpleName());
                monitoringTags.put(methodTagName, method.getName());
        
                // register a monitoring counter with appropriate tags
                // ...
        
                @Override
                protected <T, E extends Throwable> void doOnSuccess(RetryContext context,
                        MethodInvocationRetryCallback<T, E> callback, T result) {
        
                    Object[] arguments = callback.getInvocation().getArguments();
        
                    // decide whether the result for the given arguments should be accepted
                    // or retried according to the retry policy
                }
        
              }
            });
        ```

* `MethodInvocationRetryListenerSupport`
  * | v2.0, NEW method `doOnSuccess`

## Declarative Retry

Sometimes, you want to retry some business processing every time it happens. The classic
example of this is the remote service call. Spring Retry provides an AOP interceptor that
wraps a method call in a `RetryOperations` instance for exactly this purpose. The
`RetryOperationsInterceptor` executes the intercepted method and retries on failure
according to the `RetryPolicy` in the provided `RepeatTemplate`.


### <a name="javaConfigForRetryProxies"></a> Java Configuration -- for -- Retry Proxies

* steps
  * `@EnableRetry` | one of your `@Configuration` classes
  * use `@Retryable` | methods (or | type level for ALL methods) / you want to retry

* _Examples:_
  * _Example1:_ specify SEVERAL retry listeners
      ```java
      @Configuration
      @EnableRetry
      public class Application {
    
          @Bean
          public Service service() {
              return new Service();
          }
    
          @Bean public RetryListener retryListener1() {
              return new RetryListener() {...}
          }
    
          @Bean public RetryListener retryListener2() {
              return new RetryListener() {...}
          }
    
      }
    
      @Service
      class Service {
          @Retryable(RemoteAccessException.class)
          public service() {
              // ... do something
          }
      }
      ```
  * _Example2:_ `@Retryable` attributes -- to control the -- `RetryPolicy` and `BackoffPolicy`
    ```java
    @Service
    class Service {
        // random backoff / [100ms, 500ms] & < 12 attempts 
        @Retryable(maxAttempts=12, backoff=@Backoff(delay=100, maxDelay=500))
        public service() {
            // ... do something
        }
    }
    ```

* `@Retryable`
  * .`stateful`
    * default: `false`
    * control whether the retry is stateful or NOT

* TODO: To use stateful retry, the intercepted method has to have
arguments, since they are used to construct the cache key for the state.

The `@EnableRetry` annotation also looks for beans of type `Sleeper` and other strategies
used in the `RetryTemplate` and interceptors to control the behavior of the retry at runtime.

The `@EnableRetry` annotation creates proxies for `@Retryable` beans, and the proxies
(that is, the bean instances in the application) have the `Retryable` interface added to
them. This is purely a marker interface, but it might be useful for other tools looking to
apply retry advice (they should usually not bother if the bean already implements
`Retryable`).

If you want to take an alternative code path when
the retry is exhausted, you can supply a recovery method. Methods should be declared in the same class as the `@Retryable`
instance and marked `@Recover`. The return type must match the `@Retryable` method. The arguments
for the recovery method can optionally include the exception that was thrown and
(optionally) the arguments passed to the original retryable method (or a partial list of
them as long as none are omitted up to the last one needed). The following example shows how to do so:

```java
@Service
class Service {
    @Retryable(retryFor = RemoteAccessException.class)
    public void service(String str1, String str2) {
        // ... do something
    }
    @Recover
    public void recover(RemoteAccessException e, String str1, String str2) {
       // ... error handling making use of original args if required
    }
}
```

To resolve conflicts between multiple methods that can be picked for recovery, you can explicitly specify recovery method name.
The following example shows how to do so:

```java
@Service
class Service {
    @Retryable(recover = "service1Recover", retryFor = RemoteAccessException.class)
    public void service1(String str1, String str2) {
        // ... do something
    }

    @Retryable(recover = "service2Recover", retryFor = RemoteAccessException.class)
    public void service2(String str1, String str2) {
        // ... do something
    }

    @Recover
    public void service1Recover(RemoteAccessException e, String str1, String str2) {
        // ... error handling making use of original args if required
    }

    @Recover
    public void service2Recover(RemoteAccessException e, String str1, String str2) {
        // ... error handling making use of original args if required
    }
}
```

Version 1.3.2 and later supports matching a parameterized (generic) return type to detect the correct recovery method:

```java
@Service
class Service {

    @Retryable(retryFor = RemoteAccessException.class)
    public List<Thing1> service1(String str1, String str2) {
        // ... do something
    }

    @Retryable(retryFor = RemoteAccessException.class)
    public List<Thing2> service2(String str1, String str2) {
        // ... do something
    }

    @Recover
    public List<Thing1> recover1(RemoteAccessException e, String str1, String str2) {
       // ... error handling for service1
    }

    @Recover
    public List<Thing2> recover2(RemoteAccessException e, String str1, String str2) {
       // ... error handling for service2
    }

}
```

Version 1.2 introduced the ability to use expressions for certain properties. The
following example show how to use expressions this way:

```java

@Retryable(exceptionExpression="message.contains('this can be retried')")
public void service1() {
  ...
}

@Retryable(exceptionExpression="message.contains('this can be retried')")
public void service2() {
  ...
}

@Retryable(exceptionExpression="@exceptionChecker.shouldRetry(#root)",
    maxAttemptsExpression = "#{@integerFiveBean}",
  backoff = @Backoff(delayExpression = "#{1}", maxDelayExpression = "#{5}", multiplierExpression = "#{1.1}"))
public void service3() {
  ...
}
```

Since Spring Retry 1.2.5, for `exceptionExpression`, templated expressions (`#{...}`) are
deprecated in favor of simple expression strings
(`message.contains('this can be retried')`).

Expressions can contain property placeholders, such as `#{${max.delay}}` or
`#{@exceptionChecker.${retry.method}(#root)}`. The following rules apply:

- `exceptionExpression` is evaluated against the thrown exception as the `#root` object.
- `maxAttemptsExpression` and the `@BackOff` expression attributes are evaluated once,
during initialization. There is no root object for the evaluation but they can reference
other beans in the context

Starting with version 2.0, expressions in `@Retryable`, `@CircuitBreaker`, and `BackOff` can be evaluated once, during application initialization, or at runtime.
With earlier versions, evaluation was always performed during initialization (except for `Retryable.exceptionExpression` which is always evaluated at runtime).
When evaluating at runtime, a root object containing the method arguments is passed to the evaluation context.

**Note:** The arguments are not available until the method has been called at least once; they will be null initially, which means, for example, you can't set the initial `maxAttempts` using an argument value, you can, however, change the `maxAttempts` after the first failure and before any retries are performed.
Also, the arguments are only available when using stateless retry (which includes the `@CircuitBreaker`).

Version 2.0 adds more flexibility to exception classification.

```java
@Retryable(retryFor = RuntimeException.class, noRetryFor = IllegalStateException.class, notRecoverable = {
        IllegalArgumentException.class, IllegalStateException.class })
public void service() {
    ...
}

@Recover
public void recover(Throwable cause) {
    ...
}
```

`retryFor` and `noRetryFor` are replacements of `include` and `exclude` properties, which are now deprecated.
The new `notRecoverable` property allows the recovery method(s) to be skipped, even if one matches the exception type; the exception is thrown to the caller either after retries are exhausted, or immediately, if the exception is not retryable.

##### Examples

```java
@Retryable(maxAttemptsExpression = "@runtimeConfigs.maxAttempts",
        backoff = @Backoff(delayExpression = "@runtimeConfigs.initial",
                maxDelayExpression = "@runtimeConfigs.max", multiplierExpression = "@runtimeConfigs.mult"))
public void service() {
    ...
}
```

Where `runtimeConfigs` is a bean with those properties.

```java
@Retryable(maxAttemptsExpression = "args[0] == 'something' ? 3 : 1")
public void conditional(String string) {
    ...
}
```

#### <a name="Additional_Dependencies"></a> Additional Dependencies

The declarative approach to applying retry handling by using the `@Retryable` annotation
shown earlier has an additional runtime dependency on AOP classes that need to be declared
in your project. If your application is implemented by using Spring Boot, this dependency
is best resolved by using the Spring Boot starter for AOP. For example, for Gradle, add
the following line to your `build.gradle` file:

```
    runtimeOnly 'org.springframework.boot:spring-boot-starter-aop'
```

For non-Boot apps, you need to declare a runtime dependency on the latest version of
AspectJ's `aspectjweaver` module. For example, for Gradle, you should add the following
line  to your `build.gradle` file:

```
    runtimeOnly 'org.aspectj:aspectjweaver:1.9.20.1'
```
### Further customizations

Starting from version 1.3.2 and later `@Retryable` annotation can be used in custom composed annotations to create your own annotations with predefined behaviour.
For example if you discover you need two kinds of retry strategy, one for local services calls, and one for remote services calls, you could decide
to create two custom annotations `@LocalRetryable` and `@RemoteRetryable` that differs in the retry strategy as well in the maximum number of retries.

To make custom annotation composition work properly you can use `@AliasFor` annotation, for example on the `recover` method, so that you can further extend the versatility of your custom annotations and allow the `recover` argument value
to be picked up as if it was set on the `recover` method of the base `@Retryable` annotation.

Usage Example:
```java
@Service
class Service {
    ...
    
    @LocalRetryable(include = TemporaryLocalException.class, recover = "service1Recovery")
    public List<Thing> service1(String str1, String str2){
        //... do something
    }
    
    public List<Thing> service1Recovery(TemporaryLocalException ex,String str1, String str2){
        //... Error handling for service1
    }
    ...
    
    @RemoteRetryable(include = TemporaryRemoteException.class, recover = "service2Recovery")
    public List<Thing> service2(String str1, String str2){
        //... do something
    }

    public List<Thing> service2Recovery(TemporaryRemoteException ex, String str1, String str2){
        //... Error handling for service2
    }
    ...
}
```

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Retryable(maxAttempts = "3", backoff = @Backoff(delay = "500", maxDelay = "2000", random = true)
)
public @interface LocalRetryable {
    
    @AliasFor(annotation = Retryable.class, attribute = "recover")
    String recover() default "";

    @AliasFor(annotation = Retryable.class, attribute = "value")
    Class<? extends Throwable>[] value() default {};

    @AliasFor(annotation = Retryable.class, attribute = "include")

    Class<? extends Throwable>[] include() default {};

    @AliasFor(annotation = Retryable.class, attribute = "exclude")
    Class<? extends Throwable>[] exclude() default {};

    @AliasFor(annotation = Retryable.class, attribute = "label")
    String label() default "";

}
```

```java
@Target({ ElementType.METHOD, ElementType.TYPE })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Retryable(maxAttempts = "5", backoff = @Backoff(delay = "1000", maxDelay = "30000", multiplier = "1.2", random = true)
)
public @interface RemoteRetryable {
    
    @AliasFor(annotation = Retryable.class, attribute = "recover")
    String recover() default "";

    @AliasFor(annotation = Retryable.class, attribute = "value")
    Class<? extends Throwable>[] value() default {};

    @AliasFor(annotation = Retryable.class, attribute = "include")
    Class<? extends Throwable>[] include() default {};

    @AliasFor(annotation = Retryable.class, attribute = "exclude")
    Class<? extends Throwable>[] exclude() default {};

    @AliasFor(annotation = Retryable.class, attribute = "label")
    String label() default "";

}
```

### XML Configuration

The following example of declarative iteration uses Spring AOP to repeat a service call to
a method called `remoteCall`:

```xml
<aop:config>
    <aop:pointcut id="transactional"
        expression="execution(* com..*Service.remoteCall(..))" />
    <aop:advisor pointcut-ref="transactional"
        advice-ref="retryAdvice" order="-1"/>
</aop:config>

<bean id="retryAdvice"
    class="org.springframework.retry.interceptor.RetryOperationsInterceptor"/>
```

For more detail on how to configure AOP interceptors, see the [Spring Framework Documentation](https://docs.spring.io/spring/docs/current/spring-framework-reference/index.html).

The preceding example uses a default `RetryTemplate` inside the interceptor. To change the
policies or listeners, you need only inject an instance of `RetryTemplate` into the
interceptor.

## Micrometer Support

* `MetricsRetryListener`
  * "spring-retry" v.2.0.8+
  * ways to be injected
    * | `RetryTemplate`
    * `@Retryable(listeners)` == `listeners` argument
  * -- based on -- [Micrometer](https://docs.micrometer.io/micrometer/reference/index.html) `MeterRegistry`
  * exposes a `spring.retry` timer / from `open()` -- till -- `close()` listener callbacks
    * -> covers the WHOLE retry operation
    * adds tags
      * `name` / -- based on -- `RetryCallback.getLabel()` value
      * `retry.count` 
        * if NO any retries entered (== first call is successful)  -> `0`
      * `exception`
        * if ALL the retry attempts have been exhausted -> last exception -- is thrown back to the -- caller
  * ways to customize it
    * static tags
    * `Function<RetryContext, Iterable<Tag>>`

## Getting Support
* see
  * [Spring Retry tags on Stack Overflow](https://stackoverflow.com/questions/tagged/spring-retry)
  * [Commercial support](https://spring.io/support)
