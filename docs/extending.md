# Extending Readux

This document assumes familiarity with what's written intended
[the design](design.md), which explains the rationale and introduces core
terminology.

In general, Readux is intended to be extended through middleware and store
enhancers, both of which are described below.

## Middleware
Middleware is used whenever you wish to automatically do something
before an action reaches your store's root reducer or before the
resulting model is persisted.

### Example: Handling Promises
Say we wanted to automatically process incoming actions which submits a
promise as their argument, splitting the action `:fetch-posts` into 
`:fetch-posts.rq`, `:fetch-posts.success` and `:fetch-posts.error` to signify
when the request is made, and when it ends in a success or an error,
respectively. We could wrap this process ourselves for each async request our
application would make, or write a piece of middleware to handle promises for us.

*Incidentially, if you wish to do this, or already use [promesa](http://funcool.github.io/promesa/latest/),
try [readux-promesa](https://github.com/readux/readux-promesa)*.

### Example: Verifying app model with cljs.spec
Similarly, if you wished to ensure that for each action, the model
retained some specific structure, you could implement a middleware function
to validate the resulting model against a clojure.spec schema.

### Writing middleware
**Signature:** `next -> model, action -> new-model`

We see from the signature that middleware takes a single argument '`next`', which
is the middleware/reducer function to execute next. From this, we get a function,
which, when given a '`model`' and an '`action`', yields a new model which is the
result of applying the action to the old model.

```clojure
;; This middleware does nothing but pass the action along.
(defn passthrough-middleware
  [next]
  (fn [model action]
    (apply next model action)))
```

Often times, you only need to do something to an action or in response to
an action *before* it is received by the reducer, but sometimes it can
be handy to work on the resulting model too.

Readux ships with a middleware function, `log-model-diff` which can show you
every action that is dispatched and how it changed the model. 
`log-model-diff` stores a copy of the model before passing it on to the next
middleware function in the chain, ideally the reducer itself, comparing the 
resulting model to the initial model.
Try reading its [source](https://github.com/readux/readux/blob/master/src/cljs/readux/middleware/log_model_diff.cljs)
for inspiration on writing your own middleware functions.

Remember, middleware functions can:

  1. prematurely abort processing simply by not calling the next function in line.
  2. alter/replace and dispatch additional actions. 
    - (see [readux-promesa](https://github.com/readux/readux-promesa/blob/master/src/cljs/readux_promesa/core.cljs) which uses this to handle promises)
  3. alter/replace the input model before passing it on to the next function
  4. alter/replace the new model before returning it to the caller (ultimately the `dispatch` function, which stores the new model)


### Using middleware
To use middleware, we supply `store` with an optional second argument
which modifies the construction of the store somehow before it is
returned to us.

In this case, using `apply-mw` allows us to install an arbitrary sequence
of functions in between `dispatch` which first receives and action, and
the store's reducer function. Here's an example:

```clojure
(require '[readux.core :as rdc])
(require '[readux.store :as rds])
;; ...
(defonce store (rdc/store app-reducer (rds/apply-mw some-other-middleware log-model-diff)))
```

In this case, the action will go through the system like so:

  1. `dispatch` this is when the action is first dispatched 
  2. `some-other-middleware`
  3. `log_model_diff`
  4. `app-reducer` - the application's reducer function.

## Store enhancers
**Signature:** `store -> store`

Store enhancers, as their name implies, modify the store object itself. Generally, store enhancers should be used when
middleware won't suffice, such as when new functionality relying on additional fields in the store object is implemented.

### Example: readux-debugger
For example, the readux-debugger creates an additional store, tied to the app-store which controls its own model and which
has its own reducer. By enhancing the store, it can replace the standard dispatch-function invoked when an action is 
dispatched.
The dispatch function normally finds the store's reducer, and passes along the action, storing the value returned as the
new model.
By replacing the dispatcher, the readux-debugger can intercept and assure that debug-related actions never reach the
app reducer while ensuring app-related actions are dispatched to both the debugger and app stores.

(**NOTE** not presently released, coming soon)

### Writing Store Enhancers 

The smallest possible example is also a bit silly, typically you want to wrap the `:dispatch` function or add some
additional entries to facilitate some expanded interface.

```clojure
(defn make-stores-great-again
  [app-store]
  (swap! app-store assoc :is-great? true))
```

With our additional data, we can implement new store functions, such as this:

```
(defn great-store?
  [store] 
  (get @store :is-great? false))
```

The ability for a store to use middleware functions is actually implemented as
a store enhancer, see `apply-mw` in [readux store.cljs](https://github.com/readux/readux/blob/master/src/cljs/readux/store.cljs).

### Using Store Enhancers
The store function accepts an additional argument, allowing you to specify an enhancer function.
To use multiple store enhancers, use [composition](http://clojuredocs.org/clojure.core/comp).

```
(def s (store my-reducer make-stores-great-again))
(great-store? s)
;;=> true
```