---
title: Solving Sudoku Puzzles
description: Getting back to Haskell, Tree traversal and [AlgorithmX](https://en.wikipedia.org/wiki/Knuth%27s_Algorithm_X) based solution
---

## Getting back to Haskell
Haskell ecosystem has seen quite a bit of change in the last few years. I thought it would be great to familiarize with the language again after a long break. 

### My dev environment
To get things off the ground, I was able to find a couple of very helpful blog posts that helped me set-up my dev environment. I am a *vi* user. And I was lucky to find this post on [setting NeoVIM up for Haskell development](https://mendo.zone/fun/neovim-setup-haskell/). This helps set up a very nifty dev environment with a lot of helpful features with the help of Intero.

I also came across [an opinionated take on Haskell](https://lexi-lambda.github.io/blog/2018/02/10/an-opinionated-guide-to-haskell-in-2018/). Its a pretty long post and it will take me quite some time to digest it, but certain useful things mentioned there made sense, particularly when you are trying to setup an environment. Specifically, it mentions, do not ever run `stack install`. As it explains, Stack works best when things are installed in local sandboxes. So, for a given project, it makes more sense to run `stack build --copy-compiler-tool hlint hoogle weeder`. This way, the binaries are installed in the local stack sandbox, and can be reused if you happen to use the same sandbox for another project.

## Sudoku
With the dev environment ready to go, its time to dive in. 

Sudoku is a simple puzzle with a 9x9 grid. The objective is to fill the grid with numbers 1 through 9, such that each row, column, and a 3x3 block has 1 to 9 exactly once. A much better description of it is available [in its wikipedia page](https://en.wikipedia.org/wiki/Sudoku). We would need some sample Sudoku puzzles to test out code. A ton of puzzles are available [here](https://github.com/simonmar/parconc-examples/blob/master/sudoku17.49151.txt).

### Depth First Search
Given the nature of the problem, a brute force, DFS based approach seems like an obvious choice to solve the problem. The idea is to start from the first open position, find out all available options for that position, and then consider each option, one-by-one. Each time, we move deeper into the tree. If we come across a situation where we still have open position, but no viable options, we backtrack. The following code snippet shows one such implementation.

We take a grid (a vector of size 81 in this example that we internally map to a 9x9 grid), and a starting position. We check if the position is filled, else, we find out the potential options by looking at the 3 constraints. We check which numbers need to be filled in the column, the row, and the block. The viable option for a given position is the intersection of these 3 sets. Once we have the possible candidates, we iterate through them one by one - exploring them till we either find a solution, or run out of options. The `concat $ map` line at the end does this.

```haskell
getPotentialCandidates :: V.Vector Char -> Int -> Set.Set Char
getPotentialCandidates v pos 
  | char /= '.' = Set.singleton char
  | otherwise = 
    let rowId = (pos-1) `div` 9
        colId = (pos-1) `mod` 9
        Just row = getRow (rowId+1) v
        Just col = getColumn (colId+1) v
        Just block = getBlock ((rowId `div` 3)*3 + (colId `div` 3) + 1) v 
        missingRowNums = getMissingNumbers row 
        missingColNums = getMissingNumbers col 
        missingBlkNums = getMissingNumbers block
     in missingRowNums `Set.intersection` missingColNums `Set.intersection` missingBlkNums
  where char = v V.! (pos - 1)
  
solve :: V.Vector Char -> Int -> [V.Vector Char]
solve v 81 = [v]
solve v pos 
  | char /= '.' = solve v (pos + 1)
  | otherwise = 
    let candidates = Set.toList $ getPotentialCandidates v pos
        recurSolve candidate = solve (V.update v (V.singleton (pos-1, candidate))) (pos+1)  
     in concat $ map recurSolve candidates 
  where char = v V.! (pos - 1)  
```

The code works! Well, sort of. It definitely tries to do what we intended. It's not the best approach though and the code just take a very long time (does not even finish) for even the first problem. Turns out, the very first position might have a large number of possibilities, and our choice of naively choosing the next position right after might also not be the best approach, as that would give us a dense tree, and we'd have a lot of branches to check.

The problem needs a re-look. A DFS based approach does seem reasonable. We might have to be a little clever pruning our tree though. Ultimately, that would decide how quickly we are able to get to the solution. The nature of the problem ensures that as we go down a path, the number of options reduces, so part of the work is done for us. We need to be clever about choosing which path to start with. One way to do this would be to start with a box that has the least possible number of options. This seems quite intuitive, as this is also how one generally starts solving a Sudoku puzzle. So we try that approach. The main recursion is now modified to look like the following:

```haskell
getPositionWithFewestOptions :: V.Vector Char -> (Int, Set.Set Char)
getPositionWithFewestOptions v = 
  let openPositions = filter (\p -> v V.! p == '.') [0 .. 80]
      posOptions = map (\p -> (p+1, getPotentialCandidates v (p+1))) openPositions
      sortedOptions = sortBy (\(_,o1) (_,o2) -> compare (Set.size o1) (Set.size o2)) posOptions
   in if (length openPositions > 0 ) then head sortedOptions else (0, Set.empty)
   
solve :: V.Vector Char -> [V.Vector Char]
solve v = 
  let (pos, candidatesSet) = getPositionWithFewestOptions v 
      candidates = Set.toList candidatesSet 
      recurSolve c = solve (V.update v (V.singleton (pos-1, c)))
  in if pos == 0 then
     --when pos == 0, we have the solution            
       [v] 
     else 
       concat $ map recurSolve candidates   
```

We now start with getting available options for all open positions. This is done in the `getPositionWithFewestOptions` function. We then sort them by size and pick the one with fewest possible options. We then try these options one by one. 

When we run it, we immediately see the difference. We are able to move through the first 10 problems in ~30s. The first problem takes ~2s to solve.

```
$ time stack exec sudoku-solver-exe -- sudoku17.1.txt 
Total puzzles read: 1
Puzzle
".......1."
"4........"
".2......."
"....5.4.7"
"..8...3.."
"..1.9...."
"3..4..2.."
".5.1....."
"...8.6..."

Tree traversal solution...
"693784512"
"487512936"
"125963874"
"932651487"
"568247391"
"741398625"
"319475268"
"856129743"
"274836159"


real    0m2.158s
user    0m2.559s
sys     0m0.259s

```
Lets see how we are doing memory-wise. We run the program with `stack exec sudoku-solver-exe -- sudoku17.1.txt +RTS -s`

```
  2,226,565,720 bytes allocated in the heap
      18,428,496 bytes copied during GC
         184,024 bytes maximum residency (4 sample(s))
         124,808 bytes maximum slop
               0 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0      2138 colls,  2138 par    0.452s   0.038s     0.0000s    0.0006s
  Gen  1         4 colls,     3 par    0.002s   0.000s     0.0001s    0.0001s

  Parallel GC work balance: 2.31% (serial 0%, perfect 100%)

  TASKS: 18 (1 bound, 17 peak workers (17 total), using -N8)

  SPARKS: 0(0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.000s  (  0.001s elapsed)
  MUT     time    2.034s  (  1.997s elapsed)
  GC      time    0.454s  (  0.038s elapsed)
  EXIT    time    0.000s  (  0.004s elapsed)
  Total   time    2.488s  (  2.041s elapsed)

  Alloc rate    1,094,929,649 bytes per MUT second

  Productivity  81.7% of total user, 97.9% of total elapsed

```
We see that GC takes up around 20% of the total execution time. Theres a decent amount of memory being allocated and then GC'ed. To be able to extract more details, we need to re-build with `stack build --profile`. This will build the profile version of all dependencies as well. The profiling build would be really slow.

We can run this using `$stack exec sudoku-solver-exe -- sudoku17.1.txt +RTS -p`. This would generate a file named `sudoku-solver-exe.prof`. This gives a break-down of how much time was spent in each function. That can help us guide potential optimizations. More details are available on [Haskell Profiling](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/profiling.html).

This is an interesting section of the profile - 
```
     getPositionWithFewestOptions.posOptions       Solver                            src/Solver.hs:62:7-82                                 1571      18769    0.3    0.9    99.0   99.4
       getPositionWithFewestOptions.posOptions.\    Solver                            src/Solver.hs:62:31-67                                1573     729771    0.1    0.2    98.7   98.5
        getPotentialCandidates                      Solver                            src/Solver.hs:(44,1)-(56,30)                          1575     729771    5.1    3.3    98.7   98.4
         getPotentialCandidates.(...)               Solver                            src/Solver.hs:50:9-40                                 1604     729771    0.0    0.0    39.1   36.8
          getColumn                                 Solver                            src/Solver.hs:(22,1)-(26,23)                          1606     729771    5.2    6.7    39.1   36.8
```

This tells us that a bulk of the time is spent in `getPotentialCandidates` function. This makes sense since most of the time since we use this to evaluate every branch at every level in the tree traversal. 

In the next post, we look at structuring this as an optimization problem and see how that further minimizes the number of options in each step, and drastically reduces the run time.
