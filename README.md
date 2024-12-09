# Rushakoff Go style guide

## Preface

This document details the Go style that has served me well in a decade or more of using the language.
"Style" here is not limited to formatting and organization of source code;
it also includes general patterns for producing maintainable and testable code.

"Maintainable code" is certainly subjective,
I expect to be able to jump back in after six months of not touching a codebase with these patterns,
and my only fights should be with reacquainting with the business logic and concepts.
I should not have to struggle with navigating the codebase, tracing through overcomplicated type hierarchies,
or backfilling tests around the area of code that needs to be modified.

I have used these patterns in both small and large codebases,
from solo projects to projects with dozens of contributors.

This is a living document, so it is possible that future revisions
may retract or contradict earlier details.
In other words, little or none of this is set in stone;
break the rules if it makes sense to do so.

If otherwise in doubt about a style rule here,
I tend to use the Go standard library as a point of reference.

Lastly, the langauge is named Go.
The word "golang" is meant for use in internet searches.

## Comment style

### Doc comments

The official [Go Doc Comments](https://go.dev/doc/comment) document is the source of truth on comments.

The key point that I see most frequently violated is:

> Go doc comments use complete sentences.

Complete sentences include punctuation.
Doc comments are expected to read correctly when rendered on [pkg.go.dev](https://pkg.go.dev).

The Comments doc mentions [semantic linefeeds](https://rhodesmill.org/brandon/2012/one-sentence-per-line/),
which I treat as a hard requirement in any code that I author.
Ultimately, semantic linefeeds produce highly readable diffs,
and reading commit history is something I do frequently.
Semantic linefeeds also happen to be usually easily readable without any special editor settings.

The rule in Go doc comments is that the first sentence begins with the name of the function, method, or type being documented.
Another common mistake I notice is applying that rule to fields within a struct,
or methods on an interface declaration.
For those cases, it is not necessary to repeat the field name or interface method,
but feel free to use them if it helps readability.

#### Package comments

I expect every package to include a package comment.
My convention is to always put the package comment in a file named `doc.go`.
While it may be tempting to assume up front that a particular package
will only ever have the one source file,
sooner or later many packages get multiple files.

Keeping the package comment in a consistent `doc.go` file saves the effort
of searching for it when it needs to be edited.

## Package layout

First, follow the conventions in the [Go Blog post on package names](https://go.dev/blog/package-names).

Be sure that package names are all lowercase, with no underscores.

One of the most annoying things I run into when consuming third party libraries is duplicate package names.
If you are developing a standalone application, it's less of an issue to have one or two files
that have to disambiguate your `foo/http` package and the standard library's `net/http` package;
but do not export a package named `http` from your library that is likely to be imported alongside
other uses of `net/http`, requiring every import to be renamed.

For that matter, do not have packages `foo/bar` and `quux/bar` that are frequently imported together
and have to be renamed on import.
In that case, I would name them `foo/foobar` and `quux/quuxbar` to distinguish them as children
of the respective packages.

<!-- TODO: maybe mention upward/downward dependence? -->

### Internal packages

Make judicious use of [`internal` directories](https://pkg.go.dev/cmd/go#hdr-Internal_Directories).
This allows you to have exported types, functions, and methods that can still be tested
without reaching into their unexported code,
but prevents an external user from using that code directly.

On a very small project I might put code directly in an `internal` package.
As the size of the project grows, I am more likely to use `internal/foo` to simplify reading;
I do not want to have to guess which `internal` package is being referenced when I see `internal.Foo`.

### Test packages

If your project has a case for some shared test behavior --
for example if you have tests for the compliance of expected behavior on implementations of an interface,
or if you have test fixtures that make sense to share across related tests --
it makes sense to pull that into a separate test related package.

Nest the new package under the source package, and name it to match the source package with a `test` suffix.
The standard library does this with [`net/http/httptest`](https://pkg.go.dev/net/http/httptest).

## Concurrent Go code

### Mutexes

In a struct declaration, organize the fields such that the mutex
is the first entry in a "paragraph" around the fields it accesses;
unless the mutex protects all fields, in which case it may be the first, or one of the first, fields.

The mutex is conventionally named `mu`.
If there are multiple mutexes, which is uncommon but sometimes useful,
suffix the field names like `fooMu` and `barMu`.

Do not embed the mutex.
The mutex should almost never be used directly by the caller.

I don't mind using mutexes in narrow cases.
For instance, a map shared by multiple goroutines,
where any access is going to be one read and maybe a write,
is a great fit for a mutex.
But if acquiring the lock is followed by more complicated work, such as
reaching into the filesystem and iterating through files of unpredictably large size,
then I tend to lean more towards dedicating a goroutine to the work.

### Context

I use [context](https://pkg.go.dev/context) values liberally,
primarily for cancellation.

This means when I author types that start background goroutines,
the type accepts a context value.
We rely on that context value to cleanly shut down any goroutines:
every select statement has a `ctx.Done()` case (or a `default` case so the select won't block).

The root of the application typically uses [`os/signal.NotifyContext`](https://pkg.go.dev/os/signal#NotifyContext)
so that an interrupt signal propagates as a canceled context through all live goroutines.
Or in any tests involving types with background work,
we use `context.WithCancel(context.Background())` and defer the cancel,
ensuring that resources are cleaned up at the end of the test.
This also has the benefit of indirectly asserting that clean shutdowns are possible;
if a shutdown hangs, the test hangs too.

It is somewhat common to see a pattern of a type that does not accept a context,
and has a separate `Stop` (or `Quit` or `Shutdown`) method to end its work.
We rely on context to interrupt the work anyway,
so instead we first cancel the context,
and we use a `Wait` method that blocks until all background work is complete.

If a type starts only one background goroutine,
a plain empty struct channel suffices to indicate a complete shutdown.
Otherwise, a [`sync.WaitGroup`](https://pkg.go.dev/sync#WaitGroup) is appropriate.

## Tests

### Parallel tests

By default, all Go tests run in serial.
It is reasonable for the language to choose that as a pessimistic default.
But, if you are writing code that does not rely on global state,
you ought to call `t.Parallel` in every possible test.
This maximizes CPU core usage and results in a very fast test suite.
Developers are much more likely to run the full set of tests when the tests are fast.

### Data race detector

Make a habit of running your tests with the [race detector](https://go.dev/doc/articles/race_detector) enabled,
i.e. `go test -race`.

In working on distributed systems in particular,
aim to have some set of tests such that you run multiple in-process instances of your service.
In this configuration with the race detector enabled,
you will discover real data races that can occur in production --
or sometimes you will at least discover global state that you may not have already known about.
(Of course, testing this way is not exhaustive,
and if you can afford to run an instance of your service with the race detector enabled,
that is worth doing to discover other data model violations.)

### Testing frameworks

The standard library's [`testing`](https://pkg.go.dev/testing) package is perfectly fine.
But, for a larger project with many contributors of various skill levels,
a testing package like [`testify`](https://pkg.go.dev/github.com/stretchr/testify)
simplifies testing by introducing consistency in failure messages,
and it has a reasonable amount of helpers for various common comparisons.

I tend to use testify in anything but the smallest projects,
because I think it is one of the most likely testing packages
that other contributors may have used before.
But, I don't think it is wrong to use a different third party testing package.

### Test packages

For a package `foo` with a file `bar.go`, you would typically put your tests in `bar_test.go`.

The [`testing` package documentation](https://pkg.go.dev/testing) clarifies:

> The test file can be in the same package as the one being tested, or in a corresponding package with the suffix "`_test`".
> If the test file is in the same package, it may refer to unexported identifiers within the package...
> If the file is in a separate "`_test`" package, the package being tested must be imported explicitly and only its exported identifiers may be used.

Conventional test-driven development wisdom recommends the latter --
tests always go in the `_test`-suffixed package so that we are only testing the exported interface.

If I find myself in the extremely rare situation where I need to test something that must not otherwise be exported,
I will often name that file with the suffix `_internal_test.go` just to make it more clear that
it is reaching into internals that we would typically not otherwise touch.

On the other hand, if I have several unexported types with nontrivial behavior,
and those types are complicated enough to warrant their own tests,
at that point I will likely move the types to an `internal` package
so that I can use them and test them as exported types,
without letting other consumers access them.

## Errors

My general point of view on errors in Go is that the author of the code
must assist the users in understanding why an error occurred and how to resolve it.

### Strong error types

`fmt.Errorf` is often sufficient for a general indication of a failed operation.
Suppose your function had to validate user input:
something like `fmt.Errorf("foo values must be 6 alphanumeric characters; got %q", input)`
tells the user exactly what was wrong.
(Note also that we use `%q` here which will put quotes around the input,
so an empty input will be an obvious empty string instead of a less obvious "did we get the whole error message?")

But, if the caller may need to react to a specific error, prefer to use either:
- a package-level `ErrFoo` error value typically constructed with `errors.New`,
  if there is no contextual data to add -- for example,
  `var ErrServerOverloaded = errors.New("server overloaded")
- or a type satisyfing the `error` interface, if there is contextual information that can be extracted

Callers that care about a specific error can use
[`errors.As`](https://pkg.go.dev/errors#As) to check if an error matches a particular type,
or [`errors.Is`](https://pkg.go.dev/errors#Is) to check if an error matches a particular value.

### Wrapping

Simply doing `_, err := somethingThatMayFail(); return err` is certainly more convenient to write,
but when you have many calls in a stack that all propagate that one error,
debugging becomes much more challenging.
It might be easy to find that a function `foo` originated a particular error message,
but when `foo` is called from dozens of other functions,
the user may have to do a wide search to figure out which invocation is causing the operation to fail.

Therefore, by default, I always wrap errors,
e.g. `return fmt.Errorf("failed to persist foobar message: %w", err)`.

Using this pattern may result in some kind of long error message, like
`failed to write output: failed to persist foobar message: file /tmp/foobar.messages exceeds configured limit of 1024 bytes`;
but all of that information gives a user fully sufficient information
to identify most of, if not all of, the entire failing call stack.
A programmer armed with that information does not need to waste time
searching for where an error occurred.

When applying this wrapping pattern, I try to avoid repeating the wrapped error messages.
In terms of assisting other programmers,
a repeated error message is better than no error message,
but unique error messages are easily discovered.

### Joined errors

[`errors.Join`](https://pkg.go.dev/errors#Join) is a relatively new feature,
being introduced in go 1.20,
so not everyone is aware of its existence.

Joined errors are very useful when multiple things can go wrong at once,
and we can help the user by telling them what to fix all in one pass.
It is very frustrating as a consumer to get a long sequence of "fix this one thing",
look up how to fix it, run again, and get another "fix this one other thing."
It is much more user friendly to get one error message saying to fix foo, bar, and baz.

I most often use joined errors when there are multiple discrete input values to validate
(such as in a configuration file),
or when there are multiple required options in an options pattern.

<!-- TODO:
- prefer single long lived goroutine
- prefer non-anonymous goroutines
- supported versions of Go
-->
