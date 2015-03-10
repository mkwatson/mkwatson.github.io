---
layout: post
title: I Almost Used Eval&#58; Interpreted vs. Compiled Clojure
---

*originally published [here](http://markkettner.roughdraft.io/866b853b065896d9b138-i-almost-used-eval-interpreted-vs-compiled-clojure)*

## Bad Clojure

In the process of writing a [RESTful](http://en.wikipedia.org/wiki/Representational_state_transfer) [web service](http://en.wikipedia.org/wiki/Web_services) using [Korma](http://sqlkorma.com/), I found myself wanting to write a function for threading data through a run-time defined number of other functions (and then [```eval-ing```](http://clojuredocs.org/clojure_core/1.2.0/clojure.core/eval) this generated expression). I knew I was writing terrible code, and ended up with something like this:

{% highlight clojure %}
;; Piece of data
user> (def data 0.123456789)
#'user/data

;; Array of functions
user> (def fns [#(+ 1 %) #(* 13.1 %) int range set])
#'user/fns

;; Function for threading data through each function in a list
user> (defn bad-fn [l]
        `(-> data
             ~@(map (fn [f] `(~f)) l)))
#'user/bad-fn

;; Generate S-expression at run-time
user> (bad-fn fns)
(clojure.core/->
 user/data
 (#<user$fn__2307 user$fn__2307@4b19c6e8>)
 (#<user$fn__2309 user$fn__2309@5a25baf2>)
 (#<core$int clojure.core$int@114379f5>)
 (#<core$range clojure.core$range@4f3aa5a6>)
 (#<core$set clojure.core$set@42ed7c83>))
 
;; And eval the sucker
user> (eval *1)
#{0 1 2 3 4 5 6 7 8 9 10 11 12 13}
{% endhighlight %}

## Better Clojure

I quickly realized a safer and more idiomatic way to accomplish what I wanted using [``` reduce```](http://clojuredocs.org/clojure_core/1.2.0/clojure.core/reduce):

{% highlight clojure %}
(reduce #(%2 %1) data fns)
{% endhighlight %}

## Interpreted vs. Compiled

After this, I started thinking about [interpreted]([http://en.wikipedia.org/wiki/Interpreted_language) vs. [compiled](http://en.wikipedia.org/wiki/Compiled_language) Clojure. I knew that eval wasn't safe for many reasons (including input validation and compile vs. run-time confusion), but wanted to see how this code compared in performance. So I tried to run a quick benchmark using [criterium](https://github.com/hugoduncan/criterium) ```quick-bench```, however it hung on my "bad" code (my patience may be at fault). So finally, using a hack-ier benchmarking:

{% highlight clojure %}
;; Interpreted
user> (time 
       (dotimes [_ 10000]
         (eval ((fn [l]
                  `(-> data
                       ~@(map (fn [f] `(~f)) l)))
                fns))))
"Elapsed time: 17538.103 msecs"
nil

;; Compiled
user> (time
       (dotimes [_ 10000]
         (reduce #(%2 %1) data fns)))
"Elapsed time: 55.195 msecs"
                   
;; And just to prove the code transforms the data the same
user> (eval ((fn [l]
               `(-> data
                    ~@(map (fn [f] `(~f)) l)))
              fns))
#{0 1 2 3 4 5 6 7 8 9 10 11 12 13}
user> (reduce #(%2 %1) data fns)
#{0 1 2 3 4 5 6 7 8 9 10 11 12 13}
{% endhighlight %}

The results are telling, compiled code ran about 300x as fast. I believe there is more to this deviation than interpreted vs. compiled (any comments are greatly appreciated). Regardless, this is enough to convince me to never use ```eval```.

_Hey, this is my first ever blog post!_  
**Please leave [comments](https://gist.github.com/MarkKettner/866b853b065896d9b138)**