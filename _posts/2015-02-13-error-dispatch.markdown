---
layout: post
title:  "Error handling in Pedestal 0.4.0"
excerpt: "Error handling in Pedestal apps with io.pedestal.interceptor.error/error-dispatch"
date:   2015-02-13 9:30
categories: clojure
---

Pedestal 0.4.0 includes [`io.pedestal.interceptor.error/error-dispatch`](https://github.com/pedestal/pedestal/blob/88b940ca9c4b68de18f452f0e168400872f3c10e/service/src/io/pedestal/interceptor/error.clj#L5), which uses `core.match` to make error handling very tidy. For example

{% highlight clojure %}
(def my-error-handler
  (error-int/error-dispatch [context ex]
    ;; match errors from body-params interceptor
    {:interceptor :io.pedestal.http.body-params/body-params}
    (let [response (-> (ring.util.response/response "Invalid data given") :status 400)]
      (assoc context :response response))

     ;; match FileNotFoundException in the :enter stage
    {:exception-type :java.io.FileNotFoundException :stage :enter}
    ;; respond to this error differently

    ;; pass along the error if we don't match
    :else (assoc context :io.pedestal.impl.interceptor/error ex)
    ))

{% endhighlight %}

creates an error handler that tries to matches any `Throwable` from ,

The `error-dispatch` macro splices its forms after the binding vector (`[context ex]` above) into a [`core.match` expression](https://github.com/clojure/core.match/wiki/Basic-usage#map-patterns) matching against the `ex-data` of our exception, which is defined in [`io.pedestal.impl.interceptor`](https://github.com/pedestal/pedestal/blob/88b940ca9c4b68de18f452f0e168400872f3c10e/service/src/io/pedestal/impl/interceptor.clj#L30), and contains `:execution-id`, `:stage`, `:interceptor`, `:exception-type`, and `:exception`.

So `my-error-handler` will respond with a `400` for any `Throwable` from the interceptor named `:io.pedestal.http.body-params/body-param`, etc.

Giving a nice separation between the semantically-correct error for an interceptor to throw (e.g. a parser error for invalid JSON) and the response that's generated (a status 400 for invalid input).
