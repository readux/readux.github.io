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
Sometimes we create a piece of functionality which we'd like to reuse in
different parts of the application.Examples would be a standardized yes/no
modal, a document editor, or maybe just a nicely styled drop-down menu with
fuzzy search completion.

In traditional GUI-frameworks, larger composite widgets could be made from
smaller base-widgets, their internal events properly connected and exposed
as a single widget.

Readux allows the same style of reuse through the concept of *context*.
Contextualising does two things:

  1. on dispatch, actions whose type lack a context, are prefixed with the context
    * e.g. given context `:work-list`, `:add-todo` becomes `:work-list/add-todo`.
    * NOTE: already contextualized actions, e.g. `:app/login` won't be altered.
  2. queries lacking a context, are prefixed with the context
    * e.g. given context `:work-list`, `:todos` becomes `:work-list/todos`.
    * NOTE: already contextualized queries, e.g. `:projects/list`, won't be altered.

Think of action types and query id's as akin to pathnames on a unix system.
If the keyword lacks a namespace, they are *relative* to whatever context the
component is used in.
If the keyword has a namespace, it's like an absolute path and will resolve
to the same thing regardless of the context in which the component is used.

To accomplish this, the `query` and `dispatch` functions are passed directly to
the component. The interface mirrors their respective functions except the
`store` argument is omitted.

It's perfectly possible to write reusable components without contextualising
them. Though if you do so, your components will need to receive a series of
callbacks for the dispatches and queries needed.
The drawback of that model is most noticable when components are nested in
eachother, though for relatively simple, flat components, it's a viable
alternative.

### Contextualising a component
To contextualize a component, use `readux.core/connect`. `connect` takes
a component, a store, the context and a path into the model.

Note that readux doesn't enforce any link between context and path into
the model - different reducers, and thus different parts of the model,
may want to react to the same action.

However, you generally want to ensure that the path to the reducer reacting
to (the contextualized) actions dispatched from the component matching the
path given to the connected component.  

```
;; Connect component 'counter-ctrl' to context ':counter1' operating
;; on the data (get-in model [:counters :1])
(rdc/connect counter-ctrl store :counter1 [:counters :1])
```

### Example - multiple counters
The code snippets below are extracted from the readux "counter" example.

Examine the [full source](https://github.com/readux/readux/tree/master/examples/counters),
check it out and issue `boot dev` to run it. Keep in mind, the example
contains additional comments.


Given a reducer managing two counters:

```clojure
(def app-reducer
  (rdc/composite-reducer
    {:counters
     {:1 {:init {:counter 11}
          :actions counter-actions
          :ctx :counter1}
      :2 {:init {:counter 20}
          :actions counter-actions
          :ctx :counter2}}
     :num-actions num-actions-reducer}))
```

We create a controller component, that is, a component wires up our
presentational component(s) to our application logic. In this case
by making some action dispatch and query callbacks.

```clojure
(defn counter-ctrl
  [dispatch query]
  (let [value (query [:counter-value])
        on-inc #(dispatch {:type :incr})
        on-dec #(dispatch {:type :decr})]
    [counter value on-inc on-dec]))
```

Finally, we connect two counters to different contexts and display them both:

```clojure
(defn- app
  [store]
  (let [counter1 (rdc/connect counter-ctrl store :counter1 [:counters :1])
        counter2 (rdc/connect counter-ctrl store :counter2 [:counters :2])
        num-actions (rdc/query store [:num-actions])]
    (fn app-render []
      [:div
       [counter1]
       [counter2]
       [:p (str "processed '" @num-actions "' actions so far.")]])))
```
