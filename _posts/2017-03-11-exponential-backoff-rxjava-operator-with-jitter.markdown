---
layout: post
title: Exponential-Backoff RxJava operator with Jitter
date: 2017-03-11 10:30:00
tags: [java, kotlin, rxjava, reactive programming]
---

Stripe recently published a technical [article](https://stripe.com/blog/idempotency) about how they handle errors between the clients and the server. The whole post is worth a read, and one of the topics mentioned was “Being a good distributed citizen”, in which mobile clients play fair when receiving failures from network operations. From their article (emphasis mine):

>It’s usually recommended that clients follow something akin to an [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff) algorithm as they see errors. The client blocks for a brief initial wait time on the first failure, but as the operation continues to fail, it waits proportionally to 2^n, where n is the number of failures that have occurred. **By backing off exponentially, we can ensure that clients aren’t hammering on a downed server and contributing to the problem**.  
>
>Exponential backoff has a long and interesting [history](http://www.cs.utexas.edu/users/lam/NRL/backoff.html) in computer networking.
>
>Furthermore, **it’s also a good idea to mix in an element of randomness**. If a problem with a server causes a large number of clients to fail at close to the same time, then even with back off, their retry schedules could be aligned closely enough that the retries will hammer the troubled server. This is known as [the thundering herd problem](https://en.wikipedia.org/wiki/Thundering_herd_problem).
>
>We can address thundering herd by **adding some amount of random “jitter” to each client’s wait time**. This will space out requests across all clients, and give the server some breathing room to recover.

Network failures are everywhere and can happen at any time, and if you have several clients failing at the same time, it is a BIG deal.

Since I couldn’t find any flexible yet powerful solution for this problem written in Kotlin I decided to write my own using [RxJava 2](https://github.com/ReactiveX/RxJava). The operator is:

{% highlight kotlin %}
/**
 * Exponential backoff that respects the equation: delay * retries ^ 2 * jitter
 */
class ExpBackoff(
  private val jitter: Jitter,
  private val delay: Long,
  private val unit: TimeUnit,
  private val retries: Int = 0
) : Function<Observable<out Throwable>, Observable<Long>> {

  @Throws(Exception::class)
  override fun apply(observable: Observable<out Throwable>): Observable<Long> {
    return observable
        .zipWith(Observable.range(1, retries), BiFunction<Throwable, Int, Int> { _, retryCount ->
          retryCount
        })
        .flatMap { attemptNumber -> Observable.timer(getNewInterval(attemptNumber), unit) }
  }

  private fun getNewInterval(retryCount: Int): Long {
    var newInterval = (delay * Math.pow(retryCount.toDouble(), 2.0) * jitter.get()).toLong()
    if (newInterval < 0) {
      newInterval = Long.MAX_VALUE
    }
    return newInterval
  }
}
{% endhighlight %}

With `Jitter` being:

{% highlight kotlin %}
interface Jitter {
  fun get(): Double

  companion object {
    val NO_OP = { 1 }
  }
}
{% endhighlight %}

A default implementation that could deviate up to 15% could be written as:

{% highlight kotlin %}
class DefaultJitter : Jitter {
  private val random = Random()

  /** Returns a random value inside [0.85, 1.15] every time it's called  */
  override fun get(): Double = 0.85 + random.nextDouble() % 0.3f
}
{% endhighlight %}

The implementation here is not 1:1 to what Stripe did, but it could be changed easily to adapt to your needs.

Its usage is then very simple. Just apply it before subscribing:

{% highlight kotlin %}
observable
  ...
  .retryWhen(ExpBackoff(DefaultJitter(), delay = 1, unit = SECONDS, retries = 3))
  .subscribe(/* */)
{% endhighlight %}
