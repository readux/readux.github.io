# Design Overview

## The store & the model
Readux aims to simplify state management by encouraging all state to be managed
by a *store* which holds a single entity, which we call the *model*.

Generally, we try to *normalize* the data in the model, that is, to organise it
in such a way, that everything is described just once.
This is desirable because a single source of truth means data can't get out of
sync - we can't have a user profile and the metadata associated a post show
two different profile pictures for the same user - because everything there is
to know about the user is described *exactly once* in our model.

## Actions & Updating the model
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

## Getting data from the model
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

# Notes

1) Technically a `reaction` - think of it as a atom that can do an arbitrary
   calculation which is only re-done when the data it depends on changes
   underneath it. Because we just return the model, it effectively works as
   a read-only atom.

