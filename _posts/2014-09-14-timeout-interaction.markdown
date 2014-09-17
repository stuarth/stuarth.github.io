---
layout: post
title:  "Surprising timeout chan interactions in core.async"
date:   2014-09-14 8:56:00
categories: clojure
---

## The issue distilled

We'll create two timeout channels, close one, and read from the other. 

{% highlight clojure %}
(let [a (timeout 100)
      b (timeout 100)
      t0 (System/currentTimeMillis)]
  (close! a)  
  (<!! b)    
  (- (System/currentTimeMillis) t0))
  ;; how much time elapsed?
{% endhighlight %}

I expected this would yield ~100ms. However, evaluating the expression instead shows that almost no time has elapsed

## What's going on?

Let's dive into the code! The relevant bit's in [timers.clj](https://github.com/clojure/core.async/blob/53bf7866f195e6ba247ff7122b99784e66e9f1bb/src/main/clojure/clojure/core/async/impl/timers.clj#L43)

{% highlight clojure %}

(defn timeout
  "returns a channel that will close after msecs"
  [^long msecs]
  (let [timeout (+ (System/currentTimeMillis) msecs)
        me (.ceilingEntry timeouts-map timeout)]
    (or (when (and me (< (.getKey me) (+ timeout TIMEOUT_RESOLUTION_MS)))
          (.channel ^TimeoutQueueEntry (.getValue me)))
        (let [timeout-channel (channels/chan nil)
              timeout-entry (TimeoutQueueEntry. timeout-channel timeout)]
          (.put timeouts-map timeout timeout-entry)
          (.put timeouts-queue timeout-entry)
          timeout-channel))))

{% endhighlight %}

Bells are probably going off when you see `TIMEOUT_RESOLUTION_MS` (defined above: `(def ^:const TIMEOUT_RESOLUTION_MS 10)`). Let's unpack the surrounding code. First in our `let` block, we have `timeout`, which converts the relative `msecs` into a clock time to expire. We then set `me` to `(.ceilingEntry timeouts-map timeout)`.

`timeouts-map` is a [java.util.concurrent.ConcurrentSkipListMap](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentSkipListMap.html) and its method [`ceilingEntry`](http://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentSkipListMap.html#ceilingEntry(K)) "Returns a key-value mapping associated with the least key greater than or equal to the given key, or null if there is no such entry". So `me` will be the entry (if present) in `timeouts-map` with the first value that's greater than `timeout`.

The unexpected behavior comes from the next line

{% highlight clojure %}
(when (and me (< (.getKey me) (+ timeout TIMEOUT_RESOLUTION_MS)))
          (.channel ^TimeoutQueueEntry (.getValue me)))
{% endhighlight %}

So, if we found an `me`, and its key is within the tolerance of `TIMEOUT_RESOLUTION_MS`, we return an existing channel rather than creating a new one! This explains the behavior at the top: `a` and `b` share a channel, so closing `a` immediately closed `b`, allowing the entire block to evaluate almost instantly.

### Thanks!

A huge thanks to [Brian Rowe](https://github.com/briprowe) 


