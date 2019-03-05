---
title: OneRef - Simple Functional State Management for React
date: "2019-03-03T22:12:03.284Z"
---

<strong>UNPUBLISHED DRAFT</strong>: This is an unpublished draft. If you are reading this, it is hopefully because I invited you to review this. <strong>Please do not share the link to
this with others or on social media yet.</strong> Thank You!

---

> Everything should be made as simple as possible, but not simpler.
> --Albert Einstein

<strong>OneRef</strong> is a tiny (around 150 lines of code) state management library
built on top of React Hooks.
The goal of OneRef is to present
the simplest possible interface for state management to application programmers while retaining
sufficient expressive power to support real-world use cases such as interaction of application state with asynchronous platform APIs.

OneRef is fundamentally very similar to [Redux](https://redux.js.org/), and was originally developed
around the same time (2015). I developed the first version of OneRef to support my first production React application ([Tabli](https://chrome.google.com/webstore/detail/tabli/igeehkedfibbnhbfponhjjplpkeomghi), a tab manager extension for Google Chrome), and later used it in [Tad](https://www.tadviewer.com), a desktop application for viewing and analyzing CSV files.
At the time I completed the initial version of OneRef it seemed sufficiently similar to Redux that it didn't seem worth publicizing widely.

Recently, however, I rewrote OneRef for use with TypeScript and React Hooks. As part of this process, I struggled with a problem that still seems to present a significant challenge when using state management libraries: How to cleanly support application actions that need to interleave asynchronous platform operations with reading and updating application state. In Redux, this problem is addressed by middleware libraries such as [redux-thunk](https://github.com/reduxjs/redux-thunk) and [redux-saga](https://redux-saga.js.org/). For this latest iteration of
OneRef, I came up with an additional OneRef primitive, `awaitableUpdate`. This operation
seems to cleanly address this challenge of handling asynchronous operations, but could be controversial.
I'm writing this blog post mainly to present `awaitableUpdate` and solicit feedback from the React community.

---

<strong>A Quick Caveat:</strong> I happen to work at Facebook, the
company behind React. Facebook is a big place these days.
While I have immense admiration for the work of the React team,
I do not work with them directly. In fact, I don't even do React
development as part of my job.
The work reported here
was done entirely on my own time as part of
maintaining [Tabli](https://chrome.google.com/webstore/detail/tabli/igeehkedfibbnhbfponhjjplpkeomghi) and [Tad](https://www.tadviewer.com), my hobby side projects.
Any and all mistakes, terrible ideas, conclusions, opinions or recommendations included here are entirely my own, not those of my employer or any other teams.

---

## The OneRef Architecture

(Include tl;dr for those famililiar with Redux)

The OneRef architecture looks roughly like this:

![OneRef Architecture](./oneref-architecture.png)

The application state (`AppState`) is represented as a purely functional (immutable) data structure.
I use [Immutable.JS](https://immutable-js.github.io/immutable-js/) for this purpose, but this is not a strict requirement. At the top level of a OneRef application, there is a single mutable ref cell
(StateRef) that holds the current application state.

Application state is updated in OneRef by calculating a new AppState from the existing AppState and setting the top level Ref to this new AppState. OneRef provides a single entry point, `update`, for this purpose:

```typescript
type StateTransformer<T> = (s: T) => T;
type StateRef<T> = ...; // Abstract type
function update<T>(ref: StateRef<T>, tf: StateTransformer<T>) { ... }
```

This results in notifying listeners, including the top level View, which renders the updated AppState. Immutable.js ensures that constructing the new AppState is efficient by optimizing sharing and re-use with the existing AppState, while React's Virtual DOM ensures the efficiency of rendering this updated AppState. 
The other components of the architecture are as follows:
An updater (sometimes called a refUpdater) is a helper function that updates the application state by setting the top level ref cell. 
The React Views are ordinary React views. A high level View component accepts the AppState and a refUpdater as properties and passes these along to child components via properties as needed. 
The Actions in OneRef are ordinary JavaScript functions called by event handlers in view components. They typically live in a source file outside of the React views and take whatever parameters are needed to perform the action plus an additional refUpdater parameter to enable the action to update the application state.

## TodoMVC in OneRef

## Hello, Async

## Composition and Reuse

## Initialization Effects and Asynchronous Subscriptions

## Other concerns

### Testing

### Performance