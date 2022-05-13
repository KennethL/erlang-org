---
layout: post
title: Erlang/OTP 25 Highlights
tags: erlang otp 25 release
author: Kenneth Lundin
---

OTP 25 is finally here. This post will introduce the new features that I am most excited about.

Erlang/OTP 25 includes contributions from ??+ external contributors totaling
??+ commits, ??+ PRs.

You can download the readme describing all the changes here: [Erlang/OTP 25 Readme](/patches/otp-25.0).
Or, as always, look at the release notes of the application you are interested in.
For instance here: [Erlang/OTP 25 - Erts Release Notes - Version 13.0](https://www.erlang.org/doc/apps/erts/notes.html#erts-13.0).

This years highlights are:

* [New functions in the `maps`and `lists` modules](#new-functions-in-the-maps-and-lists-modules)
* [Selectable features and the new `maybe_expr` feature](#selectable-features-and-the-new-maybe_expr-feature)
* [Dialyzer](#dialyzer)
* [Improvements of the JIT](#improvements-of-the-jit)
* [ETS-tables with adaptive support for write concurrency](#ets-tables-with-adaptive-support-for-write-concurrency)
* [Relocatable installation directory](#relocatable-installation-directory)
* [New option `short` for `erlang:float_to_list/2` and `erlang:float_to_binary/2`](#new-option-short-for-erlangfloattolist2-and-erlangfloattobinary2)
* [The new module `peer` supersedes the slave module](#the-new-module-peer-supersedes-the-slave-module)
* [`gen_xxx` modules has got a new `format_status/1` callback](#genxxx-modules-has-got-a-new-formatstatus1-callback)
* [The `timer` module has been modernized and made more efficient](#the-timer-module-has-been-modernized-and-made-more-efficient)
* [Crypto and OpenSSL 3.0](#crypto-and-openssl-30)
* [CA-certificates can be fetched from the OS standard place](#ca-certificates-can-be-fetched-from-the-os-standard-place)

# New functions in the `maps` and `lists` modules

Triggered by suggestions from the users we have introduced new functions in the [`maps`](/doc/man/maps.html) and [`lists`](/doc/man/lists.html) modules in `stdlib`.

## `maps:groups_from_list/2,3`

For short we can say that this function take a list of elements and group them. The result is a map `#{Group1 => [Group1Elements], GroupN => [GroupNElements]}`.

Let us look at some examples from the shell:

```erlang
> maps:groups_from_list(fun(X) -> X rem 2 end, [1,2,3]).
#{0 => [2], 1 => [1, 3]}
```

The provided fun calculates `X rem 2` for every element `X` in the input list and then group the elements in a map with the result of `X rem 2` as key and the corresponding elements as a list value for that key.

```erlang
> maps:groups_from_list(fun erlang:length/1, ["ant", "buffalo", "cat", "dingo"]).
#{3 => ["ant", "cat"], 5 => ["dingo"], 7 => ["buffalo"]}
```

In the example above the strings in the input list are grouped into a map based on their length.

There is also a variant of `groups_from_list` with an additional fun by which the values can be converted before they are put into their groups.

```erlang
> maps:groups_from_list(fun(X) -> X rem 2 end, fun(X) -> X*X end, [1,2,3]).
#{0 => [4], 1 => [1, 9]}
```

In the example above the elements `X` in the list are grouped according the `X rem 2` calculation but the values stored in the groups are the elements multiplied by themselves (`X * X`).  

```erlang
> maps:groups_from_list(fun erlang:length/1, fun lists:reverse/1, ["ant", "buffalo", "cat", "dingo"]).
#{3 => ["tna","tac"],5 => ["ognid"],7 => ["olaffub"]}
```

In the example above the strings from the input list are grouped according to their length and they are reversed before they are stored in the groups.

For more details see the [`maps:groups_from_list/2`](/doc/man/maps.html#groups_from_list-2) documentation.

## `lists:enumerate/1,2`

Takes a list of elements and returns a new list where each element has been associated with its position in the original list. Returns a new list with tuples of the form `{I, H}` where `I` is the position of `H` in the original list. The enumeration starts with 1 and increases by 1 in each step.

Example:

```erlang
> lists:enumerate([a,b,c]).
[{1,a},{2,b},{3,c}]
```

There is also a `enumerate/2` function which can be used to set the initial number to something else than 1. See example below:

```erlang
> lists:enumerate(10, [a,b,c]).
[{10,a},{11,b},{12,c}]
```

For more details see the [`lists:enumerate/1`](/doc/man/lists.html#enumerate-1) documentation.

## `lists:uniq/1,2`

Removes duplicates from a list while preserving the order of the elements. The first occurrence of each element is kept. 
We already have `lists:usort` which also removes duplicates but returns a sorted list.

Examples:

```erlang
> lists:uniq(fun([3,3,1,2,1,2,3]).
[3,1,2]
> lists:uniq([a, a, 1, b, 2, a, 3]).
[a, 1, b, 2, 3]
```

`lists:uniq/2` allows the user to specify with a fun how to determine that 2 elements in the list are equal. In the example below the provided fun is just testing the first element of the 2 tuples for equality.

Examples:

```erlang
> lists:uniq(fun({X, _}) -> X end, [{b, 2}, {a, 1}, {c, 3}, {a, 2}]).
[{b, 2}, {a, 1}, {c, 3}]
```

For more details see the [`lists:uniq/1`](/doc/man/lists.html#uniq-1) documentation.

# Selectable features and the new `maybe_expr` feature

Selectable features is a new mechanism and concept where a new potentially incompatible feature (language or runtime), can be introduced and tested without causing troubles for those that don't use it.

When it comes to language features the intention is that they can be activated per module with no impact on modules where they are not activated.

Let's use the new `maybe_expr` feature as an example.

In module `my_experiment` the feature is activated and used like this:

```erlang
-module(my_experiment).
-export([foo/1]).

%% Enable the feature maybe_expr in this module only
%% Makes maybe a keyword which might be incompatible
%% in modules using maybe as a function name or an atom
-feature(maybe_expr,enable).
foo() ->
  maybe
    {ok, X} ?= f(Foo),
    [H|T] ?= g([1,2,3]),
    ...
  else
    {error, Y} ->
        {ok, "default"};
    {ok, _Term} ->
        {error, "unexpected wrapper"}
  end.
```

The compiler will note that the feature `maybe_expr` is enabled and will handle the maybe construct correctly. In the generated `.beam` file it will also be noted that
the module has enabled the feature.

When starting an Erlang node the specific feature (or all) must be enabled otherwise the `.beam` file with the feature will not be allowed for loading.

```text
erl -enable-feature maybe_expr
```

Or 

```text
erl -enable-feature all
```

For more details see the [feature section](/doc/reference_manual/feature.html) in the Erlang Reference Manual.

## The new `maybe_expr` feature EEP-49

The [EEP-49 "Value-Based Error Handling Mechanisms"](/eeps/eep-0049), was suggested by Fred Hebert already 2018 and now it has finally been implemented as the first feature within the new feature concept.

The `maybe ... end` construct which is similar to `begin ... end` in that it is used to group multiple distinct expression as a
single block. But there is one important difference in that the
`maybe` block does not export its variables while `begin` does
export its variables.

A new type of expressions (denoted `MatchOrReturnExprs`) are introduced, which are only valid within a
`maybe ... end` expression:

```erlang
maybe
    Exprs | MatchOrReturnExprs
end
```

`MatchOrReturnExprs` are defined as having the following form:

```erlang
Pattern ?= Expr
```

This definition means that `MatchOrReturnExprs` are only allowed at the
top-level of `maybe ... end` expressions.

The `?=` operator takes the value returned by `Expr` and pattern matches
it against `Pattern`.

If the pattern matches, all variables from `Pattern` are bound in the local
environment, and the expression is equivalent to a successful `Pattern = Expr`
call. If the value does not match, the `maybe ... end` expression returns the
failed expression directly.

A special case exists in which we extend `maybe ... end` into the following form:

```erlang
maybe
    Exprs | MatchOrReturnExprs
else
    Pattern -> Exprs;
    ...
    Pattern -> Exprs
end
```

This form exists to capture non-matching expressions in a `MatchOrReturnExprs`
to handle failed matches rather than returning their value. In such a case, an
unhandled failed match will raise an `else_clause` error, otherwise identical to
a `case_clause` error.

This extended form is useful to properly identify and handle successful and
unsuccessful matches within the same construct without risking to confuse
happy and unhappy paths.

Given the structure described here, the final expression may look like:

```erlang
maybe
    Foo = bar(),            % normal exprs still allowed
    {ok, X} ?= f(Foo),
    [H|T] ?= g([1,2,3]),
    ...
else
    {error, Y} ->
        {ok, "default"};
    {ok, _Term} ->
        {error, "unexpected wrapper"}
end
```

For more details see the [maybe section](/doc/reference_manual/expressions.html#maybe) in the Erlang Reference Manual.

### Motivation

With the `maybe` construct it is possible to reduce deeply nested conditional expressions and make messy patterns found in the wild unnecessary. It also provides a better separation of concerns when implementing functions.

#### Reducing Nesting

One common pattern that can be seen in Erlang is deep nesting of `case
... end` expressions, to check complex conditionals.

Take the following code taken from
[Mnesia](https://github.com/erlang/otp/blob/a0ae44f324576104760a63fe6cf63e0ca31756fc/lib/mnesia/src/mnesia_backup.erl#L106-L126),
for example:

```erlang
commit_write(OpaqueData) ->
    B = OpaqueData,
    case disk_log:sync(B#backup.file_desc) of
        ok ->
            case disk_log:close(B#backup.file_desc) of
                ok ->
                    case file:rename(B#backup.tmp_file, B#backup.file) of
                        ok ->
                            {ok, B#backup.file};
                        {error, Reason} ->
                            {error, Reason}
                    end;
                {error, Reason} ->
                    {error, Reason}
            end;
        {error, Reason} ->
            {error, Reason}
    end.
```

The code is nested to the extent that shorter aliases must be introduced
for variables (`OpaqueData` renamed to `B`), and half of the code just
transparently returns the exact values each function was given.

By comparison, the same code could be written as follows with the new
construct:

```erlang
commit_write(OpaqueData) ->
    maybe
        ok ?= disk_log:sync(OpaqueData#backup.file_desc),
        ok ?= disk_log:close(OpaqueData#backup.file_desc),
        ok ?= file:rename(OpaqueData#backup.tmp_file, OpaqueData#backup.file),
        {ok, OpaqueData#backup.file}
    end.
```

Or, to protect against `disk_log` calls returning something else than `ok |
{error, Reason}`, the following form could be used:

```erlang
commit_write(OpaqueData) ->
    maybe
        ok ?= disk_log:sync(OpaqueData#backup.file_desc),
        ok ?= disk_log:close(OpaqueData#backup.file_desc),
        ok ?= file:rename(OpaqueData#backup.tmp_file, OpaqueData#backup.file),
        {ok, OpaqueData#backup.file}
    else
        {error, Reason} -> {error, Reason}
    end.
```

The semantics of these calls are identical, except that it is now
much easier to focus on the flow of individual operations and either
success or error paths.

# Dialyzer

* Dialyzer now supports the `missing_return` and `extra_return` options to raise warnings when specifications differ from inferred types. These are similar to, but not quite as verbose, as `overspecs` and `underspecs`.

* Dialyzer now better understands the types for `min/2`, `max/2`, and `erlang:raise/3`. Because of that, Dialyzer can potentially generate new warnings. In particular, functions that use `erlang:raise/3` could now need a spec with a `no_return()` return type to avoid an unwanted warning.

# Improvements of the JIT

The [JIT compiler][jit] introduced in Erlang/OTP 24 improved
the performance for Erlang applications.

Erlang/OTP 25 introduces some major improvements of the JIT:

* The JIT now supports the [AArch64 (ARM64)][aarch64] architecture,
used by (for example) [Apple Silicon][apple_silicon] Macs and newer
[Raspberry Pi][rpi] devices.

* Better code generated based on types provided by the Erlang compiler.

* Better support for `perf` and `gdb` with line numbers for Erlang code.

[jit]: https://www.erlang.org/blog/my-otp-24-highlights/#beamasm---the-jit-compiler-for-erlang
[aarch64]: https://en.wikipedia.org/wiki/AArch64
[apple_silicon]: https://en.wikipedia.org/wiki/Apple_silicon
[rpi]: https://en.wikipedia.org/wiki/Raspberry_Pi

### Support for AArch64 (ARM64)

How much speedup one can expect from the JIT compared to the interpreter
varies from nothing to up to four times.

To get some more concrete figures we have run three different
benchmarks with the JIT disabled and enabled on a MacBook Pro (M1
processor; released in 2020).

First we ran the [EStone benchmark][estone]. Without the JIT, 691,962
EStones were achieved and with the JIT 1,597,949 EStones. That is,
more than twice as many EStones with the JIT.

Next we tried running Dialyzer to build a small PLT:

```text
dialyzer --build_plt --apps erts kernel stdlib
```

With the JIT, the time for building the PLT was reduced from 18.38 seconds
down to 9.64 seconds. That is, almost but not quite twice as fast.

Finally, we ran a benchmark for the [base64][base64] module included
in [this Github issue][gh-5639].

With the JIT:

```text
== Testing with 1 MB ==
fun base64:encode/1: 1000 iterations in 11846 ms: 84 it/sec
fun base64:decode/1: 1000 iterations in 14617 ms: 68 it/sec
```

Without the JIT:

```text
== Testing with 1 MB ==
fun base64:encode/1: 1000 iterations in 25938 ms: 38 it/sec
fun base64:decode/1: 1000 iterations in 20603 ms: 48 it/sec
```

Encoding with the JIT is almost two and half times as fast, while the
decoding time with the JIT is about 75 percent of the decoding time
without the JIT.

[estone]: https://github.com/erlang/otp/blob/be860185407d6747dca32e8d328b041cc75ffdb3/erts/emulator/test/estone_SUITE.erl
[base64]: https://www.erlang.org/doc/man/base64.html
[gh-5639]: https://github.com/erlang/otp/issues/5639

### Type-based optimizations

The JIT translates one BEAM instruction at the time to native code
without any knowledge of previous instructions. For example, the native
code for the `+` operator must work for any operands: small integers that
fit in 64-bit word, large integers, floats, and non-numbers that should
result in raising an exception.

In Erlang/OTP 25, the compiler embeds type information in the BEAM file
to the help the JIT generate better native code without unnecessary type
tests.

For more details, see the blog post [Type-Based Optimizations in the
JIT][type-based-opts].

[type-based-opts]: https://www.erlang.org/blog/type-based-optimizations-in-the-jit

# Better support for `perf` and `gdb`

It is now possible to profile Erlang systems with perf and get a mapping from the JIT code to the corresponding Erlang code. This will make it easy to find bottlenecks in the code.

The same goes for `gdb` which also can show which line of Erlang code a specific address in the JIT code corresponds to.

Perf is a Linux command-line tool for lightweight CPU profiling; it checks CPU performance counters, trace points, uprobes, and kprobes, monitors program events, and creates reports.

An Erlang node running under `perf` can be started like this:

```text
perf record --call-graph fp -- erl +JPperf true
```

and then the result from perf could be viewed like this:

```text
perf report
```

It is also possible to attach `perf` to an already running Erlang node like this:

```text
# start Erlang at get the Pid
erl +JPperf true
```

And the pid for the node is `4711`

You can then attach `perf` to the node like this:

```text
sudo perf record --call-graph fp -p 4711
```

The result from perf could then look like this:

![alt text](/blog/images/otp25/perf_callgraph.png "perf call-graph")


Frame pointers are enabled when the `+JPperf true` option is passed, so you can
use `perf record --call-graph=fp` to get more context.

Any Erlang function in the report is prefixed with a `$` and all C functions have
their normal names. Any Erlang function that has the prefix `$global::` refers
to a global shared fragment.

So in the above, we can see that we spend the most time doing `eq`, i.e. comparing two terms.
By expanding it and looking at its parents we can see that it is the function
`erl_types:t_is_equal/2` that contributes the most to this value. Go and have a look
at it in the source code to see if you can figure out why so much time is spent there.

After `eq` we see the function `erl_types:t_has_var/1` where we spend almost
5% of the entire execution in. A while further down you can see `copy_struct_x`
which is the function used to copy terms. If we expand it to view the parents
we find that it is mostly `ets:lookup_element/3` that contributes to this time
via the Erlang function `dialyzer_plt:ets_table_lookup/2`.

### perf tips and tricks

You can do a lot of neat things with `perf`. Below is a list of some of the options
we have found useful:

* `perf report --no-children`
    Do not include the accumulation of all children in a call.
* `perf report  --call-graph callee`
    Show the callee rather than the caller when expanding a function call.
* `perf archive`
    Create an archive with all the artifacts needed to inspect the data
    on another host. In early version of perf this command does not work,
    instead you can use [this bash script](https://github.com/torvalds/linux/blob/master/tools/perf/perf-archive.sh).
* `perf report` gives "failed to process sample" and/or "failed to process type: 68"
    This probably means that you are running a buggy version of perf. We have
    seen this when running Ubuntu 18.04 with kernel version 4. If you update
    to Ubuntu 20.04 or use Ubuntu 18.04 with kernel version 5 the problem
    should go away.

# Improved error information for failing binary construction

Erlang/OTP 24 introduced [improved BIF error information][ext_bif_info] to provide
more information when a call to a BIF failed.

[ext_bif_info]: https://www.erlang.org/blog/my-otp-24-highlights/#eep-54-improved-bif-error-information

In Erlang/OTP 25, improved error information is also given when the
creation of a binary using the [bit syntax][bit_syntax] fails.

Consider this function:

```erlang
bin(A, B, C, D) ->
    <<A/float,B:4/binary,C:16,D/binary>>.
```

If we call this function with incorrect arguments in past releases
we will just be told that something was wrong and the line number:

```text
1> t:bin(<<"abc">>, 2.0, 42, <<1:7>>).
** exception error: bad argument
     in function  t:bin/4 (t.erl, line 5)
```

But which part of line 5? Imagine that `t:bin/4` was called from deep
within an application and we had no idea what the actual values for
the arguments were. It could take a while to figure out exactly what
went wrong.

Erlang/OTP 25 gives us more information:

```text
1> c(t).
{ok,t}
2> t:bin(<<"abc">>, 2.0, 42, <<1:7>>).
** exception error: construction of binary failed
     in function  t:bin/4 (t.erl, line 5)
        *** segment 1 of type 'float': expected a float or an integer but got: <<"abc">>
```

Note that the module must be compiled by the compiler in Erlang/OTP 25 in
order to get the more informative error message. The old-style message will
be shown if the module was compiled by a previous release.

Here the message tells us that first segment in the construction was given
the binary `<<"abc">>` instead of a float or an integer, which is the expected
type for a `float` segment.

It seems that we switched the first and second arguments for `bin/4`,
so we try again:

```text
3> t:bin(2.0, <<"abc">>, 42, <<1:7>>).
** exception error: construction of binary failed
     in function  t:bin/4 (t.erl, line 5)
        *** segment 2 of type 'binary': the value <<"abc">> is shorter than the size of the segment
```

It seems that there was more than one incorrect argument. In this
case, the message tells us that the given binary is shorter than the
size of the segment.

Fixing that:

```text
4> t:bin(2.0, <<"abcd">>, 42, <<1:7>>).
** exception error: construction of binary failed
     in function  t:bin/4 (t.erl, line 5)
        *** segment 4 of type 'binary': the size of the value <<1:7>> is not a multiple of the unit for the segment
```

A `binary` segment has a default unit of 8. Therefore, passing a bit string of
size 7 will fail.

Finally:

```text
5> t:bin(2.0, <<"abcd">>, 42, <<1:8>>).
<<64,0,0,0,0,0,0,0,97,98,99,100,0,42,1>>
```

[bit_syntax]: https://www.erlang.org/doc/reference_manual/expressions.html#bit-syntax-expressions
[pr5281]: https://github.com/erlang/otp/pull/5281
[eep54]: https://www.erlang.org/eeps/eep-0054.html

# Improved error information for failed record matching

Another improvement is the exceptions when matching of a record fails.

Consider this record and function:

```erlang
-record(rec, {count}).

rec_add(R) ->
    R#rec{count = R#rec.count + 1}.
```

In past releases, failure to match a record or retrieve an element from
a record would result in the following exception:

```text
1> t:rec_add({wrong,0}).
** exception error: {badrecord,rec}
     in function  t:rec_add/1 (t.erl, line 8)
```

Before Erlang/OTP 15 that introduced line numbers in exceptions, knowing
which record that was expected could be useful if the error occurred in
a large function.

Nowadays, unless several different records are accessed on the same
line, the line number makes it obvious which record was expected.

Therefore, in Erlang/OTP 25 the `badrecord` exception has been changed
to show the actual incorrect value:

```text
2> t:rec_add({wrong,0}).
** exception error: {badrecord,{wrong,0}}
     in function  t:rec_add/1 (t.erl, line 8)
```

The new `badrecord` exceptions will show up for code that has been compiled
with Erlang/OTP 25.

# Relocatable installation directory

Previously shell scripts (e.g., `erl` and `start`) and the `RELEASES` file
for an Erlang installation depended on a hard coded absolute path to the
installation's root directory. This made it cumbersome to move an
installation to a different directory which can be problematic for platforms
such as Android ([#2879](https://github.com/erlang/otp/pull/2879)) where the
installation directory is unknown at compile time. This is fixed by:

* Changing the shell scripts so that they can dynamically find the
  `ROOTDIR`. The dynamically found `ROOTDIR` is selected if it differs
  from the hard-coded `ROOTDIR` and seems to point to a valid Erlang
  installation. The `dyn_erl` program has been changed so that it can
  return its absolute canonicalized path when given the --realpath
  argument (dyn_erl gets its absolute canonicalized path from the
  realpath POSIX function). The dyn_erl's --realpath
  functionality is used by the scripts to get the root dir dynamically.

* Changing the release_handler module that reads and writes to the
  `RELEASES` file so that it prepends `code:root_dir()` whenever it
  encounters relative paths. This is necessary since the current
  working directory can be changed so it is something different than
  `code:root_dir()`.

# ETS-tables with adaptive support for write concurrency

It has since long been possible to optimize an ETS table for write concurrency doing like this:

```erlang
ets:new(my_table, [{write_concurrency, true}]).
```

Now we also introduce adaptive support for write concurrency which can be configured like this:

```erlang
ets:new(my_table, [{write_concurrency, auto}]).
```

This option forces tables to automatically change the number of locks that are used at run-time depending on how much concurrency is detected. When you enable automatic write concurrency `decentralized_counters` are also activated for even more scalable ETS tables. Use this option when you know that a lot of processes will be accessing an ETS table on systems with many number of cores.

For more details you can read [PR 5208](https://github.com/erlang/otp/pull/5208) that introduced the change and the [blog post about decentralized counters](/blog/scalable-ets-counters/).

# New option `short` for `erlang:float_to_list/2` and `erlang:float_to_binary/2`

A new option called `short` has been added to the functions `erlang:float_to_list/2` and `erlang:float_to_binary/2`. This option creates the shortest correctly rounded string representation of the given float that can be converted back to the same float again.

If option `short` is specified, the float is formatted
with the smallest number of digits that still guarantees that

```erlang
F =:= list_to_float(float_to_list(F, [short]))
```

When the float is inside the range (-2⁵³, 2⁵³), the notation
that yields the smallest number of characters is used (scientific
notation or normal decimal notation). Floats outside the range
(-2⁵³, 2⁵³) are always formatted using scientific notation to avoid confusing
results when doing arithmetic operations.

The implementation is contributed by Thomas Depierre and uses the Ryū algorithm.

We present Ryū, a new routine to convert binary floating point numbers to their decimal representations using only fixed-size integer operations, and prove its correctness. Ryū is simpler and approximately three times faster than the previously fastest implementation.
<https://github.com/ulfjack/ryu>

# The new module `peer` supersedes the slave module

 The [`peer`](/doc/man/peer.html) module provides functions for starting linked Erlang nodes. The Erlang node spawning new "peer" nodes is called `origin`, and the newly started nodes are peers.

 A peer node automatically terminates when it loses the control connection to the origin. This connection could be an Erlang distribution connection, or an alternative - TCP or standard I/O. The alternative connection provides a way to execute remote procedure calls even when Erlang Distribution is not available, allowing to test the distribution itself.

Peer node terminal input/output is relayed through the origin. If a standard I/O alternative connection is requested, console output also goes via the origin, allowing debugging of node startup and boot script execution (see [-init_debug](/doc/man/erl#flags)). File I/O is not redirected, contrary to [`slave`](/doc/man/slave.html) behavior.

The peer node can start on the same or a different host (via ssh) or in a separate container (for example Docker). When the peer starts on the same host as the origin, it inherits the current directory and environment variables from the origin.

## Note

This module is designed to facilitate multi-node testing with Common Test. Use the ?CT_PEER() macro to start a linked peer node according to Common Test conventions: crash dumps written to specific location, node name prefixed with module name, calling function, and origin OS process ID). Use [`random_name/1`](/doc/man/peer.html#random_name-1) to create sufficiently unique node names if you need more control.

A peer node started without alternative connection behaves similarly to `slave(3)`.

# `gen_XXX` modules has got a new `format_status/1` callback.

The [`format_status/2`](/doc/man/gen_server.html#Module:format_status-2) callback for `gen_server`, `gen_statem` and `gen_event` has been deprecated in favor of the new [`format_status/1`](/doc/man/gen_server.html#Module:format_status-1) callback.

The new callback adds the possibility to limit and change many more things than the just the state.

The purpose with both the old and the new `format_status` callbacks are to let the user filter away sensitive information and possibly data of huge volume from the crash reports.

# The `timer` module has been modernized and made more efficient

The timer module has been modernized and made more efficient, which makes the timer server less susceptible to being overloaded. The `timer:sleep/1` function now accepts an arbitrarily large integer.

# Crypto and OpenSSL 3.0

Some applications in OTP like SSL/TLS and SSH need cryptography to
work. That is provided by the OTP application crypto, which interfaces
Erlang to an external cryptolib in C using NIFs. The main example of
such an external cryptolib is [OpenSSL](https://www.openssl.org).

The OpenSSL cryptolib exists in many versions. OTP/crypto supports
0.9.8c and later, although only 1.1.1 is still maintained by OpenSSL.

OpenSSL has released its version 3.0 series, which is their future
platform totally re-built with a new API. The APIs of previous
versions (1.1.1 and older) are partly deprecated, although still
available in 3.0. The support of 1.1.1 will also end in a future.

Since it is vital to get security patches in the cryptolib, and in a
future only the 3.0 API might be available, OTP/crypto now from
OTP-25.0 interfaces OpenSSL 3.0 using the new 3.0 API. A few functions
from old APIs are still used, but they will be replaced as soon as
possible.

You as a user will hopefully not notice any difference: if you have
OpenSSL 1.1.1 (or older - not recommended) and build OTP, that one will
be used as previously. If you have any OpenSSL 3.0 version installed,
that one will be used without need of doing anything special except
for normal handling of dynamic loading paths in the OS.

# CA-certificates can be fetched from the OS standard place

With the new functions `public_key:cacerts_load/0,1` and `public_key:cacerts_get/0` the CA certificates can be fetched from the standard place of the OS (or from a file). 

They will then be cached in decoded form by use of `persistent_term` which makes them available in an efficient way for the `ssl` and `httpc` modules. The intention with this is to make it unnecessary to depend on for example `certifi` in many packages.

On Windows and MacOSx the certificate store is not an ordinary file so the information is fetched via an API using a NIF (Windows) or with an external program (MacOSx).

Example with `ssl`

```erlang
%% makes the certificates available without copying
CaCerts = public_key:cacerts_get(), 
% use the certificates when establishing a connection
{ok,Socket} = ssl:connect("erlang.org",443,[{cacerts,CaCerts}, {verify,verify_peer}]), 
...
```

We also plan to update the http client (`httpc`) to use this soon.

# A new fast Pseudo Random Generator

A new custom designed Pseudo Random Generator [`rand:mwc59`](/doc/man/rand.html#mwc59-1)
has been implemented.  It is probably the fastest possible
generator with good quality that can be written in Erlang.
To do this it barely avoids bignums, allocating heap data,
and uses only a minimal number of fast operations.

Under the "right" circumstances: A number that takes 60 ns to generate
with the default generator can be generated in 4 ns with `rand:mwc59`.

It is intended for applications in dire need for speed
in PRNG numbers, but not any of the comfort features
that [`rand`](/doc/man/rand.html) otherwise offers.

# More details

For more details about new features and potential incompatibilities in OTP 25.0
see <https://erlang.org/download/otp_src_25.0.readme>.
