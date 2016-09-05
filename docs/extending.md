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
promise as their argument, splitting the action `:FOO` into `:FOO/PENDING`
`:FOO/SUCCESS` and `:FOO/FAILURE` to signify when the request is made,
and when it ends in a success or an error, respectively.
We could wrap this process ourselves for each async request our application
would make, or write a piece of middleware to handle promises for us.

### Example: Verifying app model with cljs.spec
Similarly, if you wished to ensure that for each action, the model
retained some specific structure, you could implement a middleware function
to validate the resulting model against a clojure.spec schema.

### Writing middleware
**Signature:** `next -> model, action [, args ...] -> new-model`

We see from the signature that middleware takes a single argument '`next`', which
is the middleware/reducer function to execute next. From this, we get a function,
which, when given a '`model`', an '`action`' (and optionally some '`args`'),
yields a new model which is the result of applying the action to the old model.

```clojure
;; This middleware does nothing but pass the action along.
(defn passthrough-middleware
  [next]
  (fn [model action & args]
    (apply next model action args)))
```

### Using middleware
Middleware is used by composing it with your application's reducer using [composition (`comp`)](http://clojuredocs.org/clojure.core/comp)
or the [threading macro (`->`)](http://clojuredocs.org/clojure.core/-%3E) threading macro.

You can express the order of execution in whichever way feels the most natural:
```clojure
((-> my-reducer baz bar foo) ...)
;;execution order: foo, bar, baz, my-reducer

(((comp baz bar foo) my-reducer) ...)
;; execution order: baz, bar, foo, my-reducer
```

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

### Using Store Enhancers
The store function accepts an additional argument, allowing you to specify an enhancer function.
To use multiple store enhancers, use [composition](http://clojuredocs.org/clojure.core/comp).

```
(def s (store my-reducer make-stores-great-again))
(great-store? s)
;;=> true
```