# Core Java — screenshot list

Part 3 examples. One topic per block. The line at the bottom is what to say.

## Fail-fast vs fail-safe iterators

A fail-fast iterator walks the real list. If the list changes during the walk, the walk stops and throws `ConcurrentModificationException`. `ArrayList` does this.

A fail-safe iterator walks a copy made at the start. A change to the real list does not stop the walk and does not throw. The walk does not see that change. `CopyOnWriteArrayList` does this.

The same-thread version is enough to see it. The `for` loop is the walk. `remove` on the list itself is the change.

Fail-fast. This throws on `"A"`.

```java
List<String> bookings = new ArrayList<>();
bookings.add("A");
bookings.add("B");
for (String booking : bookings) {
    bookings.remove(booking);
}
```

Fail-safe. This finishes. The walk still had `"A"` and `"B"` because it was walking the copy.

```java
List<String> bookings = new CopyOnWriteArrayList<>();
bookings.add("A");
bookings.add("B");
for (String booking : bookings) {
    bookings.remove(booking);
}
```

Say this: fail-fast throws if the list changes during the walk. Fail-safe walks a copy and does not throw.

## ConcurrentHashMap

A normal `HashMap` is one shared structure. Two threads updating it at the same time can lose an entry or loop forever.

`ConcurrentHashMap` does not lock the whole map. An update locks only the bucket for that key. Two threads writing two different keys both go through. Two threads writing the same bucket take turns. A read does not take that lock.

This is not a database row. The seat lock in Event is a Postgres row. This lock is one slot inside the map, in memory.

Say this: `ConcurrentHashMap` locks the bucket for the key being changed, not the whole map.

## Thread pool queue is full

Every worker thread is busy, and the queue of waiting tasks is full. The next task is not queued. The default policy throws `RejectedExecutionException` back to the caller. The task does not run.

Say this: a full pool and a full queue reject the new task.

## Mutable HashMap key

A booking is the key. You put it in the map, then change the field that `hashCode` uses. The entry stays in the bucket for the old hash. Lookup uses the new hash and searches a different bucket. `get` returns null. The entry is still inside the map.

Say this: changing a key field after `put` makes `get` miss it. The entry sits in the old bucket.

## Optional: orElse vs orElseGet

The `Optional` already holds a booking. The fallback calls Auth.

`orElse(loadUser())` calls Auth anyway. Java runs `loadUser()` before `orElse` starts, then discards that result because the booking was present.

`orElseGet(() -> loadUser())` does not call Auth. The lambda runs only when the `Optional` is empty.

Say this: `orElse` always builds the fallback. `orElseGet` builds it only when the value is missing.
