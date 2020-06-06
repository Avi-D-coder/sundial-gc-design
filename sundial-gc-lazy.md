I have been working on a simpler design for Sundial's synchronization of type state, but I'm not 100% sure it's race free.
So any thoughts on this model are helpful.

This is the model for one iteration, corresponding to the GC thread freeing one region of type T, while ensuring no pointers.

There is one coordinator and any number of threads. There are two states dirty and clean.
The works send messages to the coordinator at the start and end of a arbitrary period.
The messages arrive instantly, but are not read instantly.
The start message contains the last epoch (a few bits not an Int) the coordinator has notified the thread of.
The ending messages contain the period's state.
A thread may have multiple overlapping periods, including differing states. 
The coordinator records the time it reads ending messages.
Any thread A may become dirty by communicating with thread B, while B is in a dirty period.

The coordinator must identify when no more dirty periods can occur.
Keep in mind the coordinator cannot stop the threads.

No more dirty periods will occur, after the coordinator observes all threads to have sent a clean ending message for their oldest outstanding period that was recorded to not coincide with any thread's dirty period.
E.g. If thread A has a period a1 outstanding (which will latter be recorded as dirty), and B starts period b1 which coincides with A's dirty period, then b1 is assumed to be dirty until B sends b1's clean ending message. However if any thread starts a period c1 which is known to start after a1 ends (via epoch), upon the coordinator reading b1's clean end, c1 is known to be clean.

Note the no new direct pointers into the condemned region invariant, has been replaced with: the worker threads will inform the GC of these pointers at a delayed point.
This removes the need for lazily requesting, caching and freeing Arenas of direct GCed field's types.
There are a number of other strange implications of this (mostly good).

If you squint hard enough, it looks a bit like the Chandy Lamport snapshot algorithm.
An open question is if new threads can register to send messages without hearing back from the coordinator.
