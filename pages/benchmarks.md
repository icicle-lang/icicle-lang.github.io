---
title: Benchmarks
---

Icicle is fast. By compiling all queries to efficient C code with
unboxed data structures, we can ensure that our queries run efficiently.

The benchmarks below are true, but also unfair to Icicle's detriment.

We thought of a set of queries which would be easy and possible to
write in SQL, then wrote simple versions in Icicle, and optimised the
daylights out of the SQL versions to get the results show below.

It took about half an hour to write the Icicle, and many iterations to
figure out the SQL.

The thing is though, when we just gave the definitions for the queries
we wanted to an engineer who hadn't been thinking about this problem
for the past few years, they took 2 weeks to write the SQL to perform
them, and they ran 1000x slower than our naïve Icicle version.

## The Numbers

We generated a small 5gb dataset of simulated supermarket basket data
including:

- 250M+ line items;
- 10M+ baskets; and
- 100k+ customers.

On this data, we generated 206 simple queries using the SKUs, costs,
and brands; and 21 more complex queries which required an extra group
by and join in all SQL engines.

We ran these queries on Icicle on a single machine, as well as clusters
of Redshift, Presto, and Spark; optimising all as best we could.


|                     | Icicle          | AWS Redshift | Apache Spark | Presto   |
|---------------------|-----------------|--------------|--------------|----------|
| *Machines*          | 1x c5.9xlarge   | 2x dc2.large | 2x m5.xlarge |          |
| *Cores*             | 16 used (of 18) | 4            | 8            |          |
| *Runtime: 206*      | 48.9s           | 8m 41s       | 24m          | -        |
| *Runtime: 206 + 21* | 81.8s           | Crashed      | 31m          | Crashed  |


Icicle was simpler to write and faster. While our first iterations in SQL ran
many times slower that our most optimised versions; we didn't touch our
Icicle queries after they were first written.
