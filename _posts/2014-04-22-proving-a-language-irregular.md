---
layout: post
title: Proving a language irregular in a nontrivial way
---

Everyone knows that regular expressions (at least in the theoretical sense) have almost immediate limits on what they can recognise, though this is [not always realised by users](https://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags/1732454#1732454). The problem is that the presented proofs of languages that are not regular are typically those that are easily provable. This is because

- It is easy to present a simple use of the pumping lemma in a 1 hour lecture than anything in depth, and
- most of the reasons why regular languages are taught can be summed up in the well bracketing and $a^nb^n$ proofs - they prove that you can't parse programming languages with regular expressions.

The challenge
-------------
I found this problem on Reddit, around 9 months ago. I had a quick search for the source, but with no luck. It was set to a redditor as a homework question, by a professor who mixed up his nouns. As I recall, the professor realised his mistake and changed the question to an easier one.

*Is the language consisting of numbers encoded in base 3 for which the base 2 representation has even parity regular?*

Here, even parity means that each number has an even number of 1 bits in its binary representation.

Standard approaches
-------------------
### The Pumping Lemma
The classic two approaches are the Pumping Lemma, and the Myhill-Nerode theorem. As far as I can work out, the pumping lemma is known by all, whilst Myhill-Nerode is known by fewer.

The problem here is that (at least in my case) it's difficult to reason about whether pumping actually produces a string in the language (or not). Generally one would hope to find a word known to be in the language, but which when pumped must produce something not in the language. In my case, the nearest I could get was something of the form $3^n$, which empirically proved to be in the language about 50% of the time.

This at least implies that if I picked a random word on the language and pumped on it, the probability that the word was in the language and its pumped $3^{n+m}$ form was not would be around 0.25 (and no obvious pattern could be seen in the results). Of course, this proved nothing, but was a useful heuristic going forward - I could assume that the language would not be regular. I admit that I find this argument mostly convincing, but clearly it is not a rigorous proof at all.

### Myhill-Nerode
Broadly speaking, you just get to the same issues as the pumping lemma.

### Generating functions
Another approach is to use the generating function of a regular language. If we let $s _ L(n)$ be the number of words of length $n$ in $L$, then the ordinary generating function is defined by
$$S _ L(z) = \sum _ {n \geq 0}s _ L(n)z^n.$$
It is known that the generating function of a language $L$ is a rational function if $L$ is regular. By rational function, it is meant that there exist polynomials $P$ and $Q$ such that
$$S _ L(z) = \frac{P(z)}{Q(z)}.$$

Let us consider the number of words of each length in this language. In binary, for a word of length $n$, exactly half of all words of a given length will be in the language. Assuming that the numbers in the language are evenly distributed, this will be the case for the ternary representations (and again, empirical evidence implied this to be true). So, this is clearly rational, and again is not a witness to the irregularity of the language.

## Cobham's Theorem
Clearly at this point I wanted to try and reason specifically about the structure of the sequence. The [OEIS page](https://oeis.org/A001969) gives great context to this, and in particular it references the [Thue-Morse sequence](http://mathworld.wolfram.com/Thue-MorseSequence.html), the characteristic function for the natural numbers not in this set. This is very interesting, as the Thue-Morse sequence is [overlap-free](http://mathworld.wolfram.com/OverlapfreeWord.html).

This means that the set $S$ that we want to recognise in base 3 cannot be eventually periodic - that is to say, there cannot exist $C$ and $k$ such that for all $x \geq C$, if $x \in S$ then $x+k \in S$ also.

Thankfully, it turns out that Cobham (of Cobham-Edmonds thesis fame) at one point proved the theorem

*Let $r, s \geq 2$ be multiplicatively independent bases. A set $X \subseteq \mathbb{N}$ is $r$- and $s-$recognisable (recognisable by a DFA in bases $r$ and $s$) iff $X$ is a finite union of constants and arithmetic progressions.*

The great news here is that bases 2 and 3 are multiplicatively independent (for no $k, l$ is $r^k=s^l$ where $k, l \in \mathbb{N} - \\{0\\}$).

This means that if the language is to be recognisable in base 3, it must be a finite union of constants and arithmetic progressions.

Here follows a simple proof that such a language must be eventually periodic (there exist $P$ and $C$ such that for all $x\geq C$, if $x$ is in the set then so is $x + P$).

Suppose that our language is defined as the numbers in some set
$$\\{c _ 0, c _ 1, \dots, c _ n\\}\cup \bigcup _ {i \leq N}\\{x | \exists n\in \mathbb{N}.x = a _ i + nb _ i\\}.$$ Now consider the number $K$, defined to be the least common multiple of $b _ 0, b _ 1, \dots, b _ N$. The language must at least be periodic with period $K$, because $K$ is some integer multiple of each $b _ i$, and so is periodic with period $K$ for all $x \geq \max\\{c _ 0, \dots, c _ n\\}$.

This means that if the sequence is to be recognisable in base 3, it must be eventually periodic. As noted above, for each number $n$ in the language, Thue-Morse$(n) = 0$. It is well known that the Thue-Morse sequence is overlap-free, and so the language cannot possibly be regular.

## Conclusion
We're done. I thought lightly about this exercise for a fair while, but only recently had the motivation to actually follow it through to the end. I (broadly speaking) wrote this up as a description of how proving languages irregular can be more involved and indeed interesting than simply invoking the pumping lemma and bashing through a few lines of answer, as was required of me in my course. I wrote this mostly as a quick after-action report, but I hope it reads well.

### Acknowledgement
After getting stuck, I pointed my search towards [CompSci StackExchange](https://cs.stackexchange.com/questions/23944/proving-a-language-irregular-standard-methods-have-failed). I was stuck, and I didn't have a good enough grasp of the literature to know where to move next. The result was essentially a great advertisement for StackExchange - [Jeffrey Shallit](https://cs.uwaterloo.ca/~shallit/) who answered my question has written papers about the Thue-Morse sequence and written a book about the sequences recognisable by finite automata - as far as I can work out he was one of the best placed to help me anywhere. The proof presented is his, not mine - I got to exploring the properties of the Thue-Morse sequence, and have merely found the proof interesting enough to write about here.
