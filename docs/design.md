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
simpler to use since middleware becomes more readily composable.

## Action Maps
Reducers, covered in detail below, act on the model in response to incoming actions.

To do that, the reducer will accept a map, which map actions (by their type) to
a corresponding function. Think of an action map as a series of "if X, then do Y".
If this seems

Suppose our application has a counter which can be incremented or decremented
by one - the application's model looks like so:

```clojure
{:counter 0}
```

Asuming we name the increment and decrement actions `:increment` and `:decrement`
respectively, our action map would look something like this:

```clojure
{:increment
 (fn [model action]
   (update model :counter inc))
 :decrement
 (fn [model action]
   (update model :counter dec))}
```

### Tip: Reuse your action maps!
It will take a while to fully appreciate, but wrapping our actions into a map
will prove helpful later when building reducers and later still when making
sets of components to be reused across the application.

Imagine two forms sharing 

, as can be merged (allowing 
reuse of identical subsets of functionality) and their keys can be mangled
to contextualize actions - meaning not all counters are updated whenever
an `:increment` action is dispatched.

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

Reducers of this kind are common, that's why readux provides `reducer` to
help writing reducers which handle requirements 2 & 3.

```clojure
(def counter-reducer
  (rdc/reducer
    {:value 0}
    {:increment (fn [model action]
                  (update model :value inc))
     :decrement (fn [model action]
                  (update model :value dec))}))
```

### Combining reducers
Let's assume we're making a simple todo application.

Assume we want to build a todo application. We might want two reducers, one
managing a list of todos (each represented as a map) and another managing
a display filter - which allows us to display all, only completed or active
todos.

```clojure
(def todo-reducer
  (rdc/reducer
  {:todos [{:id 1
            :text "read the readux tutorial"
            :completed false}]}
  todo-actions))

(def filter-reducer
  (rdc/reducer
  :show-all
  filter-actions))
```

#### A composite reducer
Let's say we want the model state to look like so:

```clojure
{:todos [{:id 1
          :text "read readux tutorial"
          :completed false}]
 :filter :show-all}
```

While we could rewrite our two reducers into one big reducer, manually handling
the creation of the nested map that is our model, that would be wasted effort.

Instead, we can make a reducer, which is the two other reducers composed:

```clojure
(def app-reducer
  (rdc/composite-reducer
    {:todos  todo-reducer
     :filter filter-reducer}))
```

#### A composite reducer - described as data
`composite-reducer` also allows you to forego defining each individual reducer.
After all, reducers just require an initial model and a set of actions. So we
could also define our composite reducer like so:

```clojure
(def app-reducer
  (rdc/composite-reducer
    {:todos  {:init {:todos []
                     :filter :show-all}
              :actions todo-actions}
     :filter {:init :show-all
              :actions filter-actions}}))
```

This turns out to be especially helpful in larger apps where the app reducer
consists of many smaller reducers composed together.

#### Nested Composite Reducers
The result of calling `composite-reducer`is itself a reducer, so it's possible
to keep nesting reducers as desired. However, having these strewn about as
separate `def`'s in our source-code makes it difficult to see which part of the
model they manage.

However, `composite-reducer` allows you to write a deeply nested tree of
reducers as one big map - making it much easier to visually discern which part
of the model a given reducer manages.

Let's assume we wanted to expand our todo app to managing multiple todo lists,
one for each project. In that case, we may want the state to be:

```clojure
{:projects
 {:work
  {:label "Work Todos"
   :todos
   [{:id 1
     :text "read readux tutorial"
     :completed false}]
   :filter :show-all}
  :home
  {:label "House Chores"
   :todos
   [{:id 1
     :text "locate source of smell in the fridge"
     :completed false}]
   :filter :show-active}}
 :filter #(:work :home)}
```

With `composite-reducer` we can express that like so:

```clojure
(def app-reducer
  (rdc/composite-reducer
    {:projects
     {:work {:init todo-model
             :actions todo-actions
             :ctx :work}
      :home {:init todo-model
             :actions todo-actions
             :ctx :home}}
     :filter project-filter-reducer}))
```

`composite-reducer` automatically recognizes that the map supplied to `:projects`
isn't meant to be a reducer (it's lacking `:init` & `:actions`) - so it treats
it as a composite reducer in its own right.

Note that you can take this as far as you want, and note how our app's reducer
visually resembles the model we want it to manage :)

You may have noticed the `:ctx` key in both maps describing the reducers for
the work- and personal todo-lists.
Context essentially allows use to reuse the same actions and components in a
different part of the application (*context*) and is a mechanism for
facilitating reuse.

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

## Context - reusing work
Note that we use the `:ctx` key to set a context for the two reducers
managing a set of todos. We do this to differentiate between `:add-todo` actions
operating on our "Work" todo-list from our house chores.

Context is used to prefix actions with a unique namespace, this ensures that an
`:add-todo` action that is dispatched is only processed by the intended todo-list.

Reusing components and dispatching actions in the right context, is what allows
reusing components multiple places in an application - this is covered in the
section on [context](#context-reusing-work).

# Notes

1) Technically a `reaction` - think of it as a atom that can do an arbitrary
   calculation which is only re-done when the data it depends on changes
   underneath it. Because we just return the model, it effectively works as
   a read-only atom.

