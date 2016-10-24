# Extending Readux
This document assumes familiarity with what's written intended
the [design overview](design.md) and [reference](reference.md) pages, 
which explain the rationale and introduces core terminology.

## Of libraries and frameworks
readux, like [redux](http://redux.js.org), is a fairly unopinionated *library*. 
Libraries are limited in scope whereas frameworks are cohesive, thoughtful, but
*prescriptive* solutions to a set of problems.
Frameworks are a great thing when your use-case aligns perfectly with
what the authors intended - less so when you're fighting them to achieve
something not envisioned or catered to.

Conversely, libraries are concerned with solving a relatively small problem
and you're left to assemble the libraries needed to solve your problem.
The upside is that you can tailor a solution matching the problem.

For example, the Redux community has multiple takes on managing async
requests, from [thunks](https://github.com/gaearon/redux-thunk) to 
[promises](https://github.com/acdlite/redux-promise) through 
[generators](https://github.com/yelouafi/redux-saga).
There is no universal, *right* (<i class="fa fa-trademark" aria-hidden="true"></i>) solution - the best approach is relative
to the scale and complexity of your application. 
Even if there were, using a set of libraries means that once some superior
approach to solving the problem comes along, you can swap out a library
instead of starting all over in a new framework.

In summary, readux doesn't do much, but it's designed to play well with
other libraries, and that's a good thing(<i class="fa fa-trademark" aria-hidden="true"></i>).

## How to extend readux
In concrete terms, readux allows writing [*store enhancers*](extending.md#store-enhancers) augment the store,
adding additional functionality like 'queries', debuggers etc and [*middleware*](extending.md#middleware),
which sits between the call dispatching an action and it being forwarded to the
reducer.
Generally, if you mean to simply alter actions on a case-by-case (pure) basis,
middleware will do. If you need to retain some state, e.g. to store a map of
registered queries or data related to a debugger, then writing a store enhancer
is the way to go.

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