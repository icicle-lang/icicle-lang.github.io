---
title: Benchmarks
---

Icicle is fast. By compiling all queries to efficient C code, we can
ensure that our queries perform well.


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

The results were surprising:

|                     | Icicle          | AWS Redshift | Apache Spark | Presto   |
|---------------------|-----------------|--------------|--------------|----------|
| *Machines*          | 1x c5.9xlarge   | 2x dc2.large | 2x m5.xlarge |          |
| *Cores*             | 16 used (of 18) | 4            | 8            |          |
| *Runtime: 206*      | 48.9s           | 8m 41s       | 24m          | -        |
| *Runtime: 206 + 21* | 81.8s           | Crashed      | 31m          | Crashed  |


Icicle was simpler to write and faster. While our first iterations in SQL ran
more than 10x slower than our optimised versions; we didn't touch our
Icicle queries after they were first written.

