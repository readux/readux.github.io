# Design

## Overview

## Terminology

## Of libraries and frameworks
Readux, like [Redux](http://redux.js.org), is a fairly unopinionated *library*. 
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

In summary, Readux doesn't do much, but it's designed to play well with
other libraries, and that's a good thing(<i class="fa fa-trademark" aria-hidden="true"></i>).
