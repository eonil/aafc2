AAFC2 (App Architecture for Cocoa 2)
=============================
Eonil, 2019.

- This document describes generic points of how to write an app.
- This document basically targets Swift and Apple AppKit/UIKit platform.
- Though this is optimized for AppKit/UIKit apps, major ideas can be applied 
  to another platforms.


Summary
------------
- Basic is REPL. With "single unified state".
- Naive REPL can be very expensive. Optimize them as much as possible.
- AppKit/UIKit are eventual-consistency system. We need to pass-by-copy the state to overcome latency.
- Use Swift and take advantage of copy-on-write to overcome cost of copying.
- Do not trigger iteration by user input event. Instead, just integrate user input into app state. 
- Limit loop frequency by triggering iteration by v-sync only if needed.
- V-sync provides timestamp natively. Take advantage of it. 



REPL
-------
Basically, an AppKit/UIKit app is a GUI app and GUI means interactivity.
REPL is the simplest way to write Interactive apps. There are many other approaches,
but other approaches are usually just make things more complex with no significant benefit.

- Configure whole app as a single read-eval-print loop.

You keep a massive pure value-semantic state.
And (1) read input, (2) updaet state and perform I/O, and (3) print them.

For better description in GUI apps, I use terms like "scan"/"render" instead of "read"/"print".


Whole App Single Unified State
---------------------------------------------
One app should keep single value as its state. This is for global consistency, and especially
good ifb your app changes rapidly.
There has been many trials to divide app state into multiple pieces and control them individually.
But it's harder to keep divided states consistent. As large portion of bugs are coming from
"inconsistent state", avoiding such situation can be huge benefit.

Keep whole app state as single value and always pass them "by copy" between app components.



Bi-directional Message based I/O
----------------------------------------
Push is better than polling. Always design your messaging as bi-directional, and do not make
or assum request/response relation. Any message can arrive at any time, and your app should be
ready to process them properly. Design your messages independent and idempotent as much as
possible.



Optional Dataflow (a.k.a. Reactive) for I/O
----------------------------------------------------
Nowadays "reactive programming style" is in trend. It's actually just another name for 
"dataflow programming" which has been for decades. Please don't be confused.
This trending "reactive" style is nothing related to "FRP" at all.

Dataflow programming is very good to build specific kind of I/O pipeline. If your I/O operations
require dependencies and transformations, "dataflow programming" can be a very good option
to simplify such I/O. You just push some values into a configured pipes, and take and integrate
productions into app state when it gets ready.

I do not support applying "dataflow programming" style to whole app. Basically, an app should
remain as REPL with "single unified state". Because, "dataflow programming" works well only
if productions of each pipe don't need to be consistent. As "dataflow programming" aims to 
provide convenience in asynchronous operations, they need to be isolated, therefore it's very
hard to keep consistency between isolated pipes with another part of app state.

Therefore, at the end of the pipes, productions need to be integrated consistently into app state.
As "single unified state" keeps whole app state, it's easy to integrate them consistently.
Just run REPL, and integrate dataflow productions on each iterations.

If used, dataflow module should be exist as a isolated module and communicate with REPL 
module only exchanging messges.



Avoid Unnecessary Operations
-------------------------------------
Naive REPL implementation can be very expensive.
Operations such as copying/scanning/rendering are not cheap, and you have to optimize your app
to avoid such operations wherever possible.  

- Some copying can be avoided by employing copy-on-write strategy.
- Whole collection copying can be avoided by employing multi-level, page-based data structures 
  like B-Tree, hash-table of small arrays or proper persistent-data-structures.
- Some rendering can be avoided by triggering loop only on v-sync.
- Some rendering can be avoided by skipping operations for same data.
- For this, operation must be referentially transparent.
- For this, equality checks are required.
- Equality checks can be dirty cheap as O(1) by employing extra state metadata such as 
  timestamping or version numbers.
- Pause loop whenever possible. For example, you can run loop momentarily only for incoming 
  user input.



Optimization — Swift & Copy-on-Write
--------------------------------------------------
Always use Swift. Because Swift has very strong support for value-semantic data structures.
Swift data types are *copy-on-write* by default. Therefore, Swift value types are natively 
persistent data types, copies are cheap, and supports "single unified state" approach very well.

Although you also can implement "single unified state" approach with other languages like
Objective-C, it should be far more painful experience than Swift. 



Optimization — Limit Iteration Frequency
------------------------------------------------
Usually, you don't need REPL iteration more frequent than 60Hz. 30Hz or 20Hz iteration 
can be okay up to the purpose of the app. Therefore, you can employ a regularly repeating 
timer to perform iteration. Data production and end-user control messages can arrive between
iteration timings. Just queue and process them on iteration.

Considering energy efficiency, it's better to batch processings at once. Then the best timer
to trigger REPL iteration is v-sync. As system is supposed to perform something on v-sync,
performing extra jobs at this time can save more energy. Just wait for v-sync and trigger
REPL iteration on non-main thread. Iteration only on v-sync also natively fits to UI rendering,
and UI rendering must be performed only on main thread for AppKit/UIKit.

Here can be another problem. If incoming messages are too many, memory space can be 
insufficient. In my opinion, if there're too many messages, it doesn't matter because it there
are too many messages, it'd hard to process them too. At the first place, app should prevent
production of huge amount of messages. If it's process-able, it's storable I think. 
Also as a final protection, limit the message queue size, and reset the app if it overflows. 
If a message queue always overflows, it's a bug to fix in overall system-wide.

Limiting iteration frequency also can suppressing asynchronous infinite loop.
Asynchronous infinite loop happend due to a bug in algorithms, therefore a mechanism 
cannot them, but without limited iteration, such infinite loop can increase processings
exponentially. If iterations are limited, such infinite loop will be limited in the iteration 
frequency.



Optimization — Timestamping
------------------------------------
Limiting iteration frequency also provides another benefit of "timestamping". 
For quick equality comparison, we need unique identifier for data versions.
So far the best solution for this was monotomically increasing number for each update.
That's usually enough, but key-space consumption rate is very hard to predict. 
It's very hard to consume all of 64-bit integer key space, but unpredictability itself is a big risk. 

Anyway, if we limit iteration frequency, it means we can increase monotonic timestamp
in a constant rate, therefore we can secure enough running time for most apps. With a simple
calculation, an app can run for more than 4 billion years if it iterates in 60Hz, and I believe 
that's enough for most apps nowadays.

We can employ this timestamp to provide state version. App can store timestamp of last 
update in any part of its state, and rendering can skip them partially in O(1) performance.



Debugging — Trace Message Passages 
------------------------------------------------
Always trace and store where and how the messages are produced and travelled through.
It's not needed to run apps, but very helpful for debugging.
This is especially important if processing of a message produces subsequent messages.
Such message production can lead to infinite asynchronous loop, and such infinite loop
is very hard to detect and fix. If we can trace message production chain, it becomes
easier to detect and fix such loops.

Unfortunately, there's no magical way to trace message passsages. So far the best way
would be packing all messages into a wrapper that must be chained to spawn a new 
instance. Making a new instance of such package should be limited only for very origin
sites such as end-user event or remote notifications. For the data I/O productions, 
there must be a message initiated data I/O, and the message should be chained from.   


Further Readings
----------------
I strongly recommend to read these articles.
- [POSTMODERN IMMUTABLE DATA STRUCTURES](https://sinusoid.es/talks/immer-cppcon17/#/)



Credit & License
---------------------
Copyright (c) 2019 Eonil, Hoon H..
All rights reserved.
Using of this document is allowed under[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) license.
Using of any code is allowed under "MIT License".

