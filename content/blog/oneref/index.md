---
title: OneRef - Simple Functional State Management for React
date: '2019-03-03T22:12:03.284Z'
description: A detailed description of the OneRef state management library
---

<strong>UNPUBLISHED DRAFT</strong>: This is an unpublished draft. If you are reading this, it is hopefully because I invited you to review this. <strong>Please do not share the link to
this post with others or on social media yet.</strong> Thank You!

---

> Everything should be made as simple as possible, but not simpler.
> --Albert Einstein

<strong>OneRef</strong> is a tiny (around 150 lines of code) state management library
built on top of React Hooks.
The goal of OneRef is to present a programming interface for state management that is as
simple as possible while still providing
sufficient expressive power to support real-world use cases such as interaction of application state with asynchronous platform APIs.

I originally developed in OneRef in 2015 for use in my first production React application
([Tabli](https://chrome.google.com/webstore/detail/tabli/igeehkedfibbnhbfponhjjplpkeomghi), a tab manager extension for Google Chrome), and later used it in [Tad](https://www.tadviewer.com), a desktop application for viewing and analyzing CSV files. By the time I completed the initial version of OneRef,
[Redux](https://redux.js.org/) had gained considerable traction and established itself as a de facto
standard for React state management. At the time I decided that Redux was sufficiently similar to OneRef that OneRef wasn't worth publicizing widely.

I recently rewrote OneRef for use with TypeScript and React Hooks. As part of this process, I struggled with a problem that still seems to present a significant challenge when using state management libraries: How to cleanly support application actions that need to interleave asynchronous platform operations with reading and updating application state. In the Redux ecosystem this problem is
addressed by middleware libraries such as [redux-thunk](https://github.com/reduxjs/redux-thunk) and [redux-saga](https://redux-saga.js.org/). For this latest iteration of
OneRef, I came up with an additional OneRef primitive, **awaitableUpdate**. This operation
seems to cleanly address this challenge of handling asynchronous operations, but could be controversial.
I'm writing this blog post mainly to present awaitableUpdate and solicit feedback from the React community.

---

<strong>A Quick Caveat:</strong> I happen to work at Facebook, home of the
React team.
While I have immense admiration for the work of the React team,
I do not work with them directly. In fact, I don't even do React
development as part of my job.
The work reported here
was done entirely on my own time as part of
maintaining [Tabli](https://chrome.google.com/webstore/detail/tabli/igeehkedfibbnhbfponhjjplpkeomghi) and [Tad](https://www.tadviewer.com), my hobby open source side projects.
Any and all mistakes, terrible ideas, conclusions, opinions or recommendations included here are entirely my own, not those of Facebook or any Facebook team.

---

## The OneRef Architecture

This section presents an overview of OneRef. Readers already familiar with Redux can lightly skim this section and the next one, and resume reading at [Differences from Redux](#differences-from-redux).

The OneRef architecture looks roughly like this:

![OneRef Architecture](./oneref-architecture.png)

The application state (AppState) is represented as a purely functional (immutable) data structure.
I use [Immutable.JS](https://immutable-js.github.io/immutable-js/) for this purpose, but this is not a strict requirement; any immutable data structure representation should do.
At the top level of a OneRef application, there is a single mutable ref cell
(StateRef) that holds the current application state.

Application state is updated in OneRef by calculating a new AppState from the existing AppState and setting the top level StateRef to this new AppState. OneRef provides a single entry point, update, for this purpose:

```typescript
type StateTransformer<T> = (s: T) => T;
type StateRef<T> = ...; // opaque
function update<T>(ref: StateRef<T>, tf: StateTransformer<T>) { ... }
```

The update function in OneRef behaves exactly like the [functional update](https://reactjs.org/docs/hooks-reference.html#functional-updates) variant of [useState](https://reactjs.org/docs/hooks-reference.html#usestate) in React Hooks. The only slight difference is the ref parameter
that is passed down through React views and into event handler callbacks. This similarity is no
accident; The update operation is implemented internally in OneRef using setState from
a top level useState hook.

Actions in OneRef are ordinary JavaScript functions called by event handlers in view components. They typically live in a source file outside of the React views, take whatever parameters are needed to perform the action, and calculate a (pure) StateTransformer function passed to the update function to update the
application state.

## TodoMVC in OneRef

To make the previous discussion a bit more concrete, let's walk through the implementation of a complete example -- the classic Todo list application of [TodoMVC](http://todomvc.com/). The complete working
example is available in the OneRef examples repository at [...]().

![TodoMVC screenshot](./todomvc-screenshot.png)

To begin with, let's define an immutable representation of application state using [Immutable.JS Records](https://immutable-js.github.io/immutable-js/):

First we have the representation of individual items in the Todo list:

```typescript
import { Record } from 'immutable';

interface ItemProps {
  id: string;
  complete: boolean;
  text: string;
}

const defaultItemProps: ItemProps = {
  id: '',
  complete: false,
  text: '',
};

/* auxiliary function to generate a fresh id */
function genID() {
  return (+new Date() + Math.floor(Math.random() * 999999)).toString(36);
}

export default class TodoItem extends Record(defaultItemProps) {
  constructor(text: string, complete = false) {
    super({ id: genID(), text, complete });
  }
}
```

The Application State (AppState) is an immutable record, with a single field, **todoItems**, an (immutable) Map from item id to items:

```typescript
import { Map, Record, Seq } from 'immutable';
import TodoItem from './todoItem';

interface AppStateProps {
  todoItems: Map<string, TodoItem>;
}

const defaultStateProps: AppStateProps = {
  todoItems: Map(),
};

export default class TodoAppState extends Record(defaultStateProps) {
  /**
   * Add / update a TODO item
   *
   * functional item update -- returns a new state with the given item included in the
   * set of todo items.  If there is an existing entry for item.id, the result state
   * will map id to item (functional update).
   */
  addItem(item: TodoItem): TodoAppState {
    const nextTodoItems = this.todoItems.set(item.id, item);
    return this.set('todoItems', nextTodoItems);
  }

  /**
   * functional delete -- returns a new state with the item for the given id removed
   */
  removeItem(id: string): TodoAppState {
    const nextTodoItems = this.todoItems.delete(id);
    return this.set('todoItems', nextTodoItems);
  }

  /** An Immutable.Seq of all todo items */
  getAll(): Seq.Set<TodoItem> {
    return this.todoItems.toSetSeq();
  }

  /* returns true iff all items are complete */
  areAllComplete(): boolean {
    return this.todoItems.every(item => item.complete);
  }
}
```

**TodoAppState** includes a few convenience methods, such as **addItem**. These methods
are all functional in the sense that they do not perform in-place updates; instead they return a
new **TodoAppState** object. Immutable.JS uses structural sharing to ensure that this is relatively efficient. Thus far, all of this is independent of OneRef.

The UI is implemented as a set of functional React components. A representative example is **Header**, which contains the text entry component used to add an item to the todo list:

```typescript
import React from 'react';
import TodoAppState from '../todoAppState';
import TodoTextInput from './TodoTextInput';
import * as actions from '../actions';
import { StateRef, update } from 'oneref';

interface HeaderProps {
  stateRef: StateRef<TodoAppState>;
}

const Header = ({ stateRef }: HeaderProps) => {
  const onSave = (text: string) => {
    if (text.trim()) {
      update(stateRef, actions.create(text));
    }
  };

  return (
    <header className="header">
      <h1>todos</h1>
      <TodoTextInput
        className="new-todo"
        placeholder="What needs to be done?"
        onSave={onSave}
      />
    </header>
  );
};
export default Header;
```

The props for this component includes **stateRef**, which is passed down the component hierarchy. The **onSave** callback for the text entry component calls **update** to update the
application state, passing in **stateRef** and the result of **actions.create(...)**.

The **actions** module contains the actions such as **create** that return **StateTransformer**
functions (pure functions from AppState to AppState) that can be passed to **update**:

```typescript
import TodoItem from './todoItem';
import TodoAppState from './todoAppState';
import { StateTransformer } from 'oneref';

export const create = (text: string): StateTransformer<TodoAppState> => state =>
  state.addItem(new TodoItem(text));

export const clearCompleted: StateTransformer<TodoAppState> = state => {
  const completedIds = state
    .getAll()
    .filter(item => item.complete)
    .map(item => item.id);
  return completedIds.reduce((s, id) => s.removeItem(id), state);
};

export const updateText = (
  item: TodoItem,
  text: string
): StateTransformer<TodoAppState> => state =>
  state.addItem(item.set('text', text));

export const toggleComplete = (
  item: TodoItem
): StateTransformer<TodoAppState> => state =>
  state.addItem(item.set('complete', !item.complete));

export const toggleCompleteAll: StateTransformer<TodoAppState> = state => {
  const targetVal = !state.areAllComplete();
  // We'll set completed state of all items to targetVal:
  const updItems = state.getAll().map(item => item.set('complete', targetVal));
  const nextState = updItems.reduce((st, item) => st.addItem(item), state);
  return nextState;
};

export const destroy = (id: string): StateTransformer<TodoAppState> => state =>
  state.removeItem(id);
```

To tie everything together, at the top level the application creates an initial
**TodoAppState** instance, and calls **oneref.appContainer** to construct a top-level
React component. The argument to **appContainer** is a functional React
component that will receive **stateRef** as one of its props:

```typescript
import React from 'react';
import ReactDOM from 'react-dom';
import * as oneref from 'oneref';
import TodoListEditor from './components/TodoListEditor';
import TodoAppState from './todoAppState';

import 'todomvc-common/base.css';
import 'todomvc-app-css/index.css';

const initialAppState = new TodoAppState();

const TodoApp = oneref.appContainer<TodoAppState>(
  initialAppState,
  TodoListEditor
);

ReactDOM.render(<TodoApp />, document.getElementsByClassName('todoapp')[0]);
```

### Differences from Redux

For those familiar with Redux, the biggest difference ..

## Hello, Async!

## Composition

## Initialization Effects and Asynchronous Subscriptions

## Other concerns

#### Testing

#### Performance

```

```
