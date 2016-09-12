# Readux

Readux is a *small* (~100 LOC core before comments & asserts) library for managing state in web 
applications already using [reagent](https://github.com/reagent-project/reagent).

## Documentation is under construction...
This project is very new and the documentation you see is a first pass - certain examples mentioned in passing
(handling promises, spec validation, debugger) will become projects in their own right very soon.

Also, if you have a great idea, go ahead and implement it I will happily
mention your project in the docs :) -- Well, provided it is suitably open
(e.g. MIT/BSD/Eclipse Public License).

## Short(ish) description
Readux aims to simplify state management by encouraging all state to be managed by a *store* which holds a single entity, which we
call the *model*.

The only way to update the model is to dispatch actions against the store. An action is merely a keyword (e.g. `:ADD-TODO`)
and some (optional) data.
The point is for the dispatched actions to describe *what happened* and have the store contain the logic dictating what to 
do in response to those actions. This way, we hope to encourage the separation of application logic from the presentation.

To actually handle actions, the store takes a *reducer* function, named after the function argument to the 
[`reduce`](https://clojuredocs.org/clojure.core/reduce) function, a reducer is a function which, given an
an accumulated value (the old model) and some new input (the action) produces a new accumulated value.

In short, a reducer basically has the signature `reducer: model, action -> new-model`.

To read from the store, we may either use `store->model` to access to the entire model through a *reaction* (think read-only atom)
or write special named functions called *queries* which we can then use in various reagent components
to extract data from our model.

## Inspiration

  * [redux](https://redux.js.org)
  * [The introduction to reactive programming you've been missing](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
  * [re-frame](https://github.com/Day8/re-frame)
    * [README](https://github.com/Day8/re-frame/blob/master/README.md)
    * [WIKI: When do components update?](https://github.com/Day8/re-frame/wiki/When-do-components-update%3F)
  * [Elm (Lang)](http://elm-lang.org)
    * [Elm Tutorial](http://www.elm-tutorial.org/)
