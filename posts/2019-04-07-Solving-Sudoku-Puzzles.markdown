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

When we run it, we immediately see the difference. We are able to move through the first 10 problems in ~30s.
