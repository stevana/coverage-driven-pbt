# Coverage-guided property-based testing

*Work in progress, please don't share, but do feel free to get
involved!*

Almost ten years ago, back in 2015, Dan Luu wrote a
[post](https://danluu.com/testing/) asking why coverage-guided
property-based testing wasn't a thing.

In this post I'll show how one can implement such a thing starting from
scratch.

The technique is programming language agnostic and doesn't rely on any
language-specific instrumentation of the software under test.

## Background and prior work

Fuzzing has an interesting origin. It was a class
[project](http://pages.cs.wisc.edu/~bart/fuzz/CS736-Projects-f1988.pdf)
in an advanced operating systems course taught by Barton Miller at the
University of Wisconsin in 1988.

The project was inspired by the observation that back then, if you
logged into your workstation via a dail-up modem from home and it
rained, then frequently random characters would appear in the terminal.
The line noise wasn't the surprising thing, but rather that the extra
characters would sometimes crash the program that they tried to invoke.

Among these programs were basic utilities such as vi, mail, cc, make,
sed, awk, sort, etc, and it was reasonable to expect that these would
give an error message rather than crash and core dump if fed with some
extra characters caused by the rain.

So the project set out to basically recreate what the rain did, but more
effectively, but essentially generating random noise (stream of bytes)
and feeding that to different utilities and see if they crashed.

A couple of years later Barton et al published [*An empirical study of
the reliability of UNIX
utilities*](https://dl.acm.org/doi/10.1145/96267.96279) (1990).

Inserting random characters was effective in finding corner cases where
the programmers forgot to properly validate the input from the user.

However it wouldn't trigger bugs hiding deeper under the surface.

This changed around 2007 when people [started
thinking](https://lcamtuf.coredump.cx/afl/historical_notes.txt) about
how fuzzing can be combined with [evolutionary
algorithms](https://en.wikipedia.org/wiki/Evolutionary_algorithm).

XXX: The idea being that...

- AFL (2013),

Coverage-guided fuzzers, such as [American Fuzzy
Lop](https://lcamtuf.coredump.cx/afl/) (AFL), have been very successful
in finding [bugs](https://lcamtuf.coredump.cx/afl/#bugs) in programs
that take bytes as input. That means any kind of programs that takes
user strings, command line arguments or files as inputs, parsers, but
also arrays of ints, etc.

- <https://lcamtuf.blogspot.com/2014/11/pulling-jpegs-out-of-thin-air.html>

- AFL is the tool that Dan Luu explicitly mentions, so let's stop here
  and go back to his point, before looking at else has happened since

- PBT

- I've written about the
  [history](https://stevana.github.io/the_sad_state_of_property-based_testing_libraries.html#the-history-of-property-based-testing)
  of property-based testing and explained how it
  [works](https://stevana.github.io/the_sad_state_of_property-based_testing_libraries.html#pure-property-based-testing-recap)
  already, so I won't take up space by repeating myself here.

- The idea of combining coverage-guidance and PBT

- Now let's have a look at what has happend since Dan's post.

- First off, at some point he added an update to his post where he
  explicitly mentiones:

  - Go-fuzz?

    - <https://adalogics.com/blog/structure-aware-go-fuzzing-complex-types>

  - Hypothesis

    - Has notion of coverage:
      <https://hypothesis.readthedocs.io/en/latest/details.html#hypothesis.event>)
    - But coverage-guided testing was
      [removed](https://github.com/HypothesisWorks/hypothesis/pull/1564/commits/dcbea9148be3446392bc3af8892d49f3cc74fbe3)

- When you search for coverage guided property-based testing

- [FuzzChick](https://dl.acm.org/doi/10.1145/3360607) (2019). Not
  released, lives in an [unmaintained
  branch](https://github.com/QuickChick/QuickChick/compare/master...FuzzChick)
  that [doesn't
  compile](https://github.com/QuickChick/QuickChick/issues/277)?

  - coverage info is [same as in
    AFL](https://youtu.be/RR6c_fiMfJQ?t=2226)

- FuzzChick, related work mentions:

- [JQF + Zest: Coverage-guided semantic fuzzing for
  Java](https://github.com/rohanpadhye/jqf)?

- [Crowbar](https://github.com/stedolan/crowbar)

  - [extended abstract from OCaml
    workshop](https://github.com/ocaml/ocaml.org-media/blob/086fc25105cbccb188c28ec74126d72962921ff8/meetings/ocaml/2017/extended-abstract__2017__stephen-dolan_mindy-preston__testing-with-crowbar.pdf)
    (2017)
  - Uses fuzzing indirectly to generate the data?

- [libfuzzer](https://llvm.org/docs/LibFuzzer.html) and it's successor
  [FuzzTest](https://github.com/google/fuzztest) ("It is a
  first-of-its-kind tool that bridges the gap between fuzzing and
  property-based testing") (2022?)

- [honggfuzz](https://github.com/google/honggfuzz)

  - open PR to add it to cargo fuzz:
    <https://github.com/rust-fuzz/book/pull/14>

- [Structure-aware fuzzing using libfuzzer-sys in
  Rust](https://rust-fuzz.github.io/book/cargo-fuzz/structure-aware-fuzzing.html)

- [MUTAGEN: Reliable Coverage-Guided, Property-Based Testing using
  Exhaustive
  Mutations](https://www.mista.me/assets/pdf/icst23-preprint.pdf) (2023)

  - <https://github.com/OctopiChalmers/mutagen/>
  - Uses GHC
    [plugin](https://github.com/OctopiChalmers/mutagen/blob/main/src/Test/Mutagen/Tracer/Plugin.hs)
    to annotate source code with coverage information of: function
    clauses, case statements, multi-way ifs, and each branch of
    if-then-else expressions

## Examples and the main idea of coverage-guidance

To get a better understanding of why coverage-guidence is needed, let's
consider a concrete example.

This example is due to Dmitry Vyukov, the main author of
[go-fuzz](https://github.com/dvyukov/go-fuzz), but it's basically an
easier to understand version of the example from Dan's post[^1].

    func sut(input []byte) {
        slice := []int{}
        if input[0] == 'A' {
            if input[1] == 'B' {
                if input[2] == 'C' {
                    if input[3] == 'D' {
                        slice[input[4]] = 1 // out-of-bounds here
                    }
                }
            }
        }
    }

Even if we generate random inputs of exactly the length 4, it would
still take
$O(2^8 \cdot 2^8 \cdot 2^8 \cdot 2^8) = O((2^8)^4) = O(2^{32}) = 4294967296$
tries to trigger the bug (and obviously even longer if we tried arrays
of varying length).

With coverage-guidance we keep track of inputs that resulted in
increased coverage. So, for example, if we generate the array
`[]byte{'A'}` we get further into the nested ifs, and so we take note of
that and start generating longer arrays that start with 'A' and see if
we get even further, etc.

By building on previous succeses in getting more coverage, we can
effectively reduce the problem to only need
$O(2^8 + 2^8 + 2^8 + 2^8) = O(2^8 \cdot 4) =
O(2^{10}) = 1024$ tries. With other words coverage-guidence turns an
exponential problem into a polynomial problem!

## Prototype implementation

Great, but where do we get this coverage information from?

AFL and `go-fuzz` both get it from the compiler.

AFL injects code into every [basic
block](https://en.wikipedia.org/wiki/Basic_block).

When I've been thinking about how to implement coverage-guided
property-based testing in the past, I always got stuck thinking that
parsing the coverage output from the compiler in between test case
generation rounds would be annoying and slow.

I didn't know that you could get this information from a library
provided by the GHC compiler in Haskell, until I read Shae "shapr"
Erisson does in his
[post](https://shapr.github.io/posts/2023-07-30-goldilocks-property-tests.html).

While this certainly makes things easier, it wasn't until I read about
Antithesis' ["sometime
assertions"](https://antithesis.com/docs/best_practices/sometimes_assertions.html)
that I started seeing a really simple solution to the problem.

These "sometimes assertions" can be thought of as generalised coverage,
in that if we would annotate every single line, expression or branch
with a sometime assertion we'd get back line-, expression-, or
branch-based coverage.

But the cool thing about "sometimes assertions" is that we don't need to
annotate every single line, expression or branch, we can annotate
*interesting* points in our program.

The final piece of the puzzle, and I think this is the only original
idea that this post adds, is that property-based testing already has
functionality for implementing "sometimes assertions": the `label`,
`classify` and `collect` machinary for gathering run-time statistics of
the generated data!

This machinary is [crucial](https://www.youtube.com/watch?v=NcJOiQlzlXQ)
for writing good tests and has been part of the QuickCheck
implementation since the very first version[^2]!

So the question is: can we implement coverage-guided property-based
testing using the internal notion of coverage that property-based
testing already has?

- QuickCheck as defined in the appendix of the original
  [paper](https://dl.acm.org/doi/10.1145/351240.351266) (ICFP, 2000)

  - Extended monadic properties

- Edsko de Vries'
  [Mini-QuickCheck](https://www.well-typed.com/blog/2019/05/integrated-shrinking/)

``` haskell
type Seed = Int

type Shrinking = Bool

checkM :: forall a c. (Show a, Show c)
       => Seed -> Int -> Gen a -> (Shrinking -> Coverage c -> [a] -> IO Bool) -> IO ()
checkM seed numTests gen p = do
  coverage <- emptyCoverage
  mShrinkSteps <- go numTests coverage 0 []
  case mShrinkSteps of
    Nothing -> do
      putStrLn "\nOK"
      cov <- readCoverage coverage
      putStrLn $ "Coverage: " ++ show cov
    Just shrinkSteps -> do
      putStrLn $ "Failed: " ++ show (NonEmpty.head shrinkSteps)
      putStrLn $ "Shrinking: " ++ show shrinkSteps
      putStrLn $ "#Shrinks: " ++ show (NonEmpty.length shrinkSteps - 1)
      putStrLn $ "Shrunk: " ++ show (NonEmpty.last shrinkSteps)
      cov <- readCoverage coverage
      putStrLn $ "Coverage: " ++ show cov
  where
    go :: Int -> Coverage c -> Int -> [a] -> IO (Maybe (NonEmpty [a]))
    go 0 _cov _before _cmds = return Nothing
    go n _cov  before  cmds = do
      let sz  = n * 3 `div` 2
      let cmd = generate sz (mkStdGen (seed + n)) gen
      cmds' <- randomMutation cmds cmd
      (ok, after) <- withCoverage (\cov -> p False cov cmds')
      if ok
      then do
        let diff = compareCoverage before after
        case diff of
          Increased -> do
            -- putStr "p"
            go (n - 1) _cov after cmds'
          Same      -> do
            -- putStr "p"
            go (n - 1) _cov after cmds'
          Decreased -> do
            -- putStr "."
            go (n - 1) _cov before cmds
      else do
        putStrLn "\n(Where `p` and `.` indicate picked and dropped values respectively.)"
        Just <$> shrinker (p True _cov) (shrinkList (const [])) cmds'
```

``` haskell
```

``` haskell
randomMutation :: [a] -> a -> IO [a]
randomMutation [] x = return [x]
randomMutation xs x = do
  appendOrUpdate <- randomIO -- XXX: nondet
  if appendOrUpdate
  then return (xs ++ [x])
  else do
    ix <- randomRIO (0, length xs - 1)
    return (update ix xs x)
  where
    update :: Int -> [a] -> a -> [a]
    update ix xs0 x' = case splitAt ix xs0 of
      (before, _x : after) -> before ++ x' : after
      (_, []) -> error "update: impossible"
```

``` haskell
newtype Coverage a = Coverage (IORef (Set a))

emptyCoverage :: IO (Coverage a)
emptyCoverage = Coverage <$> newIORef Set.empty

addCoverage :: Ord a => Coverage a -> a -> IO ()
addCoverage (Coverage ref) x = modifyIORef' ref (Set.insert x)

readCoverage :: Coverage a -> IO (Set a)
readCoverage (Coverage ref) = readIORef ref

checkCoverage :: Coverage a -> IO Int
checkCoverage (Coverage ref) = Set.size <$> readIORef ref

data CoverageDiff = Decreased | Same | Increased
  deriving Eq

compareCoverage :: Int -> Int -> CoverageDiff
compareCoverage before after = case compare after before of
  LT -> Decreased
  EQ -> Same
  GT -> Increased

withCoverage :: (Coverage c -> IO a) -> IO (a, Int)
withCoverage k = do
  c <- emptyCoverage
  x <- k c
  after <- checkCoverage c
  return (x, after)
```

``` haskell
newtype Gen a = Gen (Size -> StdGen -> a)

type Size = Int
```

The full source code is available
[here](https://github.com/stevana/coverage-guided-pbt).

## Testing some examples with the prototype

## Conclusion and further work

- Makes more sense for stateful systems than pure functions? Or atleast
  properties that expect a sequence of inputs?

- Don't rerun all commands for every newly generate command

  - only reset the system when shrinking

- Problem of strategy (pick something as basis for progress): coverage,
  logs, value of memory, helps bootstap the process. Generalise to
  support more?

- Local maxima?

- Problem of tactics: picking a good input distributed for the testing
  problem at hand. Make previous input influence the next input?
  Dependent events, e.g. if one packet gets lost, there's a higher
  chance that the next packet will be lost as well.

- Save `(Coverage, Mutation, Frequency, Coverage)` stats?

- More realistic example, e.g.: leader election, transaction rollback,
  failover?

- Annoying to sprinkle sometimes assertions everywhere?

  - Can it be combined with logging or tracing?

- Use size parameter to implement AFL heuristic for choosing integers?
  Or just use `frequency`?

- Type-generic mutation?

- sometimes_each?

## See also

- <https://aflplus.plus/docs/power_schedules/>
- <https://github.com/mboehme/aflfast>
- <https://mboehme.github.io/paper/CCS16.pdf>
- <https://carstein.github.io/fuzzing/2020/04/18/writing-simple-fuzzer-1.html>
- <https://carstein.github.io/fuzzing/2020/04/25/writing-simple-fuzzer-2.html>
- <https://carstein.github.io/fuzzing/2020/05/02/writing-simple-fuzzer-3.html>
- <https://carstein.github.io/fuzzing/2020/05/21/writing-simple-fuzzer-4.html>
- [How Antithesis finds bugs (with help from the Super Mario
  Bros)](https://antithesis.com/blog/sdtalk/)
- Swarm testing
- [AFL
  "whitepaper"](https://lcamtuf.coredump.cx/afl/technical_details.txt)
- [AFL mutation
  heuristics](https://lcamtuf.blogspot.com/2014/08/binary-fuzzing-strategies-what-works.html)
- Shae "shapr" Erisson's post [*Run property tests until coverage stops
  increasing*](https://shapr.github.io/posts/2023-07-30-goldilocks-property-tests.html) (2023) (2023)
  and [trynocular](https://github.com/shapr/trynocular) library.
  - This only uses coverage as a stopping condition, not to actually
    drive the generation...

[^1]: Here's Dan's example in full:

        // Checks that a number has its bottom bits set
        func some_filter(x int) bool {
            for i := 0; i < 16; i = i + 1 {
                if !(x&1 == 1) {
                    return false
                }
                x >>= 1
            }
            return true
        }

        // Takes an array and returns a non-zero int
        func dut(a []int) int {
            if len(a) != 4 {
                return 1
            }

            if some_filter(a[0]) {
                if some_filter(a[1]) {
                    if some_filter(a[2]) {
                        if some_filter(a[3]) {
                            return 0 // A bug! We failed to return non-zero!
                        }
                        return 2
                    }
                    return 3
                }
                return 4
            }
            return 5
        }

    As I hope we can agree, it's very similar to Dmitry's example,
    except it's a bit less clear what exactly happens in the if
    statement.

[^2]: See the appendix of the original
    [paper](https://dl.acm.org/doi/10.1145/351240.351266) that first
    introduced property-based testing. It's interesting to note that
    this functionality is older than shrinking.
