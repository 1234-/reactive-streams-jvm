## Reactive-Streams SPI Specification

Clarification of terminology used throughout this document:

* "Publisher": an implementation of the `rx.async.spi.Publisher` interface
* "Subscriber": an implementation of the `rx.async.spi.Subscriber` interface
* "Subscription": an implementation of the `rx.async.spi.Subscription` interface
* "Processor": an implementation of the `rx.async.api.Processor` interface
* "subscriber": a Subscriber which is currently subscribed, i.e. has an active Subscription
* Foo::bar: an instance method `bar` on the class `Foo`


## Specification Rules

### Publisher::subscribe(Subscriber)

* when Publisher is in `completed` state
    * must not call `onSubscribe` on the given Subscriber
    * must trigger a call to `onComplete` on the given Subscriber

* when Publisher is in `error` state
    * must not call `onSubscribe` on the given Subscriber
    * must trigger a call to `onError` on the given Subscriber

* when Publisher is in `shut-down` state
    * must not call `onSubscribe` on the given Subscriber
    * must trigger a call to `onError` with a `java.lang.IllegalStateException` on the given
      Subscriber

* when Publisher is neither in `completed`, `error` nor `shut-down` state
    * must trigger a call to `onSubscribe` on the given Subscriber if the Subscription is to be accepted
    * must trigger a call to `onError` on the given Subscriber if the Subscription is to be rejected
    * must reject the Subscription with a `java.lang.IllegalStateException` if the same Subscriber already
      has an active Subscription [1]


### Subscription::requestMore(Int)

* when Subscription is cancelled
    * must ignore the call
* when Subscription is not cancelled
    * must register the given number of additional elements to be produced to the respective subscriber
    * must throw a `java.lang.IllegalArgumentException` if the argument is <= 0
    * is allowed to synchronously call `onNext` on this (or other) subscriber(s) if and only if the
      next element is already available
    * is allowed to synchronously call `onComplete` or `onError` on this (or other) subscriber(s)


### Subscription::cancel

* when Subscription is cancelled
    * must ignore the call
* when Subscription is not cancelled
    * the Publisher must eventually cease to call any methods on the corresponding subscriber
    * the Publisher must eventually drop any references to the corresponding subscriber [2]
    * the Publisher must shut itself down if the given Subscription is the last downstream Subscription [3]


### A Publisher

* must not call `onNext`
    * more times than the total number of elements that was previously requested with 
      Subscription::requestMore by the corresponding subscriber
    * after having issued an `onComplete` or `onError` call on a subscriber

* must produce the same elements in the same sequence for all its subscribers [4]
* must support a pending element count up to 2^63-1 (Long.MAX_VALUE) and provide for overflow protection
* must call `onComplete` on a subscriber after having produced the final stream element to it [5]
* must call `onComplete` on a subscriber at the earliest possible point in time [6]
* must start producing with the oldest element still available for a new subscriber
* must call `onError` on all its subscribers if it encounters a non-recoverable error
* must not call `onComplete` or `onError` more than once per subscriber


### Subscriber::onSubscribe(Subscription), Subscriber::onNext(T)

* must asynchronously schedule a respective event to the subscriber
* must not call any methods on the Subscription, the Publisher or any other Publishers or Subscribers


### Subscriber::onComplete, Subscriber::onError(Throwable)

* must asynchronously schedule a respective event to the Subscriber
* must not call any methods on the Subscription, the Publisher or any other Publishers or Subscribers
* must consider the Subscription cancelled after having received the event


### A Subscriber

* must not accept an `onSubscribe` event if it already has an active Subscription [7]
* must call Subscription::cancel during shutdown if it still has an active Subscription
* must ensure that all calls on a Subscription take place from the same thread or provide for respective 
  external synchronization [8]
* must be prepared to receive one or more `onNext` events after having called Subscription::cancel [9]
* must be prepared to receive an `onComplete` event with or without a preceding Subscription::requestMore call
* must be prepared to receive an `onError` event with or without a preceding Subscription::requestMore call
* must make sure that all calls on its `onXXX` methods happen-before the processing of the respective
  events [10]


### A Processor

* must obey all Publisher rules on its producing side
* must obey all Subscriber rules on its consuming side
* must cancel its upstream Subscription if its last downstream Subscription has been cancelled [11]
* must immediately pass on `onError` events received from its upstream to its downstream [12]
* must be prepared to receive incoming elements from its upstream even if a downstream subscriber has not 
  requested anything yet

### Generally

* all SPI methods should neither block nor run expensive logic on the calling thread [13]

### Footnotes

1. Object equality is to be used for establishing whether two Subscribers are the "same".

2. Cancelling a Subscription and re-subscribing the same Subscriber instance to a Publisher
   might fail as subscribing the same Subscriber twice is disallowed and cancel does not 
   enforce immediate cleanup of the old subscription. In that case it will reject the 
   Subscription with a call to `onError` with a `java.lang.IllegalStateException` in the
   same way as if the Subscriber already has an active Subscription. Re-subscribing with
   the same Subscriber instance is discouraged, but this specification does not mandate
   that it is disallowed since that would mean having to store previously canceled 
   subscriptions indefinitely

3. Explicitly adding "keep-alive" Subscribers can prevent automatic shutdown if required.

4. Producing the stream elements at (temporarily) differing rates to different subscribers is allowed.

5. The final stream element must be the same for all Subscribers.

6. In particular a Publisher should not wait for another Subscription::requestMore call before calling `onComplete`
    if the information that no more elements will follow is already available before.

7. I.e. one Subscriber cannot be subscribed to multiple Publishers at the same time.
   What exactly "not accepting" means is left to the implementation but should include behavior that makes the user
   aware of the usage error (e.g. by logging, throwing an exception or similar).

8. The Subscription implementation is not required to be thread-safe.

9. if there are still requested elements pending

10. I.e. the Subscriber must take care of properly publishing the event to its processing logic.
    ("happen-before" is used here as defined by the JLS chapter 17.4.5)

11. In addition to shutting itself down as required by the Publisher rules.

12. I.e. errors must not be treated as "in-stream" events that are allowed to be buffered.

13. I.e. they are supposed to return control to the caller quickly.
