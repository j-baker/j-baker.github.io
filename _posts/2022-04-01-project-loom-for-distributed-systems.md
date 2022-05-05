---
layout: post
title: Using Java's Project Loom to build more reliable distributed systems
---

Note: Through this document I'm going to use the example of OLTP databases - the idea generalizes fairly broadly.

Building distributed databases is very difficult. There are lots of clever ideas you can put into a distributed database, but making the ideas work well together in a real system can be very challenging because different decompositions of the same problem can massively affect how difficult it is to solve, and bugs are typically both subtle and hard to detect. This is especially problematic as the system evolves, where it can be difficult to
understand whether an improvement helps or hurts.

Paraphrasing Tony Hoare, practitioners therefore typically go down one of two paths, to either make the kernel of the software so simple that there are no bugs, or to make it complex enough that it's hard to discover bugs. In the mid-2010s, the [Jepsen project](https://jepsen.io) made inroads on challenging the latter approach.

## Jepsen

Jepsen is a software framework and blog post series which attempts to find bugs in distributed databases, especially although not exclusively around partition tolerance. The database is installed in a controlled environment (say, virtual machines or Docker containers) and operations are issued and their results recorded. Control over the environment is used to inject _nemeses_ (say, clock skew, network latency, network partitions) randomly, while the operations are being issued and recorded reliably. Invariants can be written according to the database's marketing
material, and should the results violate the invariants, you have cast iron evidence of a bug. As a comparative tool,
the results speak for themselves; between when I first heard about Jepsen in the summer of 2014 and now, it has become
a standard badge of approval and Kyle Kingsbury has managed to make a consulting career out of it.

INSERT DIAGRAM HERE

As a white box tool for bug detection, Jepsen is fantastic. If you've written the database in question, Jepsen leaves something to be desired. By falling down to the lowest common denominator of 'the database must run on Linux', testing is both slow and non-deterministic. For a quick example, suppose I'm looking for bugs in Apache Cassandra which occur due to adding and removing nodes. It's usual for adding and removing nodes to Cassandra to take hours or even days, although for small databases it might be possible in minutes, probably not much less than. A Jepsen environment could only run one iteration of the test every few minutes; if the failure case only happens one time in every few thousand attempts, without massive parallelism I might expect to discover issues only every few days.

Suppose that we either have a large server farm or a large amount of time and have detected the bug somewhere in our stack of at least tens of thousands of lines of code. We now have a bug, but no reproduction of said bug. If there is some kind of smoking gun in the bug report or a sufficiently small set of potential causes, this might just be the start of an odyssey.

Jepsen is probably the best known example of this type of testing, and it certainly moved the state of the art; most database authors have similar suites of tests. ScyllaDB documents their testing strategy [here](https://www.scylladb.com/product/technology/scylla-testing/) and while the styles of testing might vary, the strategies are similar.

Jepsen is limited here by its broadness. As the author of the database, we have much more access to the database if we so desire, as shown by [FoundationDB](https://www.foundationdb.org/).

## FoundationDB

One of the most personally influential tech talks I've watched is
[Will Wilson's FoundationDB talk at StrangeLoop 2014](https://www.youtube.com/watch?v=4fFDFbi3toc).
I highly recommend watching the talk, but if you don't have time here are my takeaways.

When the FoundationDB team set out to build a distributed database, they didn't start by building a distributed database. Instead, they built a _deterministic simulation_ of a distributed database. They built mocks of networks, filesystems, hosts, which all worked similarly to those you'd see in a real system but with simulated time and resources allowing injection of failures.

INSERT DIAGRAM HERE

They could now write deterministic tests of their system, similarly to the Jepsen tests, except for two key differences:

1. The tests were deterministic and so any test failure naturally had a baked in reproduction (one can simply re-run the test to observe the same result).
1. The tests could be made extremely fast because the test doubles enabled skipping work. For example, suppose that a task needs to wait for a second. Instead of implementing by actually waiting for a second, the simulation could simply increment a number (time) by a second. Alternatively, consider an RPC connection. Instead of actually using the TCP stack, a simulation could be used which does not require any operating system collaboration.

Once the team had built their simulation of a database, they could swap out their mocks for the real thing, writing the adapters from their interfaces to the various underlying operating system calls. At this point, they could run the same tests in a way similar to Jepsen (my understanding was that a small fleet of servers, programmable switches and power supplies was used). These real-hardware re-runs could be used to ensure that the simulation matched the real world, since any failure not seen in the simulation naturally corresponds to a deficiency in the simulation.

Once the team was satisfied with their simulation, they had a database they could sell!

I've heard that this was beneficial to the company in a number of ways, but the one that caught my eye can be paraphrased as 'because we had very high confidence in the correctness of our code and the changes we were making to it, we could be much more aggressive with regards to the technical architecture of the system'.

INSERT DIAGRAM HERE

The problem with this approach is that it makes a huge technical tradeoff - programming languages and operating system concepts typically do not make it easy to solve problems using this mechanic because large parts of the system are not naturally controllable in a deterministic way. Let's use a simple Java example, where we have a thread that kicks off some concurrent work, does some work for itself, and then waits for the initial work to finish.

```java
ExecutorService executor = createExecutorService();
Future<String> task = executor.submit(() -> {
   Future<String> concurrentComputation = executor.submit(this::doExpensiveComputation);
   return doExpensiveComputation2() + concurrentComputation.get();
});
```

If the `ExecutorService` involved is backed by multiple operating system threads, then the task will not be executed in
a deterministic fashion because the operating system task scheduler is not pluggable. If instead it is backed by a single operating system thread, it will deadlock.

An alternative approach might be to use Promises/ListenableFutures/callbacks.

```java
ListeningExecutorService executor = MoreExecutors.listeningDecorator(createExecutorService());
ListenableFuture<String> task = Futures.submitAsync(() -> {
   ListenableFuture<String> concurrentComputation = executor.submit(this::doExpensiveComputation);
   String ourResult = doExpensiveComputation2();
   return Futures.transform(
      concurrentComputation,
      result -> ourResult + result,
      MoreExecutors.directExecutor());
}, executor);
```
This approach can be more easily made deterministic, but there is high syntactic overhead, and we may well lose the optionality of execution order. For example, the above code when executed deterministically will always run `doExpensiveComputation2` before `doExpensiveComputation`. Many languages offer syntactic sugar for this in the form of async-await control flow. In these cases, it's uncommon to be able to control execution due to the implementation of the construct, so we have the same problem as the multi-threaded approach.

Such an approach is viable in isolated areas. Palantir's [Dialogue](https://github.com/palantir/dialogue/) uses this model to implement an RPC library. By utilizing an asynchronous implementation and using a [deterministic scheduler](https://github.com/palantir/dialogue/tree/develop/simulation/src/main/java/com/palantir/dialogue/core) (as well as swapping out the underlying Apache HTTP client for a stub implementation), they can easily simulate the load balancing logic and generate predictions of how well different algorithms handle different cases, producing [pretty charts of the results](https://github.com/palantir/dialogue/blob/develop/simulation/src/test/resources/slow_503s_then_revert%5BCONCURRENCY_LIMITER_ROUND_ROBIN%5D.png) which can be posted to pull requests each time they change. This had a side effect - by measuring the runtime of the simulation, one can get a good understanding of the CPU overheads of the library and optimize the runtime against this. In some ways this is similar to [SQLite's approach to CPU optimization](https://www.sqlite.org/draft/cpu.html). Even then, there were tradeoffs made for reliability reasons. Due to the lack of no try-finally construct, each interaction between an InputStream and Dialogue is a dangerous dance.

More sophisticated implementations can easily become unwieldy. I briefly experimented with an implementation of Palantir's transactional key-value store AtlasDB in such a style early on in 2020 in some free time. While I had something that worked by the end and passed rudimentary correctness tests, the code I had written was unfortunately inefficient, [inscrutable](https://github.com/palantir/atlasdb/blob/jbaker/new/atlasdb-new/src/main/java/com/palantir/atlasdb/v2/api/transaction/scanner/ReadCommittedDataReader.java#L51) and disastrous to debug, since a stack trace 10 elements deep might instead become 10 anonymous runnables in an executor.

FoundationDB's usage of this model required them to build their own programming language, [Flow](https://apple.github.io/foundationdb/flow.html), which is transpiled to C++. The simulation model therefore infects the entire codebase and places large constraints on dependencies, which makes it a difficult choice.

Java's Project Loom considerably shakes up this tradeoff.

## Project Loom

[Project Loom](https://wiki.openjdk.java.net/display/loom/Main) provides 'virtual' threads as a first class concept within Java. There is plenty of good information in the 2020 blog post '[State of Loom](http://cr.openjdk.java.net/~rpressler/loom/loom/sol1_part1.html)' although details have changed in the last two years.

I will give a simplified description of what I find exciting about this. In traditional Java we can launch a Thread. If it needs to pause for some reason, the thread will be paused, and will resume when it is able to.

With Loom's virtual threads, when a thread starts, a `Runnable` (function of type `unit -> unit`) is submitted to an `Executor` (function of type `Runnable -> unit`, contract that the Runnable function will eventually be called). When that task is run by the executor, if the thread needs to block, the submitted runnable will exit, instead of pausing. When the thread can be unblocked, a new runnable is submitted to the same executor to pick up where the previous `Runnable` left off.

By utilizing this API, we can exert fine grained deterministic control over the concurrency primitives within Java. Let's give a micro level example. Suppose we're trying to test the correctness of a buggy version of Guava's `Suppliers.memoize` function.

```
class BrokenMemoizingSupplier<T> implements Supplier<T> {
    private final Lock lock;
    private final Supplier<T> delegate;
    private volatile boolean initialized = false;
    private T value;

    BrokenMemoizingSupplier(Supplier<T> delegate) {
        this(new ReentrantLock(), delegate);
    }

    @VisibleForTesting
    BrokenMemoizingSupplier(Lock lock, Supplier<T> delegate) {
        this.lock = lock;
        this.delegate = delegate;
    }

    @Override
    public T get() {
        if (!initialized) {
            lock.lock();
            try {
                // This code is broken because initialized may have changed between the call to 'lock' and now.
                T result = delegate.get();
                value = result;
                initialized = true;
                return result;
            } finally {
                lock.unlock();
            }
        }
        return value;
    }
}
```

This test is enough to deterministically exhibit the bug:

```
class BrokenMemoizingSupplierTest {
    // This will fail
    @ParameterizedTest
    @MethodSource("range")
    void testAllCases(int seed) {
        AtomicInteger countCalls = new AtomicInteger();
        BrokenMemoizingSupplier<Integer> supplier = new BrokenMemoizingSupplier<>(
                new YieldingLock(), countCalls::incrementAndGet);
        Random random = new Random(seed);
        RandomizingExecutor executor = new RandomizingExecutor(random);
        newVirtualThread(executor, supplier::get);
        newVirtualThread(executor, supplier::get);
        executor.drain();
        assertThat(supplier.get()).isOne();
    }

    static Stream<Integer> range() {
        return IntStream.range(0, 10).boxed();
    }

    /**
     * This executor executes randomly for brevity. By tracking which tasks launch which other tasks, one could
     * generate all valid interleavings for small cases such as this with slightly more code.
     */
    private static final class RandomizingExecutor implements Executor {
        private final Random random;
        private final List<Runnable> queue = new ArrayList<>();

        private RandomizingExecutor(Random random) {
            this.random = random;
        }

        @Override
        public void execute(Runnable command) {
            queue.add(command);
        }

        public void drain() {
            while (!queue.isEmpty()) {
                Collections.shuffle(queue, random);
                Runnable task = queue.remove(queue.size() - 1);
                task.run();
            }
        }
    }

    // Thread.yield will pause the thread and cause it to become schedulable again.
    private static final class YieldingLock extends ReentrantLock {
        @Override
        public void lock() {
            Thread.yield();
            super.lock();
        }
    }
}
```

Now, this test is highly limited as compared to a tool like [jcstress](https://shipilev.net/blog/2016/close-encounters-of-jmm-kind), since any issues related to compiler reordering of reads or writes will be untestable. This approach does however have the rather large advantage that its application can be scaled to the level of whole systems rather than an individual component.

## Raft implementation

To demonstrate this approach when scaled up, I challenged myself to write a toy implementation of [Raft](https://raft.github.io/raft.pdf), according to the simplified protocol in the paper's figure 2 (no membership changes, no snapshotting).

You may find the code for this [here]().


















Not only does this help them validate their load balancing logic, but by measing the CPU time used in a benchmark run (and the differences after improvements) they have a simple proxy for the overall cost of their own code.


In FoundationDB's case


1. Building distributed systems is very difficult. There are lots of clever ideas you can put into a distributed
   database, but it can be challenging to make the ideas work together in a real system because it's hard to design
   and implement them all properly in such a way that they work really well together, and especially
   not as the system evolves.

2. In practice, this leads to conservatism.  
3. This generally either limits the vision you can put into the design of your database, or ot

4. There are lots of clever ideas you can put into a distributed database, but it's very hard to make them work
   in practice because it's hard to design and implement them all properly, especially not in such a way that
   they work well together.
5. Instead of starting by building a database, FoundationDB built a _deterministic simulation_ of a database.
   They mocked up networks, filesystems, hosts, which all worked similarly to those you'd see on a real system,
   but with simulated time and resources allowing simple injection of failures.
6. They could now write deterministic tests of their system to test that it handled failure cases seamlessly.
7. Once their simulation ran well enough, they could start running similar tests on real hardware. Once those
   worked, they had a database.

Apparently the simulation environment was challenging to build, but had several practical advantages.

The simulation environment was challenging to build, but highly beneficial once it existed. The tests
written with it had a few major advantages over traditional stress tests of distributed databases.

A good example of the stress tests that modern database vendors run on their software is [Jepsen](https://jepsen.io)
which tests partition tolerance. It runs the distributed database as a cluster, injecting failures by controlling
the network, system clock, host uptime. A test is written to simulate some consistency property, and failures
randomly happen. If the failure mode is seen, the test is failed. I'm describing a single testing framework which
is very well put together and well known; most systems have some version of this for some purpose; perhaps
correctness or performance testing.

The FoundationDB version of this behaves similarly, except the nodes all exist within a single process (thread)
and the network latencies, clock skew, host failures all happen as a function of a single random number generator.



The FoundationDB version of this is rather similar, except the nodes all exist within a single process (indeed,
thread) and the network latencies, clock skew, host failures all happen as a function of a single random number
generator. The downside is that because you're testing against your mocks, it might not match reality. The
advantages are many.

1. The tests can be much faster. In a Jepsen style test, if you want to test what happens when a node's latency
   goes up to a second, you will have to wait while your many RPCs all take seconds to complete. In the FoundationDB
   style test, because time is simulated, if everything is waiting for one simulated second long RPC, the system
   clock can simply be incremented by that time and the test can proceed. Put simply, in a sufficiently simple
   system the simulated RPC latency would not be expected to affect overall test performance.
2. When the tests fail, you have a reproduction. Because the random number generator used to generate failures
   is deterministic and has a static seed, the test failing once can be used to help debug the test.
