---
layout: post
title: Using Java's Project Loom to build more reliable distributed systems
---

One of the most personally influential tech talks I've seen is
[Will Wilson's FoundationDB talk at StrangeLoop 2014](https://www.youtube.com/watch?v=4fFDFbi3toc).
The rough idea went as follows.

1. There are lots of clever ideas you can put into a distributed database, but it's very hard to make them work
   in practice because it's hard to design and implement them all properly, especially not in such a way that
   they work well together.
1. Instead of starting by building a database, FoundationDB built a _deterministic simulation_ of a database.
   They mocked up networks, filesystems, hosts, which all worked similarly to those you'd see on a real system,
   but with simulated time and resources allowing simple injection of failures.
1. They could now write deterministic tests of their system to test that it handled failure cases seamlessly.
1. Once their simulation ran well enough, they could start running similar tests on real hardware. Once those
   worked, they had a database.

The simulation environment was apparently challenging to build, but highly beneficial once it existed. The tests
written with it had a few major advantages over traditional stress tests of distributed databases.

A good example of the stress tests that modern database vendors run on their software is [Jepsen](https://jepsen.io)
which tests partition tolerance. It runs the distributed database as a cluster, injecting failures by controlling
the network, system clock, host uptime. A test is written to simulate some consistency property, and failures
randomly happen. If the failure mode is seen, the test is failed. I'm describing a single testing framework which
is very well put together and well known; most systems might have some version of this for some purpose; perhaps
correctness or performance testing.

The FoundationDB version of this is rather similar, except the nodes all exist within a single process (indeed,
thread) and the network latencies, clock skew, host failures all happen as a function of a single random number
generator. The downside is that because you're testing against your mocks, it might not match reality. The
advantages are many.

1. The tests can be much faster. In a Jepsen style test, if you want to test what happens when a node's latency
   goes up to a second, you will have to wait while your many RPCs all take seconds to complete. In the FoundationDB
   style test, because time is simulated, if everything is waiting for one simulated second long RPC, the system
   clock can simply be incremented by that time and the test can proceed. Put simply, in a sufficiently simple
   system the simulated RPC latency would not be expected to affect overall test performance.
1. When the tests fail, you have a reproduction. Because the random number generator used to generate failures
   is deterministic and has a static seed, the test failing once can be used to help debug the test.
