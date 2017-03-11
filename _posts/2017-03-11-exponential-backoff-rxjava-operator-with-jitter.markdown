---
layout: post
cover: 'assets/images/exponential-backoff-cover.jpg'
title: Exponential-Backoff RxJava operator with Jitter
date: 2017-03-11 10:30:00
tags: java reactive programming
subclass: 'post'
categories: 'leandrofavarin'
navigation: True
logo: 'assets/images/ghost.png'
---

After reading [this article](https://stripe.com/blog/idempotency) about how Stripe handle errors between the clients and the server, I decided to write one in Java using RxJava 2.

{% gist leandrofavarin/2ec45769c2383c371a63aab0e91099b0 ExpBackoff.java %}

With `Jitter` being:

{% gist leandrofavarin/2ec45769c2383c371a63aab0e91099b0 Jitter.java %}

A default implementation that could cause a 15% variance:

{% gist leandrofavarin/2ec45769c2383c371a63aab0e91099b0 DefaultJitter.java %}

The implementation here is not 1:1 to what Stripe did, but could be changed easily to adapt to your needs.

Its usage is simple. Just apply it before subscribing:

{% gist leandrofavarin/2ec45769c2383c371a63aab0e91099b0 USAGE.md %}
