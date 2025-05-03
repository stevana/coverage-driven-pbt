# Coverage-guided property-based testing

*Work in progress, please don't share, but do feel free to get involved!*

Ten years ago, back in 2015, Dan Luu wrote a
[post](https://danluu.com/testing/) asking why coverage-guided property-based
testing wasn't a thing. 

In this post I'll survey the coverage-guided landscape, looking at what was
there before Dan's post and what has happened since. I'll also show how to add
basic coverage-guidance to the first version of the
original property-based testing tool, QuickCheck, in about 35 lines of code.

Unlike many previous implementations of this idea, the technique used to
implement coverage-guidance is programming language agnostic and doesn't rely
on any language-specific instrumentation of the software under test.

## Motivation

Before we start, let me try to motivate why one would want to combine
coverage-guided fuzzing and property-based testing to begin with.

Consider the following example[^1], where an error is triggered if some input
byte array starts with the bytes `"bad!"`:

```
func sut(input []byte) {
    if input[0] == 'b' {
        if input[1] == 'a' {
            if input[2] == 'd' {
                if input[3] == '!' {
                    panic("input must not be bad!")
                }
            }
        }
    }
}
```

What are the odds that a property-based testing tool (without
coverage-guidance) would be able to find the error?

To make the calculation easier, let's say that we always generate arrays of
length $4$. An ASCII character consists of seven bits, so it has $2^7$ possible
values.

That means that the probability is $\frac{1}{2^7} \cdot \frac{1}{2^7} \cdot
\frac{1}{2^7} \cdot \frac{1}{2^7} = 3.72529 \cdot 10^{-9}$. For comparison,
the probability of winning the lottery (the Powerball jackpot) is $1$ in
$292.2$ million $= 3.42231 \cdot 10^{-9}$.

With coverage-guidance we keep track of inputs that resulted in increased
coverage. So, for example, if we generate the array `[]byte{'b'}` we get
further into the nested ifs, and so we take note of that and start generating
longer arrays that start with `'b'` and see if we get even further, etc. By
building on previous successes in getting more coverage, we can effectively
reduce the problem to only need $\frac{1}{2^7} + \frac{1}{2^7} + \frac{1}{2^7} + 
\frac{1}{2^7} = 0.03125$.

In other words coverage-guidance turns an exponential problem into a polynomial
problem!

## Background and prior work

There's a lot to cover here, so I'll split it up in before and after
Dan's post.

### Before 2015

Fuzzing has an interesting origin. It started as a class
[project](http://pages.cs.wisc.edu/~bart/fuzz/CS736-Projects-f1988.pdf) in an
advanced operating systems course taught by Barton Miller at the University of
Wisconsin in 1988.

The project was inspired by the observation that back then, if you logged into
your workstation via a dail-up modem from home and it rained, then frequently
random characters would appear in the terminal. The line noise wasn't the
surprising thing, but rather that the extra characters would sometimes crash
the program that they tried to invoke. Among these programs were basic
utilities such as `vi`, `mail`, `cc`, `make`, `sed`, `awk`, `sort`, etc, and it
was reasonable to expect that these would give an error message rather than
crash and core dump if fed with some extra characters caused by the rain.

So the project set out to basically recreate what the rain did, but more
effectively, but essentially generating random noise (stream of bytes) and
feeding that to different utilities and see if they crashed. A couple of years
later Barton et al published [*An empirical study of the reliability of UNIX
utilities*](https://dl.acm.org/doi/10.1145/96267.96279) (1990) which documents
their findings.

Inserting random characters was effective in finding corner cases where the
programmers forgot to properly validate the input from the user. However it
wouldn't trigger bugs hiding deeper under the surface, such as the `"bad!"`
example from the previous section. 

This changed around 2007 when people [started
thinking](https://lcamtuf.coredump.cx/afl/historical_notes.txt) about how
fuzzing can be combined with [evolutionary
algorithms](https://en.wikipedia.org/wiki/Evolutionary_algorithm). The idea
being that instead of generating random bytes all the time as with classical
fuzzing, we can use coverage information from one test to mutate the input for
the next test. Or to use the evolution metaphor: seeds that lead to better
coverage are mutated with the hope that they will lead to even better coverage. 

One of the first, and perhaps still most widely known, such *coverage-guided*
fuzzers is Michał Zalewski's [AFL](https://lcamtuf.coredump.cx/afl/) (2013). To
get a feel for how effective AFL-style coverage-guidance is, check out the list
of [bugs](https://lcamtuf.coredump.cx/afl/#bugs) that it found and this post
about how it manages to figure out the [jpeg
format](https://lcamtuf.blogspot.com/2014/11/pulling-jpegs-out-of-thin-air.html)
on its own[^2].

Since AFL is the tool that Dan explicitly mentions in his post, let's stop at
this point and go back to his point, before looking at what happened since
with coverage-guided fuzzers. 

Recall that Dan was asking why this idea of coverage-guidance wasn't present in
property-based testing tools. I've written about the
[history](https://stevana.github.io/the_sad_state_of_property-based_testing_libraries.html#the-history-of-property-based-testing)
of property-based testing and explained how it
[works](https://stevana.github.io/the_sad_state_of_property-based_testing_libraries.html#pure-property-based-testing-recap)
already, so I won't take up space by repeating myself here. Let's just note
that the [original
paper](https://www.cs.tufts.edu/~nr/cs257/archive/john-hughes/quick.pdf) on
property-based testing was published in 2000. So when Dan wrote asking about
this question property-based testing would have been fifteen and AFL two years
old.

The main difference between property-based testing and fuzzing is that
fuzzing requires less work by the user. Simply hook up the byte generator to
the function that expects bytes as input and off it goes looking for crashes.
Property-based testing on the other hand can test functions that take arbitrary
data structures as input (not just bytes), but you have to describe how to
generate such inputs. 

Fuzzing only looks for crashes, while property-based testing lets you specify
arbitrary relations that should hold between the input and output of the system
under test. For example, we can generate binary search trees and check that
after we insert something into an arbitrary binary search tree then it will
remain sorted (when we do an in-order traversal). Fuzzing can't check such
properties, and furthermore because they generate random bytes it's unlikely
that they'll even generate a valid binary search tree to begin with (without
lots of coverage-driven testing).

On the other hand, the coverage of property-based tests is only as good as
the user provided generators. Corner cases where slightly modified data leads
to e.g. exception handling is not explored automatically, and coverage
information is not used to guide the input generation process.

By now we should have enough background to see that the idea of combining
coverage-guidance and property-based testing makes sense. Basically what we'd
like is to:

1. Use user provided generators to kick start the exploration in the right
   direction;
2. Mutate the generated data, while preserving its type, to surface bugs that
   user provided generators alone wouldn't have found;
3. Use coverage information to iteratively get deeper into the state space of
   the system under test, like in the "bad!" example from the motivation.

### After 2015

Having covered what had happened before Dan's post, let's have a look at what
has happened in the ten years since his post. First off, it's worth noting that
at some point Dan added an update to his post:

> "Update: Dmitry Vyukov's Go-fuzz, which looks like it was started a month
> after this post was written, uses the approach from the proof of concept in
> this post of combining the sort of logic seen in AFL with a QuickCheck-like
> framework, and has been shown to be quite effective. I believe David R.
> MacIver is also planning to use this approach in the next version of
> hypothesis."

So let's start there, with Go-fuzz. At a first glance, all functions that are
tested with Go-fuzz need to take an array of bytes as input. My initial thought
was, how can I write properties which involve generating more interesting data
structures? But it turns out it's
[possible](https://news.ycombinator.com/item?id=40876822), as somebody pointed
out in the comments of an old post of mine. Furthermore there are also
[ways](https://adalogics.com/blog/structure-aware-go-fuzzing-complex-types) of
making the fuzzer aware of more complex data structures. I haven't played
around enough with this to be able to compare how well shrinking works yet
though, but overall I'd say this ticks the box of being a property-based
testing tool that is also coverage-guided.

Next up Dan mentions Python's Hypothesis. I was searching through the
documentation trying to find out how coverage-guidance works, but I couldn't
find anything. Searching through the repository I found the following [release
note](https://github.com/HypothesisWorks/hypothesis/pull/1564/commits/dcbea9148be3446392bc3af8892d49f3cc74fbe3)
(2018):

> "This release deprecates the coverage-guided testing functionality,
> as it has proven brittle and does not really pull its weight.
> 
> We intend to replace it with something more useful in the future, but the
> feature in its current form does not seem to be worth the cost of using, and
> whatever replaces it will likely look very different."

As far as I can tell, it hasn't been reintroduced since. However it's possible
to hook Hypothesis up to [use external
fuzzers](https://hypothesis.readthedocs.io/en/latest/details.html#use-with-external-fuzzers).
Hypothesis already uses random bytes as basis for its generators, unlike
QuickCheck which uses an integer seed, so I suppose that the external fuzzers
essentially fuzz the random input bytes that in turn are used to generate more
structured input. 

Traditional fuzzers are usually designed to target a single binary, where as
the test suite which uses property-based testing typically has many properties.
The [HypoFuzz](https://hypofuzz.com/docs/features.html#fuzzer-details) tool
works around this mismatch by scheduling fuzzing time among the many Hypothesis
properties in your test suite.

What else has happened since Dan's post?

One of the first things I noticed is that AFL is no longer
[maintained](https://lcamtuf.coredump.cx/afl/):

> "Note: AFL hasn't been updated for a couple of years; while it should still
> work fine, a more complex fork with a variety of improvements and additional
> features, known as AFL++, is available from other members of the community
> and is worth checking out." 

Whereas AFL is based on a single idea of how the fuzzer does its exploration
with very few knobs,
[AFL++](https://www.usenix.org/system/files/woot20-paper-fioraldi.pdf) (2020)
keeps the basic AFL evolutionary algorithm structure, but incorporates a lot of
new research on other ways to explore the state space. For example, which seed
gets scheduled and how many times it gets mutated per round are [two new
parameters](https://mboehme.github.io/paper/CCS16.pdf) that can be tweaked to
achieve different paths of exploration throughout the system under test.

The next thing I did was to search for "coverage-guided property-based
testing" in the academic literature.

One of the first papers I found was [*Coverage guided, property based
testing*](https://dl.acm.org/doi/10.1145/3360607) by Leonidas Lampropoulos,
Michael Hicks, Benjamin C. Pierce (2019). In this paper FuzzChick, Coq/Rocq
library, that adds AFL-style coverage instrumentation to QuickChick (a Rocq
QuickCheck clone) is presented. Unfortunately the only source code I could find
lives in an [unmaintained
branch](https://github.com/QuickChick/QuickChick/compare/master...FuzzChick)
that [doesn't compile](https://github.com/QuickChick/QuickChick/issues/277).

The related works section of the paper has a couple of interesting
references though.

The main inspiration for FuzzChick seems to have been Stephen Dolan et al's
OCaml library called
[Crowbar](https://github.com/ocaml/ocaml.org-media/blob/086fc25105cbccb188c28ec74126d72962921ff8/meetings/ocaml/2017/extended-abstract__2017__stephen-dolan_mindy-preston__testing-with-crowbar.pdf)
(2017). Crowbar uses a stream of bytes to drive its generators, similar to
Hypothesis, and it's this stream that AFL is hooked up to. This indirection is
Crowbar's (and by extension, I guess, also HypoFuzz's) biggest weakness. 

AFL is good at manipulating this byte stream, but because the bytes are
not used directly to test the system under test, but rather to generate
data which in turn is used for testing, some of its effectiveness is
lost. This becomes particularly obvious when data structures with sparse
pre-conditions, e.g. sorted list or a binary search tree. That's what the
authors of FuzzChick say at least, while claiming that they addressed this
weakness by doing type-aware mutations.

The other libraries that the paper mentions are from the imperative
language community. 

For example [*JQF + Zest: Coverage-guided semantic fuzzing for
Java*](https://github.com/rohanpadhye/jqf), 
[libfuzzer](https://llvm.org/docs/LibFuzzer.html) and it's successor
[FuzzTest](https://github.com/google/fuzztest) (2022?) for C++.

Rust's `cargo fuzz` seems to build upon libfuzzer, see the chapter on
[*Structure-aware fuzzing using libfuzzer-sys in
Rust*](https://rust-fuzz.github.io/book/cargo-fuzz/structure-aware-fuzzing.html)
in the Rust Fuzz Book.

The FuzzTest README claims "It is a first-of-its-kind tool that bridges
the gap between fuzzing and property-based testing". I can't tell why
they would claim that, given that it appears to have been released in
2022 and many of the tools we looked at above seem to have successfully
combined the two approaches before that. For example, how is it
different from Go-fuzz?

In my search I also found the paper [*MUTAGEN: Reliable Coverage-Guided,
Property-Based Testing using Exhaustive
Mutations*](https://www.mista.me/assets/pdf/icst23-preprint.pdf) by Agustín
Mista and Alejandro Russo (2023). This paper seems to build upon the FuzzChick
paper, however it swaps out the AFL-style coverage instrumentation for the use
of a GHC
[plugin](https://github.com/OctopiChalmers/mutagen/blob/main/src/Test/Mutagen/Tracer/Plugin.hs)
to annotate source code with coverage information of: function clauses, case
statements, and each branch of if-then-else expressions.

Imperative languages such as Go, Python, C++, Rust, and Java seem ahead of
functional languages when it comes to combining coverage-guided fuzzing and
property-based testing.

Let's try to change that by implementing a small functional programming
version, based on the original property-based testing implementation.

## Prototype implementation

### Getting the coverage information

One key question we need to answer in order to be able to implement anything
that's coverage-guided is: where do we get the coverage information from?

When I've been thinking about how to implement coverage-guided property-based
testing in the past, I always got stuck thinking that parsing the coverage
output from the compiler in between test case generation rounds would be
annoying and slow. I thought that in the best case scenario the compiler might
provide a library which exposes the coverage information[^3].

When I started researching this post that I realised that AFL, and most
coverage-guided fuzzers since, don't use the compiler's coverage statistics but
actually inject custom coverage capturing code into compiled programs at every
branch point or [basic block](https://en.wikipedia.org/wiki/Basic_block). It's
explained in more detail in the
[whitepaper](https://lcamtuf.coredump.cx/afl/technical_details.txt). The main
reason they do it is because of performance, not because it's necessarily
easier than grabbing it from the compiler, so this didn't really help me.

It wasn't until I read about Antithesis' ["sometimes
assertions"](https://antithesis.com/docs/best_practices/sometimes_assertions.html)
that I started seeing a simple solution to the problem of collecting coverage
information.

To understand how "sometimes assertions" work let's first recall how regular
assertions, or "always assertions", work. If we add `assert b "message"`
somewhere in the code base and the boolean `b` evaluates to false at run-time
then the program will fail with `"message"`. 

"Sometimes assertions" are different in that they don't need to always hold.
Consider the example:

```
for (1..10000) {
  c := flipCoin()
  sometimesAssert (c == Heads) "probably an unfair coin"
}
```

Above the "sometimes assertion" will only fail if we flip 10000 tails.

How is this related to coverage though? If we sprinkle "sometimes assertions"
at every branch point:

```
if b {
  sometimesAssert True "true branch"
} else {
  sometimesAssert True "false branch"
}
```

Then we'll get a failure when some branch hasn't been covered! The neat thing
about "sometimes assertions" is that we don't need to annotate every single
branch, we can annotate *interesting* points in our program, that's why we can
think of "sometimes assertions" as generalised coverage.

The final piece of the puzzle, and I think this is the only original idea that
this post adds[^4], is that property-based testing already has functionality for
implementing "sometimes assertions": the machinery for gathering run-time
statistics of the generated data!

This machinery is [crucial](https://www.youtube.com/watch?v=NcJOiQlzlXQ) for
writing good tests and has been part of the QuickCheck implementation since the
very first version[^5].

So the question is: can we implement coverage-guided property-based testing
using the internal notion of coverage that property-based testing already has?

### The first version of QuickCheck

Before we answer the above question, let's remind ourselves of how a
property-based testing library is implemented. For the sake of
self-containment, let's reproduce the essential parts of QuickCheck as defined
in the appendix of the original
[paper](https://dl.acm.org/doi/10.1145/351240.351266) that first introduced
property-based testing (ICFP, 2000)[^6].

#### Generating input data

Let's start with the generator[^7], which is used to generate random inputs to
the software under test:

``` {.haskell include=src/QuickCheckV1.hs snippet=Gen}
```

So a `Gen a` is basically a function from a size and a pseudo-random number
generator into `a`. The pseudo-random number generator and size can be
accessed using the following two functions:

``` {.haskell include=src/QuickCheckV1.hs snippet=rand}
```

Using these together with the `Functor`, `Applicative` and `Monad`
instances of `Gen` we can derive other useful combinators for generating
data:

``` {.haskell include=src/QuickCheckV1.hs snippet=derivedCombinators}
```

Instead of defining generators directly for different datatypes,
QuickCheck first wraps generators in a type class called
`Arbitrary`[^8]:

``` {.haskell include=src/QuickCheckV1.hs snippet=Arbitrary}
```

#### Specifying properties

Next up, let's look at how properties are expressed. The `Property` type
is a wrapper around `Gen Result`:

``` {.haskell include=src/QuickCheckV1.hs snippet=Property}
```

Where `Result` is defined as follows:

``` {.haskell include=src/QuickCheckV1.hs snippet=Result}
```

The idea being that `ok :: Maybe Bool` is `Nothing` if the input gets
discarded and otherwise the boolean indicates whether the property
passed or not. The `stamp` field is used to collect statistics about the
generated test cases, while `arguments` contains all the generated
inputs (or arguments) to the property.

In order to allow the user to write properties of varying arity another
type class is introduced:

``` {.haskell include=src/QuickCheckV1.hs snippet=Testable}
```

The key ingredient in the function instance of `Testable` is `forAll`:

``` {.haskell include=src/QuickCheckV1.hs snippet=forAll}
```

Which in turn depends on:

``` {.haskell include=src/QuickCheckV1.hs snippet=evaluate}
```

One last construct for writing properties that we need is the ability to
add assumptions or pre-conditions about the input:

``` {.haskell include=src/QuickCheckV1.hs snippet=assuming}
```

Notice how if the input doesn't pass this test, then it will be
discarded.

#### Collecting statistics

The way we collect statistics about the generated data is through these
two functions:


``` {.haskell include=src/QuickCheckV1.hs snippet=classify}
```

#### Running the tests

Finally we have all the pieces we need to be able to actually run the
tests. The testing can be configured:

``` {.haskell include=src/QuickCheckV1.hs snippet=Config}
```

Where `maxTest` is the amount of passing test cases that will be run,
`maxFail` is the amount of tests that are allowed to be discarded,
`size` is how the size parameter to the generator changes between tests,
and `every` is used to print something (or not) between each test.

The tests themselves can now be run as follows:

``` {.haskell include=src/QuickCheckV1.hs snippet=quickCheck}
```

### Example: traditional use of coverage

Okay, so the above is the first version of the original property-based testing
tool, QuickCheck. 

Before we extend it with coverage-guidance, let's have a look at an example
property and how collecting statistics is typically used to ensure good
coverage.

The example we'll have a look at is `insert`ing into an already sorted
list (from which insertion sort can be implemented):

``` {.haskell include=src/QuickCheckV1.hs snippet=insert}
```

If we do so, then we resulting list should remain sorted:

``` {.haskell include=src/QuickCheckV1.hs snippet=prop_insert1}
```

This test passes:

```
>>> quickCheck prop_insert
OK, passed 100 tests.
```

What do the test cases that are generated look like? This is where
`classify` comes in:

``` {.haskell include=src/QuickCheckV1.hs snippet=prop_insert2}
```

Running this property, we get some statistics about the generated data:

```
>>> quickCheck prop_insert'
OK, passed 100 tests.
54% empty.
27% singleton.
19% short.
```

As we can see, all of the lists that get generated are less than 3
elements long! This is perhaps not what we expected. However if we
consider that precondition says that the list must be sorted, then it
should become clear that it's unlikely to generate such longer such
lists completely by random[^9].

### The extension to add coverage-guidance

Now let's add coverage-guidance to it using the machinery for
collecting statistics about the generated data.

The function that checks a property with coverage-guidance slight different
from `quickCheck`[^10]:

``` {.haskell include=src/QuickCheckV1.hs snippet=coverCheck}
```

In particular notice that instead of `Testable a` we use an explicit predicate
on a list of `a`, `[a] -> Property`. The reason for using a list in the
predicate is so that we can iteratively make progress, using the coverage
information. We see this more clearly if we look at the coverage-guided
analogue of the `tests` function, in particular the `xs` parameter:

``` {.haskell include=src/QuickCheckV1.hs snippet=testsC1}
```

The other important difference is the `cov`erage parameter, which keeps track
of how many things have been `classify`ed (the `stamps` parameter). Notice how
we only add the newly generated input, `x`, if the `cov`erage increases.

### Example: using coverage to guide generation

We now have all the pieces to test the example from the
[motivation](#motivation) section:

``` {.haskell include=src/QuickCheckV1.hs snippet=bad}
```

This property basically says that there's no string that's equal to `"bad!"`,
which is obviously false. If we try to test this property using the unmodified
first version of QuickCheck:

``` {.haskell include=src/QuickCheckV1.hs snippet=testBad1}
```

We'll see spin away, but not actually find the bad string:

```
>>> testBad
32301
^CInterrupted.
```

I stopped it after about 32k tries. Recall from the motivation section that the
probability of generating the bad string is about as likely as winning the
lottery (of course we can run tests faster than play the lottery, but still).

Whereas if we use coverage-guided generation:

``` {.haskell include=src/QuickCheckV1.hs snippet=testBad2}
```

We find the bad string pretty quickly. I'm using verbose output here so you can
see how it first find the `"b"`, then `"ba"`, etc[^11]:

```
>>> testBad'
0: "n"          23: "#"         46: "bai"       69: "badb"
1: "T"          24: "T"         47: "ba}"       70: "bad~"
2: "L"          25: "T"         48: "bay"       71: "bada"
3: "|"          26: "_"         49: "bac"       72: "bad%"
4: "X"          27: "@"         50: "bak"       73: "bad9"
5: "\""         28: "}"         51: "ba`"       74: "badE"
6: "e"          29: "y"         52: "bad"       75: "bad8"
7: "G"          30: "-"         53: "bad8"      76: "bad{"
8: "R"          31: "s"         54: "bade"      77: "badS"
9: "}"          32: "b"         55: "bad0"      78: "badn"
10: "C"         33: "bx"        56: "badA"      79: "bad?"
11: "3"         34: "b9"        57: "badP"      80: "badn"
12: ">"         35: "bf"        58: "badQ"      81: "badq"
13: "C"         36: "bl"        59: "bad0"      82: "bady"
14: "J"         37: "ba"        60: "bade"      83: "badA"
15: "9"         38: "baC"       61: "bad)"      84: "bad4"
16: "="         39: "baX"       62: "bado"      85: "bad;"
17: "9"         40: "baA"       63: "badE"      86: "bad9"
18: "L"         41: "baE"       64: "bad\""     87: "badU"
19: ")"         42: "bay"       65: "bad@"      88: "bad!"
20: "5"         43: "baX"       66: "bad{"
21: "6"         44: "ba6"       67: "badX"
22: "x"         45: "ba@"       68: "bado"
Falsifiable, after 88 tests:
"bad!"
```

The full source code is available
[here](https://github.com/stevana/coverage-guided-pbt/blob/main/src/QuickCheckV1.hs).

## Conclusion and further work

We've seen how to add converage-guidance to the first version of the first
property-based testing tool, QuickCheck, in about 35 lines of code.

Coverage-guidance effectively reduced a exponential problem into a polynomial
one, by building on previous test runs' successes in increasing the coverage.

The solution does change the QuickCheck API slightly by requiring a property on
a list of `a`, rather than merely `a`, so it's not suitable for all properties.
I think this limitation isn't so important, because going further I'd like to
apply coverage-guidance to testing stateful systems. When testing stateful
systems, which I've written about
[here](https://stevana.github.io/the_sad_state_of_property-based_testing_libraries.html),
one always generates a list of commands anyway, so the limitation doesn't matter.

A more serious limitation with the current approach is that it's too greedy and
will seek to maximise coverage, without ever backtracking. This means that it
can easily get stuck in local maxima. Consider the example:

```
if input[0] == 'o'
  if input[1] == 'k'
    return
if input[0] == 'b'
  if input[1] == 'a'
    if input[2] == 'd'
      error
```

If we generate an input that starts with 'o' (rather than 'b'), then we'll get
stuck never finding the error. Real coverage-guided tools, like AFL, will not
get stuck like that. While I have a variant of the code that can cope with
this, I chose to present the above greedy version because it's simpler. 

Another improvement would be to allow for sometimes assertions to be present in
the system under test, completely separate from the property. Potentially even
in a completely different programming language than that of which the property
is implemented in. This would be closer to what Anithesis does, however it
requires some communication between the system under test and the test executor
which complicates things.

Thanks for making all the way to the end of the post. I hope I've managed to
convince you that adding coverage-guidence to property-based testing doesn't
have to be that complicated, while at the same time it can make testing more
effective. I'd also like to encourage you to try to port these ideas to your
favorite language and experiment!

Lastly, if you got any questions, comments, or would like to work with me on
this topic, feel free to get in [touch](https://stevana.github.io/about.html)!


[^1]: This example is due to Dmitry Vyukov, the main author of
    [go-fuzz](https://github.com/dvyukov/go-fuzz), but it's basically an easier
    to understand version of the example from Dan Luu's post. 

[^2]: For more details about how it works, see the [AFL
    "whitepaper"](https://lcamtuf.coredump.cx/afl/technical_details.txt) and
    its [mutation
    heuristics](https://lcamtuf.blogspot.com/2014/08/binary-fuzzing-strategies-what-works.html).

[^3]: In the case of Haskell, I didn't know that there was such a
    library until read Shae "shapr" Erisson's post [*Run property tests
    until coverage stops
    increasing*](https://shapr.github.io/posts/2023-07-30-goldilocks-property-tests.html)
    (2023). Note that Shae's post only uses coverage as a stopping
    condition, not to actually drive the generation of test cases.

[^4]: As I was writing up, I stumbled across the paper [*Ijon: Exploring Deep
    State Spaces via Fuzzing*](https://ieeexplore.ieee.org/document/9152719)
    (2020) which lets the user to add custom coverage annotations. HypoFuzz
    also has this
    [functionality](https://hypofuzz.com/docs/configuration.html#custom-coverage-events).

[^5]: See the appendix of the original
    [paper](https://dl.acm.org/doi/10.1145/351240.351266) that first introduced
    property-based testing. It's interesting to note that the collecting
    statistics functionality is older than shrinking.

[^6]: I gave a [talk](https://bobkonf.de/2025/stevana.html) at BOBKonf 2025
    where I sketched an even simpler implementation of coverage-guided
    property-based testing, however that code is incomplete and doesn't run.

[^7]: We'll not talk about the `coarbitrary` method of the `Arbitrary` type
    class, which is used to generate functions, in this post. 

[^8]: The reason for wrapping `Gen` in the `Arbitrary` type class is so
    that generators don't have to be passed explicitly. Not everyone
    agrees that this is a good idea, as type class instances cannot be
    managed by the module system.

[^9]: The standard workaround here is to introduce a wrapper type for
    which we write a custom generator which generates a random list and
    then sorts it before returning. That way no pre-condition is
    needed, as the input will be sorted by construction so to say.

[^10]: It might be interesting to note that we can implement this signature
    using the original combinators:
    ``` {.haskell include=src/QuickCheckV1.hs snippet=testsC2}
    ```

[^11]: To save vertical space I've also arranged the output in columns rather
    than one per line.
