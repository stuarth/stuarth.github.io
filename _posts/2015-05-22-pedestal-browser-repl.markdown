---
layout: post
title:  "Understanding Pedestal Interceptors and Selectively Injecting a Browser REPL"
excerpt: ""
date:   2015-05-22 10:36:00
categories: clojure
---

Pedestal represents interceptors, its analog to Ring middleware, as data. Let's look at how we can use that to conditionally inject a ClojureScript REPL into responses.

### Pedestal Terminology

Two working definitions as we start

* [Interceptor](https://github.com/pedestal/pedestal/blob/master/guides/documentation/service-interceptors.md) - a record containing keys corresponding functions for execution stages. Serves an analogous role to middleware in Ring.
* [context](https://github.com/pedestal/pedestal/blob/master/guides/documentation/service-context-reference.md#reference) - a map with all data relating to processing a request. Contains Ring request and response maps along with additional data.

### Marking Routes with Metadata

We'll start by adding metadata indicating that a browser REPL should be included in the response. We can either add it directly to an endpoint

{% highlight clojure %}
(require '[io.pedestal.interceptor.helpers :as h]
(require '[ring.util.response :refer [response content-type]])

(def hello-world
  (with-meta
    (h/handler
     (fn [request]
       (-> (response "<html><body><h1>hello world!!</h1></body></html>")
           (content-type "text/html"))))
    {:browser-repl true}))
{% endhighlight %}

or to an interceptor anywhere else in the chain

{% highlight clojure %}

(def include-browser-repl
  (with-meta
    (h/before
     (fn [context] context))
    {:browser-repl true}))

;; in the route definition
["/foo" ^:interceptors [include-browser-repl]
  {:get foo-handler}]
{% endhighlight %}

## Interceptor Application

Interceptors serve a similar role as middlewares in Ring but the implementation is substantially different. See Pedestal's docs for [a full discussion](https://github.com/pedestal/pedestal/blob/master/guides/documentation/service-interceptors.md#ring-request-processing), but for now the important difference is in how middlewares and interceptors are combined. In Ring, middlewares compose functionally

{% highlight clojure %}
(defn middleware-ex
  [handler transform-request transform-response]
  (fn [request]
    (let [response (handler (transform-request request))]
      (transform-response response))))

(def new-handler (-> some-handler
                     (middleware-ex fn-1 fn-2)
                     (middleware-ex fn-3 fn-4)))

;; new-handler is now a fn [request] that could be
;; composed with additional middleware
{% endhighlight %}

Pedestal represents interceptors sequentially, adding them as a queue to the request / response `context` (under `:io.pedestal.impl.interceptor/queue`). In broad strokes, requests are processed by taking an interceptor from the queue and applying its `:enter` function to the `context`, producing a new `context` that the next interceptor in the queue will act on. As each interceptor is visited, it's added to a stack in the `context` that will be traversed on the `:leave` stage. [The code](https://github.com/pedestal/pedestal/blob/73b5854d4ba01a6cd2146ca8a37ae6a2b76e8995/service/src/io/pedestal/impl/interceptor.clj#L121-L146) is well worth a read.

Knowing that the other interceptors involved in processing a request are visible in the context's `:io.pedestal.impl.interceptor/queue`, we can write an interceptor that scans them looking for the `:browser-repl` metadata

{% highlight clojure %}
(require '[io.pedestal.interceptor.helpers :refer [around]])

(defn browser-repl
  "produces an interceptor that scans the context's queue on enter for
  brower-repl metadata, calls inject-repl on leave if found"
  [inject-repl]
  (around
   ::browser-repl
   (fn [context] ;; fn to call on :enter
     (cond-> context
       (some->> context ;; scan the queue for :browser-repl metadata
                :io.pedestal.impl.interceptor/queue
                (map meta)
                (some :browser-repl))
       (assoc :include-browser-repl true))) ;; add a flag to inject the repl
   (fn [{response :response :as context}] ;; on :leave
     (cond-> context
       (:include-browser-repl context) ;; did we find browser-repl metadata?
       (update :response inject-repl))))) ;; add a repl!
{% endhighlight %}

We use `around` to implement `browser-repl` so that we can examine the interceptor queue in the `before` stage and set a flag in the `context` called `:include-browser-repl` that we'll read when a response is present in the `leave` phase and decide whether to call `inject-repl`.

`inject-repl` is a function that handles the mechanics of modifying the response, along the lines of

{% highlight clojure %}
(defn inject-repl
  [{body :body :as response}]
  (let [html (if (string? body)
               body
               (slurp body))
        repl-template (enlive/template
                       (enlive/html-snippet html) []
                       [:body]
                       (enlive/append
                        (enlive/html [:script "goog.require(\"pedestal_browser_repl.dev\");"])))]

    (->> (repl-template)
         (apply str)
         (response))))
{% endhighlight %}

We use `around` in the `browser-repl` fn above so that we can

_Note_ [Brian Rowe and Alex Redington suggested](https://groups.google.com/d/msg/pedestal-users/Peoc1LheR8I/xFXalhQFJRcJ) a couple of very nice extensions to this idea that I'd recommend studying before implementing it.

Kinda cool, right? Interceptors are data, and neat approaches fall out. We use a similar pattern for authorization.

Speaking of we -- RentPath is hiring! Tweet me if remote Clojure work sounds fun!

Also, big thanks to __Brian Rowe__ for a number of improvements to this post!
