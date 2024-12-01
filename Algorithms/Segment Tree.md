---
title: Segment Tree
topics:
  - Algorithm
  - Data Structure
description: A Segment Tree is a data structure that stores information about array intervals as a tree.
---

# Segment Tree

- Data structure that stores information about **array intervals** as a tree
- **Efficiently range queries answering** and **Quick modification**
- Very **flexible** data structure, so it can be easily generalized to larger dimensions
  - 2D Segment Tree can answer sum or minimum queries over some subrectangle of a given matrix in $O(\log^2 n)$ time
- Require only a linear amount of memory
  - $4n$ vertices for working on an array of size $n$.

## Simplest form of a Segment Tree

Given an array $a[0 ... n-1]$, the Segment Tree must be able to
- find the sum of elements between the indicies $l$ and $r$ **in $O(\log n)$ time**
- handle changing values of the elements in the array **in $O(\log n)$ time**
  - Simple array can update element in $O(1)$, but requires $O(n)$ to compute each sum query
  - Precomputed prefix sums can compute sum queries in $O(1)$, but requires $O(n)$ changes to the prefix sums

### Structure of the Segment Tree

1. Compute and store the sum of the elements of the whole array
2. Then split the array into two halves and compute the sum of each halve and store
3. Each of them splits in half, and so on until all segments reach size $1$