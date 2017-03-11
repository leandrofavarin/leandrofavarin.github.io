---
layout: post
title: Exponential-Backoff with Jitter RxJava operator
tags:
- java
- reactive
- programming
---

After reading [this article](https://stripe.com/blog/idempotency) about how Stripe handle errors between the clients and the server, I decided to write one in Java using RxJava 2.

``` java
public class ExpBackoff implements Function<Observable<? extends Throwable>, Observable<Long>> {
  private final Jitter jitter;
  private final long delay;
  private final TimeUnit units;
  private final int retries;

  /**
   * Exponential backoff that respects the equation: delay * retries ^ 2 * jitter
   *
   * @param retries The max number of retries or -1 to for MAX_INT times.
   */
  public ExpBackoff(Jitter jitter, long delay, TimeUnit units, int retries) {
    this.jitter = jitter;
    this.delay = delay;
    this.units = units;
    this.retries = retries > 0 ? retries : Integer.MAX_VALUE;
  }

  @Override public Observable<Long> apply(
      @NonNull Observable<? extends Throwable> observable) throws Exception {
    return observable
        .zipWith(
            Observable.range(1, retries),
            (BiFunction<Throwable, Integer, Integer>) (throwable, retryCount) -> retryCount)
        .flatMap((attemptNumber) -> Observable.timer(getNewInterval(attemptNumber), units));
  }

  private long getNewInterval(int retryCount) {
    long newInterval = (long) (delay * Math.pow(retryCount, 2) * jitter.get());
    if (newInterval < 0) {
      newInterval = Long.MAX_VALUE;
    }
    return newInterval;
  }
}
```

With `Jitter` being:

``` java
public interface Jitter {
  double get();

  Jitter NO_OP = () -> 1;
}
```

A default implementation that could cause a 15% variance:

``` java
public class DefaultJitter implements Jitter {
  private final Random random = new Random();

  /** Returns a random value inside [0.85, 1.15] every time it's called */
  @Override public double get() {
    return 0.85 + random.nextDouble() % 0.3f;
  }
}
```

The implementation here is not 1:1 to what Stripe did, but could be changed easily to adapt to your needs.

Its usage is simple. Just apply it before subscribing:

``` java
observable
  ...
  .retryWhen(new ExpBackoff(new DefaultJitter(), delay: 1, TimeUnit.SECONDS, retries: 3))
  .subscribeWith(/* */);
```
