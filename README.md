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
And (1) read user input, (2) updaet state and perform I/O, and (3) print them to UI.

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
have dependencies and transformations, "dataflow programming" can be a very good choice.
You just push some parameters to a configured pipeline, and gonna get result eventually and
integrate them into app state.

I do not support applying "dataflow programming" style to whole app. Basically, an app should
remain as REPL with "single unified state". Because, "dataflow programming" works well only
if each pipelines doesn't need to be fully consistent. And in almost all cases, apps require 
"consistency" in very wide level — globally.
REPL with "single unified state" is the best and easiest way to archive that so far in my opinion.
Therefore, "dataflow programming" can be an optional choice to simplify complex I/O.



Avoid Unnecessary Operations
-------------------------------------
Naive REPL implementation can be very expensive.
Operations such as copying/scanning/rendering are not cheap, and you have to optimize your app
to avoid such operations wherever possible.  

- Some copying can be avoided by employing copy-on-write strategy.
- Whole collection copying can be avoided by emplying multi-level data structures like B-Tree or
  collection of of smaller collections.
- Some rendering can be avoided by triggering loop only on v-sync.
- Some rendering can be avoided by performing equality checks and skipping for same data.
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



Optimization — Loop on V-Sync
----------------------------------------
Loop SHOULD NOT be directly triggered by user's input event. Detected input should be integrated into
app state, and loop should be driven by another mechanism.











Credit & License
---------------------
Copyright (c) 2019 Eonil, Hoon H..
All rights reserved.
Using of this document is allowed under[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) license.
Using of any code is allowed under "MIT License".

