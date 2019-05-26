```typescript
const init: AppStateEffect<TodoAppState> = (
  appState: TodoAppState,
  stateRef: StateRef<TodoAppState>
) => {
  const serviceIter = onerefUtils.publisherAsyncIterable(todoServer.subscribe);
  const stIter = onerefUtils.aiMap(serviceIter, actions.createTodo);
  onerefUtils.updateFromIterable(stateRef, stIter);
};
```

The body of **init** is a sequence of three calls to utility routines in **onerefUtils**. The first two calls (to **publisherAsyncIterable** and **aiMap**) are generic utilities that are useful but not specifically related to
oneref. The call to **publisherAsyncIterable** converts a callback based subscription API into an [AsyncIterable](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-3.html). The result, bound to **serviceIter**, is of type **AsyncIterable&lt;string&gt;**. The call to **aiMap** is a simple **map** operation over AsyncIterables, applying a function to every element of an async iterable. In this case, we pass **actions.createTodo**, mapping every string from the iterable
to a **StateTransformer&lt;TodoAppState&gt;**; hence **stIter** has type **AsyncIterable&lt;StateTransformer&lt;TodoAppState&gt;&gt;**.

Finally, **stIter** is passed to **updateFromIterable**. The implementation of **updateFromIterable** is an async
function that invokes the primitive oneref **update** operation on every state transformer function delivered by
the async iterable:

```typescript
export async function updateFromIterable<AS>(
  ref: StateRef<AS>,
  stream: AsyncIterable<StateTransformer<AS>>
): Promise<void> {
  for await (const st of stream) {
    update(ref, st);
  }
}
```

All of these functions live in **oneref.utils** because they are not really primitive functionality; all that's really
happening here is that the oneref **update** primitive is called from an asynchronous context after the **init** function has completed. While I find this use of **AsyncIterable** a natural and concise way to think about subscriptions, the
