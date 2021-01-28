# tasty-bench

Featherlight benchmark framework (only one file!) for performance measurement with API mimicking [`criterion`](http://hackage.haskell.org/package/criterion) and [`gauge`](http://hackage.haskell.org/package/gauge).

## How lightweight is it?

There is only one source file `Test.Tasty.Bench` and no external dependencies
except [`tasty`](http://hackage.haskell.org/package/tasty).
So if you already depend on `tasty` for a test suite, there
is nothing else to install.

Compare this to `criterion` (10+ modules, 50+ dependencies) and `gauge` (40+ modules, depends on `basement` and `vector`).

## How is it possible?

Our benchmarks are literally regular `tasty` tests, so we can leverage all existing
machinery for command-line options, resource management, structuring,
listing and filtering benchmarks, running and reporting results. It also means
that `tasty-bench` can be used in conjunction with other `tasty` ingredients.

Unlike `criterion` and `gauge` we use a very simple statistical model described below.
This is arguably a questionable choice, but it works pretty well in practice.
A rare developer is sufficiently well-versed in probability theory
to make sense and use of all numbers generated by `criterion`.

## How to switch?

[Cabal mixins](https://cabal.readthedocs.io/en/3.4/cabal-package.html#pkg-field-mixins)
allow to taste `tasty-bench` instead of `criterion` or `gauge`
without changing a single line of code:

```cabal
cabal-version: 2.0

benchmark foo
  ...
  build-depends:
    tasty-bench
  mixins:
    tasty-bench (Test.Tasty.Bench as Criterion)
```

This works vice versa as well: if you use `tasty-bench`, but at some point
need a more comprehensive statistical analysis,
it is easy to switch temporarily back to `criterion`.

## How to write a benchmark?

```haskell
import Test.Tasty.Bench

fibo :: Int -> Integer
fibo n = if n < 2 then toInteger n else fibo (n - 1) + fibo (n - 2)

main :: IO ()
main = defaultMain
  [ bgroup "fibonacci numbers"
    [ bench "fifth"     $ nf fibo  5
    , bench "tenth"     $ nf fibo 10
    , bench "twentieth" $ nf fibo 20
    ]
  ]
```

Since `tasty-bench` provides an API compatible with `criterion`,
one can refer to [its documentation](http://www.serpentine.com/criterion/tutorial.html#how-to-write-a-benchmark-suite) for more examples.

## How to read results?

Running the example above results in the following output:

```
All
  fibonacci numbers
    fifth:     OK (2.13s)
       63 ns ± 3.4 ns
    tenth:     OK (1.71s)
      809 ns ±  73 ns
    twentieth: OK (3.39s)
      104 μs ± 4.9 μs

All 3 tests passed (7.25s)
```

The output says that, for instance, the first benchmark
was repeatedly executed for 2.13 seconds (wall time),
its mean time was 63 nanoseconds and,
assuming ideal precision of a system clock,
execution time does not often diverge from the mean
further than ±3.4 nanoseconds
(double standard deviation, which for normal distributions
corresponds to [95%](https://en.wikipedia.org/wiki/68%E2%80%9395%E2%80%9399.7_rule)
probability). Take standard deviation numbers
with a grain of salt; there are lies, damned lies, and statistics.

Note that this data is not directly comparable with `criterion` output:

```
benchmarking fibonacci numbers/fifth
time                 62.78 ns   (61.99 ns .. 63.41 ns)
                     0.999 R²   (0.999 R² .. 1.000 R²)
mean                 62.39 ns   (61.93 ns .. 62.94 ns)
std dev              1.753 ns   (1.427 ns .. 2.258 ns)
```

One might interpret the second line as saying that
95% of measurements fell into 61.99–63.41 ns interval, but this is wrong.
It states that the [OLS regression](https://en.wikipedia.org/wiki/Ordinary_least_squares)
of execution time (which is not exactly the mean time) is most probably
somewhere between 61.99 ns and 63.41 ns,
but does not say a thing about individual measurements.
To understand how far away a typical measurement deviates
you need to add/subtract double standard deviation yourself
(which gives 62.78 ns ± 3.506 ns, similar to `tasty-bench` above).

To add to the confusion, `gauge` in `--small` mode outputs
not the second line of `criterion` report as one might expect,
but a mean value from the penultimate line and a standard deviation:

```
fibonacci numbers/fifth                  mean 62.39 ns  ( +- 1.753 ns  )
```

The interval ±1.753 ns answers
for [68%](https://en.wikipedia.org/wiki/68%E2%80%9395%E2%80%9399.7_rule)
of samples only, double it to estimate the behaviour in 95% of cases.

## Statistical model

Here is a procedure used by `tasty-bench` to measure execution time:

1. Set _n_ ← 1.
2. Measure execution time _tₙ_ of _n_ iterations
   and execution time _t₂ₙ_ of _2n_ iterations.
3. Find _t_ which minimizes deviation of (_nt_, _2nt_) from (_tₙ_, _t₂ₙ_).
4. If deviation is small enough, return _t_ as a mean execution time.
5. Otherwise set _n_ ← _2n_ and jump back to Step 2.

This is roughly similar to the linear regression approach which `criterion` takes,
but we fit only two last points. This allows us to simplify away all heavy-weight
statistical analysis. More importantly, earlier measurements,
which are presumably shorter and noisier, do not affect overall result.
This is in contrast to `criterion`, which fits all measurements and
is biased to use more data points corresponding to shorter runs
(it employs _n_ ← _1.05n_ progression).

An alert reader could object that we measure standard deviation
for samples with _n_ and _2n_ iterations, but report
it scaled to a single iteration.
Strictly speaking, this is justified only if we assume
that deviating factors are either roughly periodic
(e. g., coarseness of a system clock, garbage collection)
or are likely to affect several successive iterations in the same way
(e. g., slow down by another concurrent process).

Obligatory disclaimer: statistics is a tricky matter, there is no
one-size-fits-all approach.
In the absense of a good theory
simplistic approaches are as (un)sound as obscure ones.
Those who seek statistical soundness should rather collect raw data
and process it themselves in R/Python. Data reported by `tasty-bench`
is only of indicative and comparative significance.

## Tip

Passing `+RTS -T` (via `cabal bench --benchmark-options '+RTS -T'`
or `stack bench --ba '+RTS -T'`) enables `tasty-bench` to estimate and report
memory usage such as allocated and copied bytes.

## Command-line options

Use `--help` to list command-line options.

* `-p`, `--pattern`

  This is a standard `tasty` option, which allows filtering benchmarks
  by a pattern or `awk` expression. Please refer to
  [`tasty` documentation](https://github.com/feuerbach/tasty#patterns)
  for details.

* `--csv`

  File to write results in CSV format. If specified, suppresses console output.

* `-t`, `--timeout`

  This is a standard `tasty` option, setting timeout for individual benchmarks
  in seconds. Use it when benchmarks tend to take too long: `tasty-bench` will make
  an effort to report results (even if of subpar quality) before timeout. Setting
  timeout too tight (insufficient for at least three iterations of benchmark)
  will result in a benchmark failure. Do not use `--timeout` without a reason:
  it forks an additional thread and thus affects reliability of measurements.

* `--stdev`

  Target relative standard deviation of measurements in percents (5% by default).
  Large values correspond to fast and loose benchmarks, and small ones to long and precise.
  If it takes far too long, consider setting `--timeout`,
  which will interrupt benchmarks, potentially before reaching the target deviation.
