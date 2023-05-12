---
title: The Icicle Streaming Query Language Guide
---

This User Guide is a Work in Progress.

Icicle is a strongly typed pure query language that compiles to C. The Icicle runtime
provides both _batch_ and _streaming_ query execution exploiting the compiler code,
and supports different file formats and querying aspects like snapshots or requesting
particular entities at certain times.

In this guide, you can learn:

- How to install Icicle and set up a basic test environment;
- How to use the Icicle repl to quickly figure out queries;
- Event sourcing concepts and practice;
- How the type system and modern programming techniques can help you model a domain;
- How the Icicle module system enables collaboration; and
- When and why to use Icicle vs other technologies.


Using Icicle can be a bit different to using SQL and other tools for one's data
engineering workflows. Icicle covers off a lot of the boilerplate that is required
for writing SQL, and, by starting with a different data model: streams of events vs
multisets – Icicle makes a huge variety of queries far easier to write and faster
to execute.
