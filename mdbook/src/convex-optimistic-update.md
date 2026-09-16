# How Optimistic Update works in Convex

### API

In Convex, optimistic updates are local changes that can modify the query result. The `withOptimisticUpdate` method returns a function that takes the `args` and returns a function that can enqueue an optimistic update waiting to be sent to the server. The optimistic update modifies the query results directly.

```typescript
export function IncrementCounter() {
  const increment = useMutation(api.counter.increment).withOptimisticUpdate(
    (localStore, args) => {
      const { increment } = args;
      const currentValue = localStore.getQuery(api.counter.get);
      if (currentValue !== undefined) {
        localStore.setQuery(api.counter.get, {}, currentValue + increment);
      }
    },
  );

  const incrementCounter = () => {
    increment({ increment: 1 });
  };

  return <button onClick={incrementCounter}>+1</button>;
}
```

### How it works

When the client calls the mutation function, it queues the mutation onto `optimisticUpdates`. The client stores the optimistic updates in a list, each containing the optimistic update with the `args`, as well as the `mutationId`.

The client immediately sends the message to the server via Web Socket. 

Removing a mutation from the local queue is a two-phase commit from the client's point of view.

- Phase 1: `MutationResponse{requestId, result, ts: 15}` arrives - client learns about that commit.
- Phase 2: `Transition{endVersion.ts: 16` arrives - the mutation can be dropped since the latest query already contains the mutation.

That's assuming that the `MutationResponse` was successful. 