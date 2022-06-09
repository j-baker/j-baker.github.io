---
layout: post
title: Vectorized sorting in Java - down a rabbit hole to a modified JVM
---

In this post I explain a little about how to use Java's Vector APIs, attempt to explain how they turn out fast, and then use them to implement a performant sorting algorithm. I then explain some problems I found, and how I resolved them. Supporting code is published [here](https://github.com/j-baker/vector-experiments).

I'm an occasional reader of [Daniel Lemire's blog](https://lemire.me). Many of his posts follow a standard pattern, where he takes some standard operation (json parsing, bitsets, existence filters, sorting, integer encoding), reforms it using vector operations and makes it vastly faster due to increasing the data level parallelism. It's a fun read, and the numbers are scary high (most of us can only dream of writing code that processes 1GB/sec with a single core, let alone doing something useful in that time).

I work at a Java shop and so most of these techniques have ended up being somewhat out of scope for me, because historically Java didn't provide much support for vectorization. Traditional Java has skewed heavily towards scalar code, with some very limited automatic vectorization in toy cases which you won't ever notice in practice, and a number of hand-rolled vectorized intrinsics on core types to ensure the quality of the standard library (look for `@IntrinsicCandidate` in types like `Latin1String`). I've certainly faced problems that would have benefitted from vectorization, but avoided the solution due to not wanting to go through the overhead of building, testing, and distributing a cross-platform JNI library.

There's an ongoing OpenJDK project called [Project Panama](https://openjdk.java.net/projects/panama/) which has been working on improving interop between the JVM and more 'native' APIs. One of their subprojects has focused on providing vectorized intrinsics within the JVM, and I decided to test out these APIs by trying to use them to implement sorting on integer arrays.

# Mile high view of Java's Vector API

Java's Vector API is currently distributed as a preview with recent versions of Java. To use it, you must append `--enable-preview --add-modules jdk.incubator.vector` to your `java` and `javac` commands.

With the exception of char, Java supports each numerical primitive type as a vector (byte, short, int, long, float, double), with the corresponding vector type being `*Vector`, like `IntVector`. In order to use vectors, one must choose a 'species', which represents the width. Java presently supports up to 512 bit vectors, so one might choose a specific width like `IntVector.SPECIES_256`, the maximum with `IntVector.SPECIES_MAX` or the maximum hardware accelerated size with `IntVector.SPECIES_PREFERRED`.

One can then use code like the below to process data.

```java
static final VectorSpecies<Integer> SPECIES = IntVector.SPECIES_256;

void multiply(int[] array, int by) {
    int i = 0;
    int bound = SPECIES.loopBound(array.length);
    IntVector byVector = IntVector.broadcast(SPECIES, by);
    for (; i < bound; i += SPECIES.length()) {
        IntVector vec = IntVector.fromArray(SPECIES, array, i);
        IntVector multiplied = vec.mul(byVector);
        multiplied.intoArray(array, i);
    }
    for (; i < array.length; i++) {
        array[i] *= by;
    }
}
```

There are two other useful primitives. The first is `VectorMask` which you can think of as an array of booleans. Many of the methods can take masks to switch on or off behaviour on specific vector lanes, and predicates return masks.

```java
VectorSpecies<Integer> species = IntVector.SPECIES_64;
IntVector vector = IntVector.broadcast(species, 1);
VectorMask<Integer> mask = VectorMask.ofValues(species, true, false);
IntVector result = vector.mul(0, mask);
assertThat(result).isEqualTo(IntVector.fromValues(species, 0, 1));
```

The second other useful type is `VectorShuffle`. This can be used to rearrange vector lanes. For example:

```java
VectorSpecies<Integer> species = IntVector.SPECIES_128;
IntVector vector = IntVector.fromArray(species, new int[] {12, 34, 56, 78});
assertThat(vector.rearrange(VectorShuffle.fromValues(species, 4, 3, 2, 1))).isEqualTo(IntVector.fromArray(species, new int[] {78, 56, 34, 12}));
```

# How it works (well)

The benefit of SIMD code is that we can process more data within a single instruction; for example with 256 bit instructions we can perhaps process 8 ints per cycle instead of 1. In the multiply example above, we have a
vectorized loop and a scalar loop. If the two loops are identical except in the scalar case we see an x86 `MUL` instruction being run and in the vectorized case we see a `PMULQ` instruction being run, we might expect to see the full benefit of the vectorization. However, the loop is represented as a number of new Java objects, and if we have to run even only 8 additional instructions to initialize the new objects, we've given away our advantage. Java's vector APIs put a lot of effort into working well in this regard, and rely heavily on the following optimizations.

## Inlining

Inlining is a simple, easy to understand compiler technique that can dramatically improve performance.

### What is inlining?

When we inline a method, we are replacing a function call to that method with the definition of the function we are calling.

```java
boolean notInlined(String string) {
    return string1.isEmpty() || string1.length() == 1;
}

boolean inlined(String string) {
    return string1.value.length == 0 || string1.value.length == 1;
}
```

## When does Java inline functions?

There are a [number of heuristics](https://github.com/openjdk/jdk/blob/9583e3657e43cc1c6f2101a64534564db2a9bd84/src/hotspot/share/opto/bytecodeInfo.cpp#L116) within Java that decide whether or not a function should be inlined. Some are as follows:

- If a method has thrown many times, it is inlined. Presumably this is because the try-catch loop can be optimized.
- If a method A calls a method B many times more frequently (default 20x) than the outer method is called, it is inlined unless its definition is very large (> 335 JVM opcodes).
- If a method is very small (<=35 opcodes) it is inlined.

Virtual methods make this all harder, but various other analyses help with this.

There is also a magic Java-internal annotation that helps with inlining; `@ForceInline`. If this annotation is applied, and the class it is applied to comes from the 'boot classloader' (your application does not), then the data will be inlined.

## Why is it so much better when a function can be inlined?

There are two key improvements that inlining can lead to. The first is that it avoids the function call overhead. Calling a function is cheap but not free because various book-keeping must be done (flushing registers to the stack, moving the program counter to the other function, restoring when it completes, etc). Once we've inlined a function, these costs disappear.

The second improvement is arguably far more important (and is why inlining is sometimes referred to as 'the mother of all optimizations') - because we've inlined the function, the compiler possibly has more information, enabling further optimizations. For example, in the above case in psuedo-Java, the method 'inlined' is equivalent to:

```java
boolean inlined(String string) {
    if (string == null) {
        throw new NullPointerException();
    }
    if (arrayLength(string.value) == 0) { // let's assume the compiler knows that String.value is never null, because the constructors never let it be so.
        return true;
    }
    if (arrayLength(string.value) == 1) {
        return true;
    }
    return false;
}
```

In this instance, the compiler will likely be able to intuit that arrayLength will not change between the two calls. It will likely also know that an array length cannot be less than 0. The compiler therefore may execute this code as:

```java
boolean inlined(String string) {
    if (string == null) {
        throw new NullPointerException();
    }
    if (arrayLength(string.value) <= 1) {
        return true;
    }
    return false;
}
```

which ensures that not only is the function call overhead avoided, but also that the method is simpler. This optimization is used all over the place; null checks can be avoided when it's known that the value is not null, shared parts of methods can avoid being re-executed, etc.

## Escape analysis and scalar replacement

Escape analysis is a compiler technique that can be used to gain information about the lifetimes of objects. When we create an object inside a function, its lifetime is either shorter than the scope of the function, or longer. If it is possibly longer, we say that it `escapes`. If we know that an object does not escape, there are other optimizations we can do. One that Java performs is called 'scalar replacement' - roughly, it can entirely avoid creating the object and instead allocate any containing fields as stack variables.

For example, these two methods are semantically identical, and might be compiled as such.

```java
Optional<String> withoutScalarReplacement(String someString) {
    return Optional.ofNullable(someString).map(String::trim).filter(String::isEmpty);
}

Optional<String> withScalarReplacement(String someString) {
    if (someString == null) {
        return Optional.empty();
    }
    String trimmed = someString.trim();
    if (trimmed == null) {
        return Optional.empty();
    }
    if (!trimmed.isEmpty()) {
        return Optional.empty();
    }
    return Optional.of(trimmed);
}
```

In this example, we are able to fully avoid allocating two different intermediate `Optional` objects. Scalar replacement can mostly only be useful when inlining occurs. For example, in the above, `Optional.ofNullable` possibly allocates an `Optional`, which escapes. The implementation cannot avoid this allocation. If it is inlined, the object may no longer escape and therefore the scalar replacement may be viable.

## Intrinsics

In the Java standard library, many methods have a pure-Java implementation, and an 'intrinsic' implementation. Sometimes the intrinsic might be pure C++, sometimes assembly. The rough idea is that certain methods are called so frequently in Java or are so important to be fast, that we shouldn't rely on the compiler producing the right code; the JVM should contain a native implementation that is known to have the right properties that is to be used instead. When these methods are compiled, the intrinsic is substituted in. The availability of an intrinsic can be denoted via the `@IntrinsicCandidate` annotation.

## How Java's Vector API uses these tools

Each `IntVector` object in Java contains an `int[]` which represents the vector. There are pure Java implementations of each of the vector operations - check `IntVector.binaryOperations(int)` for the definitions of the binary lanewise operations. However, these are designed to be only a fallback - they are primarily called by methods like `VectorSupport.binaryOp` which has the magic annotation `@IntrinsicCandidate`. In other words, if these vectorized methods are run on a system with relevant SIMD functionality (e.g. when AVX2 or NEON is supported), then they will be replaced with native implementations.

Furthermore, all of the relevant vector methods (`fromArray`, `intoArray`, `mul`, `rearrange`, ...) have `@ForceInline` applied to them, so the definitions will be dropped directly into the code that calls them, enabling the scalar replacement. At this point, the fake `int[]` buffer can also be dropped at the time of register allocation, and made a stack variable, in the same way as other primitives.

In conclusion, when we use the vector API, everything gets inlined. This gives the compiler the most it can have to work with so as to remove enough function call overhead to actually compile a vector operation to only a couple of instructions. Practically speaking, you end up with code that looks like it creates many objects, but actually compiles to code that allocates no memory. It is extremely disconcerting.

# Quicksort using IntVectors

For purposes of this exercise, I'm going to expend my effort primarily on the operation of partitioning. I select the pivots by sampling 8 elements and using their median. I also decided to not go down the path of empirically trying to find patterns, because my goal is to exercise the SIMD operations and not to build a production sorting algorithm.

A simple in-place quicksort algorithm can be summarized as follows:

```java
void quicksort(int[] array, int from, int to) {
    int size = to - from;
    if (size <= 1) {
        return;
    }
    int pivot = partition(array, from, to);
    quicksort(array, from, pivot);
    quicksort(array, pivot + 1, to);
}

int partition(int[] array, int from, int to) {
    int pivot = array[to - 1];
    int boundary = partition(array, pivot, from, to - 1);
    int temp = array[boundary];
    array[to - 1] = temp;
    array[boundary] = pivot;
    return boundary;
}

int partition(int[] array, int pivot, int lowerPointer, int upperPointer) {
    while (lowerPointer < upperPointer) {
        if (array[lowerPointer] < pivot) {
            lowerPointer++;
        } else if (array[upperPointer - 1] >= pivot) {
            upperPointer--;
        } else {
            int tmp = array[lowerPointer];
            int index = --upperPointer;
            array[lowerPointer++] = array[index];
            array[index] = tmp;
        }
    }
    return lowerPointer;
}
```

For purposes of vectorizing, I'm primarily looking at reimplementing the 'partition' method. I also implement a special sorting function for regions <= 8 elements using a sorting network.

## Vectorizing the partition method

We're trying to replace the partition method. The purpose is to reorder a list of items such that it contains a contiguous region of items less than the pivot argument, followed by a contiguous region of items greater than the pivot argument. If we can partition a single vector, we can expand this to the whole array.

### Partitioning a single 8-lane vector

There is an AVX512 instruction which is absolutely perfect for this specific task. It's called `VPCOMPRESSQ` and development versions of Java expose it as `.compress()`, like:

```java
IntVector vec;
VectorMask<Integer> lessThan = vec.lt(pivot);
IntVector lt = vec.compress(lessThan);
IntVector gte = vec.compress(lessThan.not());
```

My computer does not have AVX512 and without hardware acceleration it executes incredibly slowly (>20ns rather than the 0.25ns I'd expect). So, we must work around! To work around, we can leverage the fact that for any mask, there is a permutation of the lanes which leads to all the 'trues' in the mask appearing before all the 'falses'. For example, if the mask reads `FFFFFFFT`, such a permutation might be `[7, 1, 2, 3, 4, 5, 6, 0]`. We can create such a mask using the following code:

```java
VectorShuffle<Integer> partitioner(VectorMask<Integer> vectorMask) {
    VectorSpecies<Integer> species = vectorMask.species();
    int numTrues = vectorMask.trueCount();

    int mask = (int) vectorMask.toLong(); // bit 2^n is set to true iff lane n is set in the mask.
    int[] ret = new int[species.length()];
    int unmatchedIndex = numTrues;
    int matchedIndex = 0;
    for (int i = 0; i < species.length(); i++) {
        if ((mask & 1) == 0) {
            ret[unmatchedIndex++] = i;
        } else {
            ret[matchedIndex++] = i;
        }
        mask >>= 1;
    }
    return VectorShuffle.fromValues(species, ret);
}
```

We have an 8-lane vector, so the total possible number of such shuffles is `2^8 = 256`. Each shuffle is conceptually 8 bytes, and so cumulatively this would need around 1kB, probably around 4kB with overheads, easily small enough to fit in L1 cache as a lookup table.

We can therefore implement a similar partitioning process using:

```java
IntVector vec;
VectorMask<Integer> lessThan = vec.lt(pivot);
IntVector rearranged = vec.rearrange(partitions[(int) lessThan.toLong()]);
```

### Partitioning two partitioned 8-lane vectors

n.b. this approach can also be used to partition a single 16 lane vector without a giant lookup table.

If we have two partitioned vectors and know how many elements less than the pivot each contains, we can easily merge them into a single partitioned vector.

```java
IntVector vec1;
int numTrues1;
IntVector vec2;
int numTrues2;

IntVector rotatedVec2 = vec2.rearrange(rotate(numTrues1));
VectorMask<Integer> mask = VectorMask.fromLong(species, (1L << numTrues1) - 1).not();
IntVector left = vec1.blend(rotatedVec2, mask); // blend takes the elements that the mask allows from the argument and the others from the base vector
IntVector right = rotatedVec2.blend(vec1, mask);
```

### Extending partitioning 16 elements to a whole array

The best approach I was able to come up here is one where I tried to avoid doing any unnecessary vector operations. From the left of the array, I read two vectors and partition them. At this point (example has 8 lanes), inside them there are either >8 (1) or <=8 (2) elements that are less than the pivot.

In case 1, every element in the left vector is less than the pivot. We can write it back into the array and increment the `from` pointer like in the simplified algorithm above. Some elements in the right array are less than and some are greater than the pivot. In case 2, we have the opposite - all elements on the right side are greater than the pivot, and so we swap that vector to the back and continue. Special casing the case where we can process two vectors at a time did not improve performance. This can be thought of as being very similar to the scalar algorithm.

```java
IntVector compareTo = IntVector.broadcast(SPECIES, pivot);

IntVector cachedVector;
int cachedNumTrues;
boolean initialize = true;
while (index + SPECIES.length() < upperBound) {
    if (initialize) {
        initialize = false;
        // load our first vector and partition it
        cachedVector = IntVector.fromArray(SPECIES, array, index);
        VectorMask<Integer> cachedMask = cachedVector.lt(compareTo);
        cachedVector = cachedVector.rearrange(IntVectorizedQuickSort.compress(cachedMask));
        cachedNumTrues = cachedMask.trueCount();
    }

    // partition our second vector
    int index2 = index + SPECIES.length();
    IntVector vector2 = IntVector.fromArray(SPECIES, array, index2);
    VectorMask<Integer> mask2 = vector2.lt(compareTo);
    int numTrues2 = mask2.trueCount();
    IntVector rearranged2 = vector2.rearrange(IntVectorizedQuickSort.compress(mask2));

    // merge our two partitioned vectors
    VectorMask<Integer> mask = IntVectorizedQuickSort.lowerOverflowMask(cachedNumTrues);
    IntVector rotated = rearranged2.rearrange(IntVectorizedQuickSort.rotateRight(cachedNumTrues));
    IntVector merged1 = cachedVector.blend(rotated, mask);
    IntVector merged2 = rotated.blend(cachedVector, mask);

    int totalTrues = cachedNumTrues + numTrues2;
    if (totalTrues < SPECIES.length()) { // merged2 contains only elements greater than the pivot. Swap it to the end of the array and continue
        cachedVector = merged1;
        cachedNumTrues = totalTrues;
        upperBound -= SPECIES.length();
        IntVector newData = IntVector.fromArray(SPECIES, array, upperBound);
        newData.intoArray(array, index2);
        merged2.intoArray(array, upperBound);
    } else { // merged1 contains only elements less than the pivot. Move forwards
        cachedVector = merged2;
        cachedNumTrues = totalTrues - SPECIES.length();
        merged1.intoArray(array, index);
        index += SPECIES.length();
    }
```

The rest is just edge cases.

### What about a non-in-place version?

The sorting algorithm I describe above is an in-place algorithm, so it requires no additional memory. If we allow ourselves O(n) extra memory, we have a simpler algorithm. Here, the critical loop looks like:

```java
int leftOffset = from;
int rightOffset = 0;
for (int i = from; i < upperBound; i += SPECIES.length()) {
    IntVector vec = IntVector.fromArray(SPECIES, array, i);
    VectorMask<Integer> mask = compareTo.compare(VectorOperators.GT, vec);
    IntVector matching = compress(mask, vec);
    IntVector notMatching = reverse(vec);
    matching.intoArray(array, leftOffset);
    notMatching.intoArray(buffer, rightOffset);
    int matchCount = mask.trueCount();
    leftOffset += matchCount;
    rightOffset += SPECIES.length() - matchCount;
}
```

In words, we're moving the lower elements into their final position and the upper elements into a buffer. Once we're done processing, we can copy the upper elements into their appropriate place. This algorithm is somewhat faster, to the tune of 10 cycles per iteration (~1 cycle per int per partitioning operation), although it appears to be much faster only on larger arrays.

# What's performance like?

Great! YMMV, but on my Zen3 desktop (using AVX2 with no AVX512 functionality), I saw the following results:

```
Benchmark               (length)   Mode  Cnt           Score          Error  Units
jdk              8  thrpt    5   170714899.546 ±   1763875.997  ops/s
jdk             10  thrpt    5   154264407.313 ±    500819.241  ops/s
jdk            100  thrpt    5    69536931.152 ±   5248268.260  ops/s
jdk           1000  thrpt    5    46402055.294 ±    174077.215  ops/s
jdk          10000  thrpt    5    34134138.728 ±    188221.976  ops/s
jdk         100000  thrpt    5    28013585.553 ±     49562.094  ops/s
jdk        1000000  thrpt    5    23439276.552 ±     21704.859  ops/s
buffered         8  thrpt    5  2426909167.588 ±  14904754.115  ops/s
buffered        10  thrpt    5   200646786.158 ±  46948027.100  ops/s
buffered       100  thrpt    5   107955736.666 ±    968960.902  ops/s
buffered      1000  thrpt    5    89037645.967 ±   1329267.539  ops/s
buffered     10000  thrpt    5    79148716.685 ±    105261.243  ops/s
buffered    100000  thrpt    5    70237129.982 ±    149517.060  ops/s
buffered   1000000  thrpt    5    65537322.850 ±   3437058.994  ops/s
inplace          8  thrpt    5  2507547203.755 ± 383141136.258  ops/s
inplace         10  thrpt    5   215032676.207 ±    320366.428  ops/s
inplace        100  thrpt    5   114147458.514 ±    843670.046  ops/s
inplace       1000  thrpt    5    77580105.869 ±  10698901.290  ops/s
inplace      10000  thrpt    5    66624691.684 ±    197844.229  ops/s
inplace     100000  thrpt    5    54046743.796 ±   4568646.180  ops/s
inplace    1000000  thrpt    5    41139103.464 ±    464187.831  ops/s
hybrid           8  thrpt    5  2495687921.669 ± 186468040.821  ops/s
hybrid          10  thrpt    5   227652656.154 ±    382682.821  ops/s
hybrid         100  thrpt    5   108415081.740 ±    274307.588  ops/s
hybrid        1000  thrpt    5    92006274.816 ±    389174.105  ops/s
hybrid       10000  thrpt    5    82193216.008 ±     84009.870  ops/s
hybrid      100000  thrpt    5    74267051.943 ±   9385926.923  ops/s
hybrid     1000000  thrpt    5    69021089.102 ±   7740152.825  ops/s
```

The units are 'ints sorted per second', so 10M with a length of 1M means that we can sort 10 arrays per second. Here, `jdk` is the standard `Arrays.sort` whereas `inplace` uses solely in-place partitioning, `buffered` prioritizes sequential access at the cost of more memory moves, and `hybrid` uses buffered for larger partition operations, inplace for smaller ones. There's a nice SIMD speedup at every level, with large arrays in the region of 3x faster than the JDK method and a 2x improvement on all but the smallest arrays.

# Problems

Programming against the vector APIs without shooting myself in the foot was a huge challenge. Getting to the scores achieved required me to be comfortable with a variety of tools including:

1. Java Microbenchmarking Harness (JMH) (used to run benchmarks).
2. Java Flight Recorder/Java Mission Control (used to check for any stray memory allocations from IntVector objects that weren't optimized away).
3. Linux Perf (to check if my improvements actually led to the number of cycles dropping).
4. Java's ability to print its compiler output as assembly (to see what's actually going on under the hood).

This took a while - it took me many iterations to get the algorithm to good, I learned a bunch about what works and what I found challenging and feel that I know how Java works considerably better.

## Inlining can optimize code, but not memory layouts!

Our single vector partition operation ends up with this chunk:

```java
v1 = v1.rearrange(compressions[(int) mask1.toLong()]);
```

If we unbox all the object references, it looks more like:

```java
v1 = v1.rearrange(compressions[(int) mask1.toLong()].array)
```

so we have a non-ideal pointer follow. Worse, the mask is stored in memory as a byte array, but in order for the rearrange instruction (`vpermd`) to properly operate, the mask must be encoded as an 8-element vector, too, which means it must be reformatted.

So the sequence of instructions we end up with looks like

```
movabsq ; load the reference to the mask stored in compressions
vmovq ; load the (64-bit) shuffle into a 128 bit register
vpmovzxbd ; expand the 8-bit numbers into the 256 bit register, ready for the permutation
vpermd ; actually run the permutation operation
```

The part that is sub-optimal is the `vpmovzxbd` instruction. This instruction takes 4 cycles on my processor. If we could store the bytes pre-unpacked in RAM (since the overall amount of memory is small) we could have the instructions look more like:

```
vmovdqu ; just straight up load the relevant offsets
vpermd ;
```

My suspicion is that the vector APIs are probably more optimized towards numerical operations like matrix multiplication, and less towards integer bit twiddling type approaches, which is why we see lookup tables underperform. My suspicion was that a `VectorShuffleList` or `VectorMaskList` could be helpful for this purpose. To test this approach I [modified the JDK](https://github.com/j-baker/jdk/commit/700679eabdf4dfa1af387e658961655713c025a8) to add a new `VectorOperators.PERM` which does not require the boxing. Hacky, yes, but required only 48 lines of diff to prove the concept.

The performance improvement varied. In the tightest loops, there was very little change. In these cases, the instruction count went down considerably, but the cycle count did not change, indicating the processor simply picking up slack. When ILP is already high and we're bound by execution ports, we see a much bigger difference. When applied to in-place sorting, the performance difference was around 10% overall, which is a significant number of cycles.

I had the same problem with masks, but it turned out to be simple to recompute the mask on each iteration.

## Length/validity checks slow everything down (especially vector shuffles)

There's a parallel between this issue and the previous one. As is usual in Java, various validity checks occur at runtime. In usual cases, solid CPU engineering (superscalar execution, branch prediction, etc) means that these checks do not massively affect runtime. However, shuffles have extra validity checks which are always performed. Specifically, running

```java
vector.rearrange(someShuffle);
```

is equivalent to

```java
checkState(!someShuffle.toVector().lt(0).anyTrue()); // make sure that none of the shuffle indices are 'exceptional' aka negative.
vector.rearrange(someShuffle);
```

and this is expensive in a tight loop because it adds a non-trivial number of instructions. In fact, the cost is serious enough that there is a system property (`jdk.incubator.vector.VECTOR_ACCESS_OOB_CHECK`) which when set to 0 disables these checks. With the checks, it takes an average of 12.14 cycles and 50 instructions to partition a single vector. Without, it takes 7.7 cycles and 32 instructions. Unfortunately, disabling the checks is unsafe; out of bounds accesses can trigger segfaults.

This is another example where the type hierarchy used hurts the implementation. With a shufflelist or similar, the validation could occur at construction time.

## `@ForceInline` for thee but not for me?

For the small cases (<=8 elements) I implemented a sorting network using a single vector. As a part of this implementation, I implemented a 'compare and exchange' method as follows.

```java
IntVector compareAndExchange(IntVector vector, VectorShuffle<Integer> shuffle, VectorMask<Integer> mask) {
    IntVector vector1 = vector.rearrange(shuffle);
    return vector.min(vector1).blend(vector.max(vector1), mask);
}
```

Here, the shuffle and mask have been precomputed - shuffle connects the lanes as necessary, and mask selects the lower or higher outputs as necessary, leading to a method that might look like:

```java
static void sort16(int[] array, int from) {
    IntVector v0 = IntVector.fromArray(SPECIES_16, array, from);
    IntVector v1 = compareAndExchange(v0, COMPARISON_16_1_S, COMPARISON_16_1_M);
    IntVector v2 = compareAndExchange(v1, COMPARISON_16_2_S, COMPARISON_16_2_M);
    IntVector v3 = compareAndExchange(v2, COMPARISON_16_3_S, COMPARISON_16_3_M);
    IntVector v4 = compareAndExchange(v3, COMPARISON_16_4_S, COMPARISON_16_4_M);
    IntVector v5 = compareAndExchange(v4, COMPARISON_16_5_S, COMPARISON_16_5_M);
    IntVector v6 = compareAndExchange(v5, COMPARISON_16_6_S, COMPARISON_16_6_M);
    IntVector v7 = compareAndExchange(v6, COMPARISON_16_7_S, COMPARISON_16_7_M);
    IntVector v8 = compareAndExchange(v7, COMPARISON_16_8_S, COMPARISON_16_8_M);
    IntVector v9 = compareAndExchange(v8, COMPARISON_16_9_S, COMPARISON_16_9_M);
    IntVector v10 = compareAndExchange(v9, COMPARISON_16_10_S, COMPARISON_16_10_M);
    v10.intoArray(array, from);
}
```

I found performance of this strangely poor until I discovered the reason - the `compareAndExchange` method that I'd written wasn't being inlined, and so the `IntVector` objects could not be optimized away. Each run of this method creates many objects, many int arrays, and in general it just causes a lot of garbage collector pressure due to how hot the method was. Inlining `compareAndExchange` manually worked and restored good performance, but is ugly.

In general what I'd be concerned about is that without some mechanism of doing so, creating performant reusable code is very difficult because there's no clear way to combine code and ensure it's properly inlined at the user level without very carefully testing it. Even with the manual inlining, very occasionally Java will allocate objects on the very last step of the sorting network.

My suspicion is that it would be easier to use these APIs performantly if the inlining threshold were higher when vector methods are used.

## Masking loads and stores added memory allocations

In the above 'sort16' implementation there is no 'to' - ideally this would allow us to sort _up to_ 16 elements. What I discovered that this was a risky proposition because again it became an easy deoptimization. If you are trying to load a smaller number of elements into a vector, you can do something like:

```java
VectorMask<Integer> mask = VectorMask.fromValues(SPECIES, true, false, false, false);
int[] array = new int[] {1, 2, 3, 4};
IntVector vector = IntVector.fromArray(SPECIES, array, 3, mask);
assertThat(vector).isEqualTo(IntVector.fromArray(SPECIES, new int[] {4, 0, 0, 0}));
```

This code works, but behind the scenes this will allocate an int array (reading the code, it looks like the mask's validity is checked by creating a mask, and this is the culprit of the allocations), increasing GC overhead. In the end I observed that in quicksort, if you're trying to sort elements 3 to 5, it's equally safe to sort 3 to 11 because of the recursive nature of partitioning, and so I just oversort (and use `Arrays.sort` on the last vector in the array instead of the sorting network).

It was entirely non-obvious that this might occur, and again, this was disconcerting because the code change required to fix the method seemed so minor. There's so much scalar replacement happening when you use these APIs that small permutations that break the optimization make things a lot more challenging.

## `SPECIES_PREFERRED` considered harmful?

I primarily tested on my modern desktop, which supports `AVX2`. At a late stage, I wanted to see how well the code would work with an `AVX512` server. I used GCP and requested an Ice Lake architecture processor. The performance crashed; using 512 bit vectors in _any_ part of the code appeared to cause the throughput to drop by a factor of 3. The purpose of `SPECIES_PREFERRED` seems to be to let the user pick the widest species with hardware acceleration and not have to specialize their code for each specific architecture, but (and I didn't dig in enough here) if derating causes performance to dive bomb that much, it becomes harder to author the code reliably.

# Conclusion

Google recently published a library for running vectorized sorting in C++. This was measured at around 800MB/sec on an older Xeon processor. My processor is probably faster, and could handle around 300MB/sec on the same size arrays as compared to 100MB/sec from the standard library. Clearly there is a gap, but I think this is a reasonable outcome. Java's vector support is impressive, the implementation is interesting, and generally I had fun. I think there's some room for improvement with regards to some runtime checks, errant memory allocations, and lookup table support, but this is what preview apis are for! Code can be found [here](https://github.com/j-baker/vector-experiments).

# Appendix: Why might we want faster sorting?

Here's one simple example I encountered recently. It's common in columnar database code to 'dictionary encode' lists of values. Here, for some subset of the stored data, we store a dictionary containing the unique values that exist, assigning unique indices to each value, and we store a list of pointers into the list to represent each element.

```java
record DictionaryEncoded<T>(List<T> dictionary, int[] ordinals) {}

DictionaryEncoded<T> encode(List<T> buffer) {
    Map<T, Integer> itemToIndex = new HashMap<>();
    List<T> indexToItem = new ArrayList<>();
    // this need not be an int[] type - if the dictionary ends up having fewer than 256 elements, it could be a byte[]! There are lots of encoding schemes that help here, e.g. https://github.com/lemire/JavaFastPFOR
    int[] ordinals = new int[buffer.size()];
    for (int i = 0; i < ordinals.length; i++) {
        ordinals[i] = itemToIndex.computeIfAbsent(element, key -> {
            int index = indexToItem.size();
            indexToItem.add(key);
            return index;
        });
    }
    return new DictionaryEncoded<>(indexToItem, ordinals);
}

<T> T get(int index, DictionaryEncoded<T> encoded) {
    return encoded.dictionary().get(encoded.ordinals()[index]);
}
```

This kind of format is used by many data systems - open source projects like Apache Arrow, Parquet, Lucene, closed source projects like Vertica all contain a model somewhat like this, since it has advantages for both storage and querying. On the storage side, it's a cheap way to reduce the amount of data stored; in my example above the worst case additional memory usage amortizes to 4 bytes per element; in the case where we have a single large element being repeated over and over, it could replace kilobytes with those same 4 bytes. This usually improves both storage cost and query performance (since databases are usually I/O bottlenecked). Further, some queries may no longer require scanning all of the data (for example a query to select the max element need only scan the dictionary). Conversely, if one is able to compute a global dictionary, if the column is accessed for a join, we could join the values column and ignore the dictionary entirely. If the dictionary represents large strings, this might dramatically improve performance because the comparison operation may go from hundreds of cycles to 1.

Unfortunately, there can be some downsides. The key to high performing data processing code is usually to do as many sequential reads as possible, minimizing the amount of random memory access to regions that don't fit in the L1 or L2 caches. This is because uncached memory accesses are far slower than a processor can handle data; [an L1 cache read might be 0.5ns, whereas a main memory read might be 100ns, whereas a disk read might be into the milliseconds](https://colin-scott.github.io/personal_website/research/interactive_latency.html). In some cases, practitioners may limit the sizes of their dictionaries to compensate for this, so as to very rarely result in a main memory or disk read, whereas in others the cost may simply be eaten as an unlikely worst case outcome.

In this latter case, suppose we need to read a subset of the values, but in a way where we don't care about the order. An example of the pattern might be:

```java
class NaiveMaxAccumulator {
    final List<T> dictionary;
    final Ordering<T> ordering;

    T max;


    void accumulate(int ordinal) {
        if (max = null) {
            max = dictionary.get(ordinal);
        } else {
            max = ordering.max(max, dictionary.get(ordinal));
        }
    }

    T getValue() {
        return checkNotNull(max);
    }
}
```

where some other code is responsible for passing us those ordinals which are a part of the query.

We could perhaps improve this code by writing something more like

```java
class DeduplicatingMaxAccumulator {
    final List<T> dictionary;
    final Ordering<T> ordering;

    final int[] buffer = new int[1 << 16];
    int bufferSize = 0;

    T max;

    void accumulate(int ordinal) {
        if (bufferSize == buffer.length) {
            flush();
        }
        buffer[bufferSize++] = ordinal;
    }

    T getValue() {
        flush();
        return checkNotNull(max);
    }

    void flush() {
        Arrays.sort(buffer, 0, bufferSize);
        int lastOrdinal = -1;
        for (int i = 0; i < bufferSize; i++) {
            int ordinal = buffer[i];
            if (ordinal != lastOrdinal) {
                if (max == null) {
                    max = dictionary.get(ordinal);
                } else {
                    max = ordering.max(max, dictionary.get(ordinal));
                }
                lastOrdinal = ordinal;
            }
        }
        bufferSize = 0;
    }
}
```

This approach gives us two potential advantages. The first is that in the naive approach, the comparison is run linearly with the number of elements, whereas the improved approach runs it linearly with the number of _unique_ elements. If the comparison function is particularly expensive (as it might be for strings or other more complex datatypes) this might be a considerable percentage of the overall time. The second advantage is that by sorting the ordinals, we can benefit from spatial locality of reference. When we issue a small read to a storage volume, it's common for the operating system to load a larger chunk of data and cache it (e.g. if we try to read 16 bytes from our app, it might read 4096 and keep it around). If we are accessing ordinals 0, 2, 5, 9, 11 and we have laid out our data linearly, these will likely be located on the same operating system pages, reducing cost. If we are accessing ordinals 0, 1000000, 2000000, 300000, 45012, etc etc, randomly jumping around, we have no hope of achieving this.

Practically speaking, I've seen this tactic lead to an order of magnitude jump. However, it's not a pure improvement - for each element we process, we are definitely adding the amortized cost of sorting that element, and possibly removing the costs of loading and comparing the element, or otherwise possibly reducing the cost of loading the element. If these are only sometimes true (for example, if loading is cheap but comparing is expensive, if the dictionary is very large we will lose out, if the dictionary is very small but the comparison is very cheap we might also lose out).

On my computer, Java's `Arrays.sort` method costs around 50ns/element in the real world. This means that we unfortunately have a lot of scope to cause ourselves performance problems by using this approach, because in some cases the cost of sorting will dominate the cost of processing. If we could sort at more like 10ns/element, the change would almost always be worthwhile.