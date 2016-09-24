---
title: Haskell Profiling and Performance Optimization - Funk SVD
description: Starting with a simple Funk SVD implementation, using some GHC profiling tools for performance optimization.
---

## Recommendation Systems: A brief and definitely not comprehensive introduction
Recommendation systems today are ubiquitous. Almost all websites use them for everything from recommending content to read to movies you might like to watch to products you might want to purchase. When there are plenty of options and resources like time or money or both are scarce, these act like your friends to help guide you through a maze of options and come out reasonably satisfied from the experience. If you did not like the experience or even if you did like it, you will rate whatever it is you consumed and the
system will *learn* from it and try to be better at it the next time. In essence, they solve a quintessentially _first world_ problem of plenty.

It is this _learning_ that is an interesting problem. Large amounts of data is collected on what different users consumed, and how they rated it, and is used to come up with mathematical models that can predict based on the past experiences what the users might like. At the core, therefore, is a user-item matrix that contains ratings of various items by various users. This matrix, almost always, is extremely sparse. I bet you don't know anyone who has seen all the movies on Netflix, for example. (If you do, that person probably needs a recommendation system to tell them to go outside once in a while, the graphics are really amazing.)
Most people would have only consumed a very small subset of the entire universe of product a business deals in. 

I will not pretend to be an expert in recommendation systems. There are several online courses that go in-depth into various techniques that can be used for recommendations. I personally completed and enjoyed the "Introduction to Recommender Systems" course on [Coursera](https://www.coursera.org), but I see that it has been made into a specialization [now](https://www.coursera.org/specializations/recommender-systems). 

While recommender systems have existed for some time now, the problem itself got it's biggest marketing boost courtesy the [*Netflix Challenge*](http://www.netflixprize.com/). Netflix gave a sizeable amount of data to train the models and that attracted a lot of researchers. Not to mention the million dollars. One such researcher goes by a pseudonym "Simon Funk" and came up with a technique that is now called [_Funk SVD_](http://sifter.org/~simon/journal/20061211.html). 

Very briefly, it is a matrix factorization technique (Singular Value Decomposition) to decompose the ratings matrix. As a result of this factorization, we end up with user vectors and item vectors, each describing some salient preferences of users and salient features of items. Once we get this, we can now use dot product of a user vector with an item vector to get an estimate of how a user would rate the item. 

Since this post is about Haskell and profiling, lets now jump right into it. The rest of this post is arranged as follows:
We start with a preliminary implementation of Funk SVD that gives us the required output. We then profile it using tools available in GHC. GHC has profilers that can help us understand how and where memory is being used. Finally, we try to modify code with an attempt of minimizing memory usage.

## [Funk SVD - Initial Version](https://github.com/shubhamchopra/RecommendationSystem/InitialVersion)
We will go over a brief code overview to understand the flow. The src/Main.hs file contains the main function, no surprises there. I read a gzip compressed rating file, that contains ratings in the form "rating|user|item" where the rating is a double, user and items are both integers. We read this file each entry is used to create a _Rating_ object. We also create the _Config_ object that fills up some parameters used in estimation. 

The meat of the code is in src/FunkSVD/SerialEstimator.hs. The following three functions are central to the logic:

```haskell
runIteration :: Config 
  -> (M.Map User Double, M.Map Item Double) 
  -> DV.Vector Rating 
  -> (M.Map User Double, M.Map Item Double)
runIteration config = DV.foldl' f
  where
    f (uV, iV) (Rating r u i) = (uV', iV')
      where
        dStart = startingEstimate config
        lRate = learningRate config
        reg = regFactor config
        ua  = M.findWithDefault dStart u uV
        ia  = M.findWithDefault dStart i iV
        eps = r - ua * ia
        dua = lRate * (eps * ia - reg * ua)
        dia = lRate * (eps * ua - reg * ia)
        uV' = M.insert u (ua + dua) uV
        iV' = M.insert i (ia + dia) iV
```
This function runs a single iteration of the estimation. It takes a vector of user and item dimension, along with the vector of ratings, and returns an updated vector of user and item dimensions. The update is as explained in the Funk SVD algorithm.

```haskell
runTillConvergenceIO :: Config -> FunkSVD -> IO FunkSVD
runTillConvergenceIO config funkSVD =
  do
  stateMaps <- newIORef (M.empty, M.empty)
  forM_ [1 .. (numIterations config)] $ \_ -> do
    maps <- readIORef stateMaps
    shuffledResiduals <- R.shuffleVectorIO $ residuals funkSVD
    writeIORef stateMaps $ runIteration config maps shuffledResiduals
  (dUv, dIv) <- readIORef stateMaps
  let residuals' = updateResiduals (residuals funkSVD) dUv dIv
      uV = userVectors funkSVD
      iV = itemVectors funkSVD
  return funkSVD{residuals = residuals'
                 , userVectors = appendVectors uV dUv
                 , itemVectors = appendVectors iV dIv}
```
This function takes the input, and runs a number of iterations to estimate a single user and item dimension. Funk SVD algorithm recommends that we randomize the ratings vector between the estimation iterations for better generalization. 

```haskell
runFullEstimationIO :: DV.Vector Rating -> Config -> IO FunkSVD
runFullEstimationIO ratings config =
  do
  (trainingSet, testSet) <- R.randomSplitIO (4 * DV.length ratings `div` 5) ratings
  let funkSVDInput = runReader (generateFunkSVDInput trainingSet) config
  funkSVDRef <- newIORef funkSVDInput
  forM_ [1 .. (numDimensions config)] $ \dim -> do
    putStrLn $ "Working on dimension " ++ show dim
    funkSVD <- readIORef funkSVDRef
    funkSVD' <- runTillConvergenceIO config funkSVD
    writeIORef funkSVDRef funkSVD'
    putStrLn $ "Average residuals = " ++ (show . globalMean abs) (residuals funkSVD')
    putStrLn $ "Residuals RMSE = " ++ show (sqrt (globalMean (**2) (residuals funkSVD')))
    putStrLn $ "Training RMSE = " ++ show (getRMSE trainingSet funkSVD')
    putStrLn $ "Test RMSE = " ++ show (getRMSE testSet funkSVD')
  readIORef funkSVDRef
```
This is the function that runs the top-level loop that tries to estimate all the dimensions. It starts by randomly keeping aside a 20% dataset to act as an out-of-sample test set. The remaining data is used for training. It also prints the mean and RMSE error along the way.

This should be enough for now, we will go into other functions as and when needed. Let's jump right into profiling now.

## Memory profiling in GHC
GHC comes with a decent set of tools to get an estimate of memory usage. All the techniques we are using here are described in [Real World Haskell](http://book.realworldhaskell.org/read/profiling-and-optimization.html). We start with the very basic:
*+RTS -s*. This is run using the command `stack exec -- RecommendationSystemInitialVersionProfiling ./testRatings.gz +RTS -s`. This prints out GC and memory stats at the end of the run. I have used a significantly small input, about 10k ratings, to keep the runtimes small for faster iterations. Let's take our initial version for a spin and see how it performs.
```
  13,760,123,472 bytes allocated in the heap
  16,451,043,328 bytes copied during GC
     386,253,360 bytes maximum residency (126 sample(s))
      17,518,264 bytes maximum slop
             685 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0     26684 colls,     0 par    6.292s   6.316s     0.0002s    0.0010s
  Gen  1       126 colls,     0 par    8.131s   8.931s     0.0709s    0.3639s

  TASKS: 4 (1 bound, 3 peak workers (3 total), using -N1)

  SPARKS: 0 (0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.000s  (  0.001s elapsed)
  MUT     time    6.387s  (  8.333s elapsed)
  GC      time   12.505s  ( 13.328s elapsed)
  RP      time    0.000s  (  0.000s elapsed)
  PROF    time    1.918s  (  1.919s elapsed)
  EXIT    time    0.000s  (  0.001s elapsed)
  Total   time   20.843s  ( 21.663s elapsed)

  Alloc rate    2,154,403,840 bytes per MUT second

  Productivity  30.8% of total user, 29.6% of total elapsed

```
The _bytes maximum residency_ is the maximum number of bytes our process was using. This number is only an estimate as it is only measured during a major GC when older generations (Gen 1 in this case) is also garbage collected. The other number, _685 MB total memory in use_ on the other hand tells us whats the maximum amount of memory this process had taken from the OS. This is the number to focus on, and is pretty huge for the size of the input we have here. It also gives us the break-down of what time was spent on
the process (MUT) vs GC. Finally, productivity tells us that only 31% of the time was spent doing our number crunching. The rest was spent garbage collecting. This code is screaming for some optimization.

So let's try to find out the memory hogs in our code. We do this by building the binaries with profiling enabled. We then execute the binaries using a command like ```stack exec -- RecommendationSystemInitialVersionProfiling ./testRatings.gz +RTS -s -p -hc```. This creates a profiling output file `RecommendationSystemInitialVersionProfiling.hp`. This information can be converted to a graphic using ```hp2ps -e8in -c <filename.hp>```. This creates a PS file with a graph that shows memory usage over
time. The _-hc_ option creates a graph that shows memory usage my closures and functions.

![Initial version, memory usage using -hc](/images/2016-09-22-HaskellProfiling-InitialVersion-hc-v1.jpg "Initial version, memory usage using -hc")

That doesn't leave any doubts about which function is the memory hog here. Lets see what types are actually taking up all that space. This can be done by running the command ```stack exec -- RecommendationSystemInitialVersionProfiling ./testRatings.gz +RTS -s -p -hy```. 


![Initial version, memory usage using -hy](/images/2016-09-22-HaskellProfiling-InitialVersion-hy-v1.jpg "Initial version, memory usage using -hy")

This makes it clear that Maps and Doubles are taking up enormous amounts of space. Something fishy is going on in the ```runIteration``` function. We get a clue from the fact that the memory usage spikes up and then suddenly goes down. This cycles goes on for 5 times, which is the same as the number of dimensions we are running. Now, at the end of the run for each dimension, we print some stats about errors and such. This is what is causing the release of all the allocated memory. Looking at the code, that specific function is all about reading from a map, doing some evaluations on doubles and inserting them back in the map in a accumulating foldl loop. It is pretty clear that the maps aren't really getting evaluated at every step, but are being built up on in thunks in true Haskell lazy fashion. How can we enforce these maps to be evaluated at each step? We use _BangPatterns_! We also use the `seq` operator. Let's change the function to the one below and see the effect it has on memory.

```haskell
runIteration :: Config 
  -> (M.Map User Double, M.Map Item Double) 
  -> DV.Vector Rating 
  -> (M.Map User Double, M.Map Item Double)
runIteration config = DV.foldl' f
  where
    f (!uV, !iV) (Rating r u i) = (uV', iV')
      where
        dStart = startingEstimate config
        lRate = learningRate config
        reg = regFactor config
        ua  = M.findWithDefault dStart u uV
        ia  = M.findWithDefault dStart i iV
        eps = r - ua * ia
        dua = lRate * (eps * ia - reg * ua)
        dia = lRate * (eps * ua - reg * ia)
        ua' = ua + dua
        ia' = ia + dua
        uV' = ua' `seq` M.insert u ua' uV
        iV' = ia' `seq` M.insert i ia' iV
```

Running it through the same set of commands, we see the following memory usage profile:

```
  12,760,008,856 bytes allocated in the heap
   3,710,445,272 bytes copied during GC
      54,785,880 bytes maximum residency (91 sample(s))
       2,281,096 bytes maximum slop
             107 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0     24886 colls,     0 par    1.578s   1.583s     0.0001s    0.0005s
  Gen  1        91 colls,     0 par    1.527s   1.653s     0.0182s    0.0509s

  TASKS: 4 (1 bound, 3 peak workers (3 total), using -N1)

  SPARKS: 0 (0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.001s  (  0.001s elapsed)
  MUT     time    5.916s  (  6.394s elapsed)
  GC      time    2.659s  (  2.791s elapsed)
  RP      time    0.000s  (  0.000s elapsed)
  PROF    time    0.446s  (  0.446s elapsed)
  EXIT    time    0.000s  (  0.001s elapsed)
  Total   time    9.054s  (  9.187s elapsed)

  Alloc rate    2,156,874,450 bytes per MUT second

  Productivity  65.7% of total user, 64.8% of total elapsed

```

Alright! We got the time down to a little over 9s. We got the productivity to 65% and we reduced the total memory usage by a factor of 6! That is some saving for a few innocuous looking changes. What exactly is going on there? With the _BangPatterns_, we are telling the compiler before-hand that we expect this parameter to be strictly evaluated since we know we would use all the information from it. It is syntactic sugar for the `seq` function that has the prototype `seq :: a -> b -> b`. This
function evaluates the first argument to WHNF before returning the second argument. We are enforcing all the evaluations to complete before moving forward with the next iteration in the foldl.

Lets take a look at the graphs, as they would now tell us if we are done. 

![memory usage using -hc](/images/2016-09-22-HaskellProfiling-InitialVersion-hc-v2.jpg "memory usage using -hc")

![memory usage using -hy](/images/2016-09-22-HaskellProfiling-InitialVersion-hy-v2.jpg "memory usage using -hy")

The peaks have gone down considerably, but we still see the same pattern of high allocation and then de-allocation. The first graph tells us that a big portion of this is in the _shuffleVectorST_ code. Lets take a look at the code. This code was adapted from [Random Shuffle](https://wiki.haskell.org/Random_shuffle):

```haskell
shuffleVectorST :: R.RandomGen t => DV.Vector a -> t -> (DV.Vector a, t)
shuffleVectorST xs gen = runST $ do
  g <- newSTRef gen
  let n = DV.length xs
      randomRST lohi = do
        (a, s') <- liftM (R.randomR lohi) (readSTRef g)
        writeSTRef g s'
        return a
      copyVectorToMutable xs' = do
        v <- GM.new n
        forM_ [0 .. (n - 1)] $ \i ->
          GM.write v i (xs' DV.! i)
        return v

  v <- copyVectorToMutable xs
  forM_ [0 .. (n - 1)] $ \i -> do
    j <- randomRST (i, n - 1)
    GM.swap v i j
  ret <- DV.unsafeFreeze v
  g' <- readSTRef g
  return (ret, g')
```
This implements Fisher-Yates shuffle with a mutable vector. The reason for this choice was that modifying a vector is _O(n)_. It can also be implemented by using Maps, but Maps in Haskell have complexity of _log(n)_ for both finds and inserts. This would bring the shuffle complexity with Maps to _n log(n)_.

To shuffle a vector, we are taking care to copy it to a mutable vector so we do the shuffle non-destructively. But that is manifesting itself as the big spike in memory usage. Also note the _MUT_ARR_PTRS_FROZEN_ part, which may not be significant compared to the peak, but shows the same pattern. That section is related to the GHC putting guard rails around anything mutable since that needs special care. Let's see if we can modify the code. Instead of copying the vector, we can permute a vector
of contiguous indices and then back-permute that to get a new vector. The code would look something like the following:

```haskell
shuffleVectorST :: R.RandomGen t => DV.Vector a -> t -> (DV.Vector a, t)
shuffleVectorST !xs gen = runST $ do
  g <- newSTRef gen
  let n = DV.length xs
      randomRST lohi = do
        (a, s') <- liftM (R.randomR lohi) (readSTRef g)
        writeSTRef g s'
        return a
      indexVector = DV.thaw $ DV.generate n id

  v <- indexVector
  forM_ [0 .. (n - 1)] $ \i -> do
    j <- randomRST (i, n - 1)
    GM.swap v i j
  ret <- DV.freeze v
  g' <- readSTRef g
  return (DV.backpermute xs ret, g')
```

Let's check out the performance with this change. 
```
  12,463,617,336 bytes allocated in the heap
   2,334,594,160 bytes copied during GC
       9,553,624 bytes maximum residency (193 sample(s))
       1,260,264 bytes maximum slop
              21 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0     24094 colls,     0 par    1.424s   1.429s     0.0001s    0.0004s
  Gen  1       193 colls,     0 par    0.640s   0.652s     0.0034s    0.0150s

  TASKS: 4 (1 bound, 3 peak workers (3 total), using -N1)

  SPARKS: 0 (0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.000s  (  0.001s elapsed)
  MUT     time    5.722s  (  5.802s elapsed)
  GC      time    2.014s  (  2.030s elapsed)
  RP      time    0.000s  (  0.000s elapsed)
  PROF    time    0.051s  (  0.052s elapsed)
  EXIT    time    0.001s  (  0.001s elapsed)
  Total   time    7.817s  (  7.833s elapsed)

  Alloc rate    2,178,072,188 bytes per MUT second

  Productivity  73.6% of total user, 73.4% of total elapsed

```
So we just shaved off a little over a second from the total run-time. The productivity is up to about 74% now. We are also using about 15MB less memory. So we made some progress. Let's look at the graphs now:

![memory usage using -hc](/images/2016-09-22-HaskellProfiling-InitialVersion-hc-v3.jpg "memory usage using -hc")

![memory usage using -hy](/images/2016-09-22-HaskellProfiling-InitialVersion-hy-v3.jpg "memory usage using -hy")

It looks like we got rid of the previous peaks, but the MUT_ARR_PTRS_FROZEN is now the prominant pattern, again doing a lot of allocation and de-allocation. Do we have any _mutations_ happenning elsewhere in the code? The usage of IORef in `runFullEstimationIO` and `runTillConvergenceIO` seems suspect. In both the cases, we are accumulating state. Since we want to emit some status to stdout, we are stuck in the IO monad. There is one function that might help us here. That is `foldM :: Monad m => (a -> b -> m a) -> a -> [b] -> m a`. Using this, we should be able to get rid of IORefs in the two functions. The code for these would now look like this:
```haskell
runTillConvergenceIO :: Config -> FunkSVD -> IO FunkSVD
runTillConvergenceIO config funkSVD =
  do

  let func (!dUv, !dIv) _ = do
        shuffledResiduals <- R.shuffleVectorIO $ residuals funkSVD
        return $ runIteration config (dUv, dIv) shuffledResiduals

  (dUv, dIv) <- foldM func (M.empty, M.empty) [1 .. (numIterations config)]

  let residuals' = updateResiduals (residuals funkSVD) dUv dIv
      uV = userVectors funkSVD
      iV = itemVectors funkSVD
  return funkSVD{residuals = residuals', userVectors = appendVectors uV dUv, itemVectors = appendVectors iV dIv}

runFullEstimationIO :: DV.Vector Rating -> Config -> IO FunkSVD
runFullEstimationIO ratings config =
  do
  (trainingSet, testSet) <- R.randomSplitIO (4 * DV.length ratings `div` 5) ratings
  let funkSVDInput = runReader (generateFunkSVDInput trainingSet) config

  let func !funkSVD dim = do
        putStrLn $ "Working on dimension " ++ show dim
        funkSVD' <- runTillConvergenceIO config funkSVD
        putStrLn $ "Average residuals = " ++ (show . globalMean abs) (residuals funkSVD')
        putStrLn $ "Residuals RMSE = " ++ show (sqrt (globalMean (**2) (residuals funkSVD')))
        putStrLn $ "Training RMSE = " ++ show (getRMSE trainingSet funkSVD')
        putStrLn $ "Test RMSE = " ++ show (getRMSE testSet funkSVD')
        return funkSVD'

  foldM func funkSVDInput [1 .. (numDimensions config)]

```
With that done, lets take this for a spin.
```
  10,760,815,200 bytes allocated in the heap
   2,400,955,744 bytes copied during GC
       4,096,808 bytes maximum residency (291 sample(s))
         336,232 bytes maximum slop
              12 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0     20627 colls,     0 par    1.261s   1.266s     0.0001s    0.0004s
  Gen  1       291 colls,     0 par    0.441s   0.443s     0.0015s    0.0027s

  TASKS: 4 (1 bound, 3 peak workers (3 total), using -N1)

  SPARKS: 0 (0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.000s  (  0.001s elapsed)
  MUT     time    5.404s  (  5.478s elapsed)
  GsHC      time    1.655s  (  1.661s elapsed)
  RP      time    0.000s  (  0.000s elapsed)
  PROF    time    0.048s  (  0.048s elapsed)
  EXIT    time    0.000s  (  0.000s elapsed)
  Total   time    7.134s  (  7.140s elapsed)

  Alloc rate    1,991,176,794 bytes per MUT second

  Productivity  76.1% of total user, 76.1% of total elapsed

```

Awesome! We got the memory usage down to a total of 12 MB, we shaved off another second from the total runtime and the productivity is also up a notch. Lets take a look at the graphs to see where we stand:

![memory usage using -hc](/images/2016-09-22-HaskellProfiling-OptimizedVersion-hc.jpg "memory usage using -hc")

![memory usage using -hy](/images/2016-09-22-HaskellProfiling-OptimizedVersion-hy.jpg "memory usage using -hy")

This looks a lot better. Ratings occupy most of the space, and they should. We do infact read the initial ratings and then hold on to them for the entire run of the program. At every dimension, we also maintain residuals along with the ratings. For all dimensions, we keep appending to the user and item vectors, so it makes sense that the memory usage by Maps would grow a little over the run of the program. 

There is one last thing that I want to show before we end this discussion. Note that we were always using the _profiling_ executable when generating the graphs. Haskell compilation can perform a large number whole-sale optimizations that exploit the pure nature of the code. When generating profiling binaries, though, the compiler holds back some optimizations. This can be seen clearly, when we run the optimized binary. 
```haskell
   6,999,678,952 bytes allocated in the heap
   1,252,786,760 bytes copied during GC
       2,321,296 bytes maximum residency (207 sample(s))
         136,280 bytes maximum slop
               7 MB total memory in use (0 MB lost due to fragmentation)

                                     Tot time (elapsed)  Avg pause  Max pause
  Gen  0     13041 colls,     0 par    0.879s   0.881s     0.0001s    0.0004s
  Gen  1       207 colls,     0 par    0.217s   0.219s     0.0011s    0.0019s

  TASKS: 4 (1 bound, 3 peak workers (3 total), using -N1)

  SPARKS: 0 (0 converted, 0 overflowed, 0 dud, 0 GC'd, 0 fizzled)

  INIT    time    0.000s  (  0.000s elapsed)
  MUT     time    3.136s  (  3.149s elapsed)
  GC      time    1.096s  (  1.099s elapsed)
  EXIT    time    0.000s  (  0.000s elapsed)
  Total   time    4.262s  (  4.249s elapsed)

  Alloc rate    2,232,027,670 bytes per MUT second

  Productivity  74.3% of total user, 74.5% of total elapsed

```
The optimized run shaved off nearly 50% of both both the total run-time and total memory usage.

##Conclusions
Haskell is inherently a pure functional lazy language. This laziness helps optimize away things that are never needed and can help in a lot of patterns like infinite streams. The pure functionalness lets GHC do a lot of optimizations on the code. However, when using it for computationally intensive tasks like here, where we know we would be evaluating all the numbers and every iteration would depend on the result of the previous iteration, giving the compiler indications using the _Bang Patterns_ and/or _seq_ can go a long way in getting significant speed-ups and memory efficiency. 

The profiling binaries can give a very good idea of what the program run-time memory usage looks like. We can also generate charts to help guide the optimization effort. Once, all that is done, we should expect the optimized binaries to peform a lot better than the profiling ones even.

##Reproducing these stats and charts
The code for both the initial and optimized version is available [here](https://github.com/shubhamchopra/RecommenderSystem). To get movie recommendations, you can get data from Netflix or [MovieLens](http://grouplens.org/datasets/movielens/). I am currently using GHC 7.10.3. You might get different numbers depending on the sample of data you use, the processor and such, but the shape/pattern of the charts would likely be similar.
