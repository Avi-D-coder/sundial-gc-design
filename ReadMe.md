### The Sundial GC design

First, we will go over the core invariants and routines that enable type safe, pause-free GC.
Second, we will add more complex features: destructors, rooting, polymorphism, and mutation.
The GC uses a novel tri-color like design, and monomorphic types to discipline the heap graph and minimizes inter thread synchronization.

## The Worker thread View

-  Arenas are single threaded bump allocators for `Gc<T>`.
- `Gc<T>` may contain normal non GCed rust types like `Box<T>`,`&str`, `Box<Mutex<T>>` and `usize`.
- All GCed types must implement `Send + Sync + Trace`
- A worker thread holding `Arena<T>` blocks garbage collection for the transitive set of types contained by `T` (transitive children).
- A worker may hold more than one `Arena<T>` at once.
- A worker will not block GC by holding different `Arena<T>`s for overlapping indefinite periods of time.
- GCed references are only usable for the lifetime of an arena (this is enforced by the type `Gc<'a, T>`).
- An `&'n Arena<T>` can safely turn a `Gc<'o, T>` into a `Gc<'n, T>`,  
with the method `fn trace_lifetime<'o, 'n>(&'n self, t: Gc<'o, T>) -> Gc<'n, T>`  
(possible for GCed direct children as well).
- By routinely acquiring arenas and extending lifetimes and then dropping the old arena, a worker may continuously use GC without blocking GC or incurring a pause.
- The largest pause will be mallocing a new arena if your working set grows.

An `Arena<T>` will observe each of it's GCed fields `C` (direct children) to be in one of two phases:

1. Fast Phase: The GC thread was not attempting to free any `Arena<C>` when this `Arena<T>` was obtained.
2. Cooperative Phase: When a dropped `Arena<C>` is condemned to be freed.

Three simplified invariants are enforced:

1. All GCed types found under `Gc<T>` are known at compile time.
2. When `C` is in the cooperative phase, no new direct references into the white set may be written.
3. Having a `Gc<C>` requires a live `Arena<T>` where `T` directly or transitively contains `Gc<C>`

The first invariant is met by deriving implementations of `Trace`.
Additionally, we must ensure `Trace` is not object safe,
so polymorphic usage (`Gc<dyn Trace>`) produces a compile error.

The Trace trait looks a bit like this:

```rust
trait Trace {
    fn shallow_trace(t: &Self, what_to_trace);
    const FIELD_COUNT: u8;
    const TYPE_INFO: Option<GcTypeInfo>;
    fn child_gc_info() -> HashSet<GcTypeInfo>;
    fn transitive_gc_info() -> HashSet<GcTypeInfo>;
}
```
It has a blanket implementation for any shallowly immutable type that does not include a `Gc` (using auto traits).

The second invariant is satisfied by `trace_lifetime` calling `shallow_trace`.
`shallow_trace` checks the `Gc` pointers of cooperative fields (`child: Gc<C>`).
If a pointer references an object in a condemned region (the white set) and the object has not been moved,
the white object is evacuated into a live arena and a forwarding pointer is installed in the old arena.
Copying is done into a thread specific `Arena<C>`, which may have to be obtained and cached.
When an object has a destructor, it must not be duplicated, hence, tracing and forwarding requires atomic synchronization.
An acquire-load is used to check if the value has been moved and a compare and swap is required to check that our move was not preempted.
If we copied a unique object, but another thread moved it first, we must roll back our allocation (just a pointer increment).

There are a number of optimizations for tracing and forwarding
ranging from marking info in unused alignment bits to probabilistic relaxed forwarding, or even no forwarding at all on worker threads.
In general, avoiding lookups in condemned Arena's forwarding maps is the goal.
Type layout and requirements determine the applicable optimizations for each type at compile time.

Phase info for all types `C` and condemned region pointers was left for the worker thread by the GC thread when the `Arena<T>` was obtained.
Acquiring an arena is done via a compare and swap and a few relaxed loads on a cache line shared with the GC thread.
Dropping an arena only incurs be a release store on the shared cache line.
Arena acquisition may incur system allocation when the GC thread does not provide a free Arena.

## The GC thread's View

The design I will describe has only one GC thread.
There are blindingly obvious opportunities for parallelization of GC work over multiple GC threads.
However, exploring them is beyond the scope of this simplified document.

The GC thread has three main duties.

1. Bookkeeping and Coordination: keep track of generations and tri-color arena sets, coordinate and track worker thread's type phases, and arena bounds.
2. Scan gray arenas for references into white arenas and copy them into older generation Arenas.
3. Running destructors (`Drop`). We will address rooting, weak pointers, and finalizers later. This almost certainly belongs on a dedicated thread.

### How does an Arena get freed?

When a worker thread drops an arena it notifies the GC thread with a release store.
The message contains the address of the arena header and the amount allocated.
The allocated portion will not be written to by worker threads until the GC thread tells workers to enter a cooperative phase.
The GC allocates arenas of associated types and ages in the same region of the heap.
This enables the GC thread to condemn multiple arenas by sending a single pointer.
To condemn a region the GC thread must own all arenas in the region.

The GC thread attempts to find a region or, failing that, a set of Arenas to condemn.
Once a white set has been selected, the gray set is known to be all `Arenas<T>` that may hold direct references to condemned types.
Remember `GcTypeInfo` information was generated at compile time
and subsequently loaded into the GC when the first thread registered to acquire an `Arena<T>`
The gray set excludes any arenas of generation `N + 1`, since we uphold a shallow generational invariant.
The objects in the gray set fall into two categories: objects allocated into a worker's live arena and objects owned by the GC.

All full dropped arenas and arena segments fall into the former category.
There are a myriad of efficient parallelization and acceleration schemes for shallow tracing of owned objects.
The most exciting possibility is SIMD or even GPU acceleration for massive heaps.
My initial implementation will stick to a boring algorithm, move evacuees, perform a relaxed write over the condemned pointer and add a forwarding pointer a thread local mutable forwarding map.
Upon completion of an arena, a pointer to the now frozen forwarding map is written into the condemned arena's header
(required atomic write ordering depends on the implementation of rooting).
The write-fence denotes the conceptual transfer of the scanned arena from the gray to a shallow black set.

Once all GC owned objects are marked black, GC must induce the workers to stop producing new members of the gray set.
The GC thread sends messages to each of the worker's arena allocators, to enter the type cooperative phase.
The message includes a small bit map marking the types of possibility condemned children, and pointers to white condemned regions.
While I have not yet settled on a precise compressed encoding for messages sent on these shared cache lines,
at least one CAS is required to ensure synchronization.

#### Acquiring an Arena

When a worker thread requires a new `Arena<T>`, it performs a few atomics loads and a CAS to acquire `T`'s phase and the white regions.
The worker may also acquire an empty Arena left by the GC thread or the remaining segment of the worker's previously freed Arena.
The CAS acquire read ensures the worker will see most recent message and
ensures GC writes prior to the GC's message CAS are will be observed by the worker/
The CAS release store informs the GC that the new arena has entered the cooperative phase.
Allocations made in cooperative arena uphold the invariant that no new direct pointer into the white set may be written.
When a cooperative Arena or segment is dropped by the worker, this invariant enables it to go directly into the shallow black set.
As workers drop their fast phase `Arena<T>`s into the owned gray set, the GC will shallow trace them until the gray set (comprised of worker's fast Arenas and GC owned Arenas/Segments) is totally empty.

#### Freeing a Condemned Arena

At this point, no new transitive references into the condemned region can be written, but a workers may still have white pointer in their stack.
To clear these pointers, the GC thread will send all arena allocators of types containing **transitive** or direct pointers to a condemned type a message.
The message is just a conformation bit.
Both direct and transitive parents of the condemned have to drop their pre-confirmation arena segment.
Direct parents are still in the cooperative phase and therefore must continue shallow tracing just as before.
Once all pre-conformation arenas have been dropped, the GC thread sends one last set of messages ending these cooperative phases (The message may start a new cooperative phase on a different white set).
When the GC observes all the cooperative phase arenas/arenas segments dropped it may run destructors and recycle or free the region.

### Rooting

Rooting is easy.
Each Arena header includes a pointer to the head and last node of a `RootList` (info could also be part of the forwarding map, list does not necessarily mean linked list).
Nodes in the root list include the index of the rooted object in the arena and a pointer to the owned `Root<T>`.
`Root<T>`s may be turned into a `Gc<T>` by an `Arena<T>` with `fn trace_root<'o, 'n>(&self, &Root) -> Gc<'n, T>` (likely also by `Arena<P>` where `P` has a field `Gc<T>`).

Before just before the Arena is condemned the rooted `Gc<T>`s are evacuated and the `Root<T>`s are updated by the GC thread.

A similar mechanism could be generalized to support finalizers and arbitrary callbacks on evacuation.

### The Polymorphic Lazy Elephant in the Type

Polymorphism and Laziness don't fit in our disciplined graph.
A polymorphic `Arena<dyn Trace>` breaks the first invariant.
An arena containing a polymorphic type is a direct reference to all types.
This would kill performance.
Unfortunately, banning polymorphism rules out tracing through thunks.
This is bad for the performance of laziness, but drastically simplifies it's implementation.
Rust closures are opaque, so tracing through laziness would already require procedural macros in expression position.
Unlike `Gc<'a, T>`, `Root<T>` does not have to be traced and can be captured by normal Rust closures, making laziness much simpler.

So how do we use a polymorphic type in a GCed type we `Box` it onto the Rust heap and stick the `Box` in the `Gc` (E.g. `Gc<Box<dyn Trait>>`).
`Gc<'r, Box<dyn Fn() -> usize>>` is type checks, if your closures needs the GC just capture a Root.

### Mutation

The solution to inline mutation is much simpler,
as it only involves ensuring an object is not duplicated or copied out from under you while in a cooperative phase.
With that being said, implementing inline mutation is a secondary goal.
Until proper mutation is implemented, mutation inside `Box`, and `Arc` will have to do.


### Okay it's Pauseless, but Throughput will Suffer

At this point you maybe saying:

- Tracing from new arenas instead of roots will require traversing many more objects.
- Memory usage is going to blowup, as a result of falsely promoting a portion of objects to the next generation.

While this GC will scan more objects,
I'm not convinced it will take longer to complete a than traditional evacuation from the roots.
Traditional tracing involves copious unpredictable branches, and random memory loads that cannot proceed until the prior load is resolved.
If the trace phase is multi threaded to reduce pause time, synchronization drives efficiency down even further.

The bet I'm making is once the traversal is narrowed with type info,
the remaining increase bytes read is less than the cost of chasing pointers.
The type graph also enables planned parallelization that requires minimal synchronization and prevents false sharing.
Once deferred evacuation and SIMD and possible GPU tracing is thrown in, I would be surprised if tracing and evacuation is not a good deal faster than root first approaches.

Memory retention is likely to be an issue.
The solution is better heuristics and smarter condemned region selection.
While the process I described is focused on a single white set of a few regions,
even a single threaded GC will interleave the steps of collecting various white regions and sets of condemned types.
Traversals will gather partial evacuations sets of uncondemned arenas to aid in region selection and get a jump on the collection.
The GC could even keep per type and thread retention statistics in order to determine if an arena should be evacuated into an older or same generation arena.

False retention can also be prevented with brute force.
The larger the percentage of the arenas generation `0..=N` condemned the lower the risk of false retention to the next generation.


##### Avi Dessauer
