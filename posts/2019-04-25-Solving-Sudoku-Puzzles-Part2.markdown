---
title: Solving Sudoku Puzzles - Part 2
description: Getting back to Haskell, Tree traversal and [AlgorithmX](https://en.wikipedia.org/wiki/Knuth%27s_Algorithm_X) based solution
---

The previous post had a basic tree-based solver to solve Sudoku puzzles. We now look at a solver that frames this as an [Exact Cover problem](https://en.wikipedia.org/wiki/Exact_cover). 

## Exact Cover
Exact cover is a constraint satisfaction problem. The wikipedia link above has a more comprehensive description of the problem. For a given set X, and a collection of subsets of X, say S, we try to find sets that together can form X, but in a way that no element of X is repeated in the sets we choose. That is the *exact* in the _Exact Cover_.

## Sudoku as Exact Cover 
[The exact cover description](https://en.wikipedia.org/wiki/Exact_cover#Sudoku) again describes in detail how we can frame Sudoku as an exact cover problem. We have 4 kinds of constraints

### Row-column constraint
We know that each cell (or an intersection of row and column) can have one of 1 through 9 values. This constraints in represented as R1C1 = {R1C1#1, R1C1#2 .. R1C1#9}. The set contains all the possibilities that can satisfy this constraint, and exactly one needs to be chosen for the solution to be viable.

### Row constraints
A given row needs to have numbers 1 through 9. This is expressed as a set of 9 constraints, one for each number. So for example, the first constraint would be:
R1#1 = {R1C1#1, R1C2#1, .. R1C9#1}. 
Again, there are 9 ways this constraint can be satisfied, and exactly one option needs to be chosen for the solution to exist.

### Column and Box constraints
Each column and 3x3 box also needs to satisfy the similar kinds of constraints as the Row constraint described above. 

All these result in 4x(9x9) = 324 constraints. Each of the options that meet the constraints together form a total of 9x9x9 = 729 total options. These are expressed as a 729x324 sized matrix. The full matrix is [shown here](https://www.stolaf.edu//people/hansonr/sudoku/exactcovermatrix.htm).

As we can see, there are a lot of 0s in this matrix. So we can express this matrix using a sparse representation. We only keep a track of the 1s, and represent it as (rowId, colId, 1). 

So we have the problem setup and ready. We can solve this using _Algorithm X_. To be able to do that, we would need to be careful to keep a set of constraints, alongside the matrix, to make sure we are able to keep a track of constraints that have not been met. 

The algorithm starts with choosing a constraint that has minimum possible options. One of the options is randomly chosen and we explore if we can get to a solution with it, or we backtrack. At every step, we eliminate all the other potential values that also meet the constraint, since we need *exact* cover. [The code is available here](https://github.com/shubhamchopra/sudoku-solver/blob/master/src/AlgoXSolver.hs).

Lets try running this on 10 puzzles - 
```
$ time stack exec sudoku-solver-exe -- sudoku17.10.txt
...
real    0m0.435s
user    0m0.461s
sys     0m0.111s
```

This was fast. The tree-traversal took ~15s to solve these 10 puzzles. This method did that in less than half a second. 

## Tree traversal vs Exact cover
So what did we miss in the previous implementation? The main reason the exact cover version works this fast, is almost complete pruning. Recall that we only pruned the set of possibilities with a very simple approach in the tree traversal. We did not really prune all non-viable options. The exact cover implementation is a lot more successful in doing that, and thereby significantly reducing the amount of backtracking needed to solve a puzzle.
