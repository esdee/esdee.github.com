---
layout: post
title: "The LinkedIn throwdown"
category: project
tags: [clojure, clojurescript]
---

{% include JB/setup %}

[LinkedIn](http://www.linkedin.com/) published a post on their engineering blog - [The client-side templating throwdown: mustache, handlebars, dust.js, and more](http://engineering.linkedin.com/frontend/client-side-templating-throwdown-mustache-handlebars-dustjs-and-more) explaining how they assessed 18 javascript templating libaries for generating LinkedIn profiles.

LinkedIn discusses the requirements they had, the criteria they used in their assessments and the pros and cons of each library they assesssed - its a great read and well worth the time.

The requirement that I found most interesting was **write-once** - they wanted the same template to be rendered on both the client and server sides. Server side html generation was required for SEO. I thought I would give the throwdown a try using clojure and clojurescript.


### The profile data
LinkedIn provided the data for one [profile ](https://gist.github.com/4d90dd839145055e9bf6#file_profile.json) and the expected [html](https://gist.github.com/4d90dd839145055e9bf6#file_profile.html).
I use [cheshire](https://github.com/dakrone/cheshire) to parse the json to clojure.

### The template file
The template file consists of a number of macros that expand into clojure vectors e.g.
{% highlight clojure %}
(defmacro organization-details
  [position]
  `(let [company# (:company ~position)]
     [:p {:class "orgstats organization-details"}
      (str (:type company#) "; " (:industry company#) " Industry")]))
{% endhighlight %}
*src/clj/linked_in/app/view_macros/profile.cljs*

expands into
{% highlight html %}
<:p class="orgstats organization-details" "Publicly Held Internet Industry">
{% endhighlight %}
when called with {% highlight clojure %} {:type "Publicly Held;" :industry "Internet"} {% endhighlight %}
There is an entry point function *template* that when called with a profile returns the template
{% highlight clojure %}
`(defmacro template
  [profile]
  `[:div.content
    (profile-header ~profile)
    (profile-summary ~profile)
    (profile-skills ~profile)
    (profile-experience ~profile)])
{% endhighlight %}

### Why macros ?
Why are is all the vector returning code in the template file macros instead of functions?
This was my plan originally - template would be the sole macro everything else would be functions.

But I found that clojurescript did not expand the functions name as I expected unless they were all either defined solely within the template macro or macros rather than functions.

### Generating html on the client side
I use [crate](https://github.com/ibdknox/crate) to generate html,
      [domina](https://github.com/levand/domina) to manipulate the dom,
      [remote](https://github.com/brentonashworth/one/blob/master/src/lib/cljs/one/browser/remote.cljs)
      from [One](https://github.com/brentonashworth/one) to get data from the server.

{%highlight clojure %}
(ns linked-in.core
  (:use [crate.core :only (html)]
        [domina :only (swap-content!)]
        [domina.xpath :only (xpath)])
  (:require [cljs.reader :as reader]
            [one.browser.remote :as remote])
  (:require-macros [linked-in.app.view-macros.profile :as profile]))

; This is where the profile will be displayed
(def content-node (xpath "//div[@class='content']"))

(defn show-profile
  [profile]
  (->> profile
    profile/template
    html
    (swap-content! content-node)))

; Retrieve the clojure map from the server and display it
(defn ^:export start
  []
  (remote/request
    100
    "profiles/client/100"
    :on-success #(->> % :body reader/read-string show-profile)
    :on-error #(->> % :status (str "*ERROR* ") js/alert)))
{%endhighlight%}

### Generating html on the server side
I use [hiccup](https://github.com/weavejester/hiccup) to generate html,
      [moustache](https://github.com/cgrand/moustache) for routing,
      and the html is returned as the body of a
      [ring](https://github.com/mmcgrana/ring) response.

The function that does the work looks like this.
{%highlight clojure %}
(defn- profile
  [_ id]
  (->
   (get-profile id)
   profile/template html
   response))
{%endhighlight%}

### Gotchas
Crate creates nodes by the following function
{%highlight clojure%}
(defn create-elem [nsp tag]
  (. js/document (createElementNS nsp tag)))
{%endhighlight%}
On of the LinkedIn requirements was to replace `\n`  with  `<br/>` in a text portion of the profile.
A simple string replace would not work here because it would be escaped by `createElementNS`. I split the text based on the `\n` character and return a vec of `["text" [:br] ...]` to deal with this.

### Conclusions
I think that the clojurescript solution addresses a lot of the issues raised in the throwdown and avoids many of the cons listed for the other solutions in the LinkedIn post. It was great being able to use clojure on the client and server. In addition, beimg able to develop the client side using the browser based repl (no page refreshes!) was fun - hopefully the subject of another post.
