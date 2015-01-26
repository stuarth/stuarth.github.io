---
layout: post
title:  "Configuring CSRF protection in a simple Pedestal App"
date:   2015-01-25 10:30
categories: clojure
---

OWASP has an [excellent description](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_%28CSRF%29) of CSRF attacks and why they're a concern. The tl;dr is that an attacker can utilize users' existing sessions to cause them to perform unwanted actions unless precautions are taken.

### Using `io.pedestal.http.csrf`

Pedestal's `io.pedestal.http.csrf` namespace implements the [synchronizer token pattern](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)_Prevention_Cheat_Sheet#General_Recommendation:_Synchronizer_Token_Pattern) that gives us the tools we need.

We'll start with a blank `pedestal-service` template

{% highlight sh %}
lein new pedestal-service csrf
{% endhighlight %}

And add Pedestal's CSRF namespace to our service

{% highlight clojure %}
(ns csrf.service
  (:require [io.pedestal.http :as bootstrap]
            [io.pedestal.http.route :as route]
            [io.pedestal.http.body-params :as body-params]
            [io.pedestal.http.route.definition :refer [defroutes]]
            [io.pedestal.http.csrf :as csrf] ;; add this
            [ring.util.response :as ring-resp]))
{% endhighlight %}

The trick to using `csrf/anti-forgery` with a form parameter (as of 0.3.1) is to realize that the `io.pedestal.http.body-params/body-params` interceptor, which is responsible for parsing parameters encoded in the request body, is *not* included by `io.pedestal.http/default-interceptors`. That means naively enabling CSRF protection in your service map, e.g.

{% highlight clojure %}
(def service {... 
              ::bootstrap/enable-csrf {} ;; doesn't work with form params!
              ...})
{% endhighlight %}

won't work because the anti-forgery interceptor will attempt to read from `form-params` that haven't been parsed by `body-params` yet. Resolving it is simple, we'll update our `service` definition to include
{% highlight clojure %}
(def service {...
              ::bootstrap/enable-session {} 
              ...})
{% endhighlight %}

and include our csrf middleware on the appropriate routes _after_ `io.pedestal.http.body-params/body-params` has been applied

{% highlight clojure %}
(defroutes routes
  [[["/"
     ^:interceptors [(body-params/body-params)
                     bootstrap/html-body
                     (csrf/anti-forgery)]
     ["/csrf-protected "{:post csrf-protected-handler}]]]])
{% endhighlight %}

All that's left is to include the request's anti-forgery token in our forms, something like

{% highlight clojure %}
(format "<input type=\"hidden\" name=\"__anti-forgery-token\" value=\"%s\"/>"
  (::csrf/anti-forgery-token request))
{% endhighlight %}
