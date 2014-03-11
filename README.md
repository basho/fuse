# Fuse — A Circuit Breaker implementation for Erlang

This application implements a so-called circuit breaker implementation for Erlang.

[![Build Status](https://travis-ci.org/jlouis/fuse.png?branch=master)](https://travis-ci.org/jlouis/fuse)

# Introduction

When we build large systems, one of the problems we face is what happens when we have long dependency chains of applications. We might have a case where applications call like this:

	app_A → app_B → app_C
	
Now, if we begin having errors in application `B` down the road, the problem is that application `A` needs to handle this by waiting for a timeout of Application B all the time. The problem is that this incurs latency in the code base. A Circuit Breaker detects the error in the underlying system and then avoids making further queries to the underlying system. This allows you to handle the breakage systematically in the system.

The broken circuit will be retried once in a while in the system. The system will then auto-heal if connectivity comes back for the underlying systems. Note that this will be able to break cascading errors so they don't make the system fail with long latency timeouts.

# Tutorial

To use fuse, you must first start the fuse application:

	application:start(fuse).
	
but note that in real systems it is better to have other applications *depend* on fuse and then start it as part of a release boot script. Next, you must install a fuse into the system by *installing* a fuse descriptions:

	Name = database_timeout,
	Opts = [
		{policy, {counter, 5}},
		{folsom_metric, 'fuse.database_timeout'},
		alarm,
		{recheck_interval, 5*60*1000}
	],
	fuse:install(Name, Opts).
	
This sets up a *fuse* with a given Name and a given set of options. Fuses are name-created idempotently, so your application can recreate a fuse if it wants. Note however, that fuse recreation has two major rules:

* Reinstalling a fuse resets its internal state.
* Reinstalling a fuse can reset its options.

So re-creation of a fuse overwrites the existing fuse.

Once you have installed a fuse, you can use it in one of two ways:

	case fuse:ask(database_timeout) of
		ok -> …;
		blown -> …
	end,
	
This queries the fuse for its state and lets you handle the case where it is currently blown. Now suppose you have a working fuse, but you suddenly realize you get errors of the type `{error, timeout}`. Since you think this is a problem, you can tell the system that the fuse is under strain. You do this by *melting* the fuse:

	case emysql:execute(Stmt) of
	    {error, connection_lock_timeout} ->
	    	fuse:melt(database_timeout),
	    	…
	    …
	end,
	
The fuse has a policy, so once it has been melted too many times, it will blow for a while until it has heated down. Then it will let a single request through again to make sure it works like expected.

Another way to run the fuse is to use a wrapper function. Suppose you have a function with the following spec:

	-spec exec() -> {ok, Result} | {melt, Result}
	  when Result :: term().

	%% To use this function:
	case fuse:run(fun exec/0) of
		{ok, Result} -> …;
		blown -> …
	end,
	
this function will do the asking and melting itself based on the output of the underlying function. This is highly recommended since it is often easier to handle.
	
## Options to give to the fuse

The fuses support several options which you can give them in order to configure them appropriately:

* `alarm` — When the fuse blows, raise the appropriate alarm through SASL
* `folsom_metric` — Keep folsom metric data under the given prefix

# Tests

Fuse is written with two kinds of tests. First of all, it uses a set of Common Test test cases which runs the basic functionality of the system. Furthermore, fuse is written with Erlang QuickCheck test cases. EQC tests are written before the corresponding code is written, and as such, this is EQC Driven Development.

To run the standard tests, execute:

	make tests
	
To run the EQC tests, you have to

	cd eqc_test
	make console
	
And then in the Erlang console, you can execute

	make:all([load]).
	eqc:module(fuse_eqc).
	

