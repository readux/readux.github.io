# Design

## Overview

### The store & the model
Readux aims to simplify state management by encouraging all state to be managed
by a *store* which holds a single entity, which we call the *model*.

Generally, we try to *normalize* the data in the model, that is, to organise it
in such a way, that everything is described just once.
This is desirable because a single source of truth means data can't get out of
sync - we can't have a user profile and the metadata associated a post show
two different profile pictures for the same user - because everything there is
to know about the user is described *exactly once* in our model.

### Actions & Updating the model
The only way to update the model is to dispatch actions against the store.
An action is merely a keyword (e.g. `:add-todo`) and some (optional) data.
The point is for the dispatched actions to describe *what happened* and have
the store contain the logic dictating what to do in response to those actions.
This way, we hope to encourage the separation of application logic from the
presentation.

The store needs some way to handle the actions, to deside how the model 
should be changed to reflect an `:add-todo` action. That's the job of
the *reducer* function.
The reducer takes two arguments, the old model (i.e. the state of our
entire application as of this moment) and the action, and from these
constructs a new model.
The only important requirement here is that the reducer is *pure*, that is,
given the same input, we must always get the same output.

### Getting data from the model
It's possible to call `store->model` on a store to get a (read-only) atom(1).
However, using that approach is bad for two reasons:

  1) Every component is now aware of, and *depends on* how our model is structured
     right now.

  2) We will quite possibly repeat and re-run code which transforms the raw data
     into a format suitable for our components.

In readux you're encouraged to use *queries* to extract data from the store and
transform it into some format more suitable to for one or more reagent components.
By writing these query functions, we solve (1) in that changing the structure of
the store only requires rewriting our queries and (2) because each query should
only be computed when necessary, more on that later.

## Extending readux

### Of libraries and frameworks
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

### How to extend readux
In concrete terms, readux allows writing [*store enhancers*](extending.md#store-enhancers) augment the store,
adding additional functionality like 'queries', debuggers etc and [*middleware*](extending.md#middleware),
which sits between the call dispatching an action and it being forwarded to the
reducer.
Generally, if you mean to simply alter actions on a case-by-case (pure) basis,
middleware will do. If you need to retain some state, e.g. to store a map of
registered queries or data related to a debugger, then writing a store enhancer
is the way to go.

# Reference
## Action
Actions in readux follow the structure of [flux standard actions (FSA)](https://github.com/acdlite/flux-standard-action).

An FSA-action must not have any keys beside `:type`, `:payload`, `:error` and `:meta`, and only
`:type` is required.

* `:type`- the type of the action, e.g. `:add-todo`, `:show-editor`
* `:payload` - any data which should be sent along with the action.
* `:error`
    * optional boolean field
    * analguous to a rejected promise (i.e. processing the action failed at some step). 
    * If `:error` is set, the `:payload` should contain an error object describing what went wrong.
* `:meta` 
    * intended to hold data not directly pertaining to the action.

```clojure
;; Example action.
{:type :add-todo
 :payload "Buy milk"}
```

This might seem constricting, but having a standard format makes it simpler to write
middleware and store enhancers which work on actions - and ultimately makes readux 
simpler to use since middleware functions the same and interoperate better.

## Reducer 
**Signature:** `model, action -> new-model`

A reducer is a function which takes a model and some action, producing from
them a new model. This might sound complex, but it's actually not, it works
exactly like `(reduce + 0 [1 2 3 4])`. The result of [reduce](), `10` in 
this case, is the result of continuously using the `+` function on the
previous result and the next value in line - and that's exactly how the
reducer function is meant to work.

Because of the way reducers can be composed together, there's a few requirements
which a reducer must meet:

  1. **Purity** - Given the same input (model & action) the reducer must always produce the same result
  2. **Handle nil input** - When the first action is dispatched, the model is initially empty. The reducer must treat a nil-value for a model as a queue to initilize it.
  3. **Return a model** - If an action doesn't apply to the reducer, simply return the model as it was.

It turns out that obeying these rules means we can build reducers which
delegates parts of their model to other reducers. But first, let's see
a simple reducer:

### Writing simple reducers
Here's a small example reducer for a counter:
```clojure
(defn counter-reducer 
  [model action]
  (let [model (or model {:value 0})] ;; initialize if needed
    (case (:type action)
      :increment (update model :value inc)
      :decrement (update model :value dec)
      model))) ;; return model (i.e. no change) as a fall-back
```

Notice how the original model is returned if the action was neither 
`:increment` nor `:decrement`. Also notice how `model` is initialised if a nil
value is received.

Reducers of this kind are common, that's why readux provides `reducer-fn` to
help writing reducers which handle requirements 2 & 3.

```clojure
(def counter-reducer
  (reducer-fn
    [model action]
    {:value 0}
    {:increment (update model :value inc)
     :decrement (update model :value dec)}))
```

### Combining reducers

**TODO** Write an example.

## Query
Consider a blog post which shows the name, a blurp and a profile picture of the
author - we don't wan't to store this with every post - we want a single source
of truth such that a new profile picture requires one change to be reflected
across the page.
That means we'll tend to normalize our data and have a model like so:
```
{:users {0 {:name "Bob"
            :avatar "http......"}
         1 {:name "Sarah"
            :avatar "http......"}}
 :posts {100 {:author 0
              :title "On the duplicitous nature of cats"
              :text "...."}
         101 {:author 1
              :title "Monads are for newbs"
              :text "...."}
         102 {:author 1
              :title "Winter 2016 Anime List"
              :text "..."}}}
```

In Readux, you're encouraged to use *queries*, which are functions operating
on the store, producing some output which fits the needs of one or some 
components.

### Using queries
E.g., for our posts component, we might have a query 'fetch-post &lt;id&gt;'
which produces:

```clojure
(rdc/query store [:fetch-post 101])
;;=>
{:author {:name "Sarah"
          :avatar "http...."}
 :id 101
 :title "Monads are for newbs"
 :text ""}
```



That is, queries are used to:

  1. Avoid coupling the structure of the store to each component which reads from it
  2. Avoid re-running the computation required to transform the data more often than necessary (i.e. run when the model changes).

### Writing queries
Continuing from the example above, let's actually implement it. Adding a new
query takes two steps; defining the query function and registering it with the
store.

#### Writing the query function
```clojure
;; Example: defining the `post` query.
(defn post-query
  [model [query-id post-id]]
  (assert (= query-id :fetch-post))
  (let [users (reaction (:users @model))
        posts (reaction (:posts @model))]
    (-> (get @posts post-id)
        (update :author #(get @users %))
        (reaction))))
```

Notice how we start by extracting two smaller parts of the model, `:users` and
`:posts` from the model into separate reactions before transforming the data
into the form we'd like to use.
This leverages the nature of reactions to ensure our query is only re-run if
the content of the `:posts` or `:users` fields change - in this way, we can
scale our queries to work well even in applications with a gigantic model which
gets updated very frequently. Our query only gets re-run when the subtrees
`:posts` or `:users` change in the model.

This example is somewhat contrived; our query is simple and it likely won't
matter much. The point, however, is that expensive queries such as sorting a
set of items or say, rendering post content stored as markdown into HTML,
can be tweaked to run only when necessary by making them depend on
reactions which expose only the data they should operate on.

#### Registering the query with the store
Secondly, the query needs to be registered with the store:
```clojure
;; Registering the 'post' query with the store.
(rdc/query-reg! store :post fetch-post-query)
```

Notice how we register the query under an id (in this case, `:post`). This is
the id we use to refer to the query from within our components when we
write something like `(rdc/query store [:fetch-post 101])`.

Because you typically want to register a lot of queries right away, readux
provides a helper function, `queries-reg!`:
```clojure
(queries-reg! {:post post-query
               :posts posts-index-query})
```

# Notes

1) Technically a `reaction` - think of it as a atom that can do an arbitrary
   calculation which is only re-done when the data it depends on changes
   underneath it. Because we just return the model, it effectively works as
   a read-only atom.

