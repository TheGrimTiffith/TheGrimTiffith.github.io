---
title: Wordle - Done Quick
date: 2022-02-19
---
The horrors of Wordle 243 required going back to look at the solver, but before we can make it smarter - we first have to make it run *faster*, as the current implementation can't
even run using just the dictionary of ~2K **answer** words, let alone the ~14K guess words without running out of memory. There is a lot of work to do, and it's my favorite kind of
work - performance work!

## A quick aside on Wordle, solvers and why bother - it's already solved?
At this stage [Wordle](https://www.nytimes.com/games/wordle/index.html) likely needs no explanation, but just in case. Wordle is a game where you attempt to guess the hidden 5 
letter word in 6 guesses or less. The letters in each guess word are scored against the hidden word where Green = letter is present and correct, Yellow - letter is present, but 
in the wrong place and Grey - letter is not present.

There are several great resources out there that have explained the best starting word, at least prior to [NYT taking over](https://www.nytimes.com/2022/01/31/business/media/new-york-times-wordle.html),
is **SALET**. The following are both far more comprehensive explanations of the Information Theory approach as well as brute force permutations:
* [**Information Theory Approach**](https://www.youtube.com/watch?v=v68zYyaEmEA&t=0s) - and then it's [follow-up correction / completion](https://www.youtube.com/watch?v=fRed0Xmc2Wg) that updated CRANE -> SALET.
* [**Memoized Brute Force**](http://sonorouschocolate.com/notes/index.php?title=The_best_strategies_for_Wordle) - provides exhaustive strategy files both for normal mode and hard mode
and outlines the importance of personal fitness function - i.e. do you want the best *average* score (AVG), or the *lowest maximum* score (p100)

I won't be producing anything like the quality of this work, however I do think there is a useful niche to talk through; the programming side, particularly the performance engineering.
The Sonorous Chocolate page eludes to this in it's [description of the algorithm](https://sonorouschocolate.com/notes/index.php?title=The_best_strategies_for_Wordle#Description_of_algorithm)

> By contrast, proving 7920 is the best value took about two days on a 6-core computer using the same program. (I have no doubt this can be improved, but it will still be a lot slower than the heuristic/partial search.)

**GAME ON!**

## Running out of memory
We start with a very poor implementation of the code, definately not my finest quick hack. After running for ~30s it crashes due to running out of memory:

![Tell Tale Out Of Memory Error](https://github.com/TheGrimTiffith/TheGrimTiffith.github.io/blob/main/images/wordle/out-of-memory-error-stack-trace.png?raw=true)

Unfortunately I don't have a good memory allocation profiler handy, but taking a couple of memory snapshots quickly illustrates the problem:

![Large retained memory for choices and decisions](https://github.com/TheGrimTiffith/TheGrimTiffith.github.io/blob/main/images/wordle/retained-memory-profile-prior-to-OOM.png?raw=true)

Why is it retaining *so* much memory? well because I made a bone-headed retention mistake of course! The code is structured to recursively evalute all permutations for each *scoring* given the discovered information so far, however once it's done it's search of the child space it doesn't *"take the 'best' result"* - it retains all the options, and then picks the *"best"* on read/access as shown in the following snippet ([full code](https://github.com/TheGrimTiffith/wordle-tinkering/blob/5c9984c2d5cdfd7187b93b72086a16f58fd0b6d0/src/main/kotlin/com/fafflegriff/wordle/graph/DecisionTree.kt#L141)):

```kotlin
private class ChoiceNode(choices : List<DecisionNode>) {
        val maxDepth: Int
        val allGuesses: Int
        val orderedChoices : List<DecisionNode>

... 

        fun makeChoice() : CharArray {
            // choose first result from the ranked (ascending) choices
            val decision = choiceNode.orderedChoices[0]
            decisionNode = decision
            // set as new root for the graph, ready for the result to transition
            return dictionary[decision.guessWordId]
        }
...
```

This is *extremely* wasteful. Consider there being three valid options left; ``[ teddy, theft, testy ]`` - the algorthim constructs three possible candidate trees; those of picking each of the words first, and having the remaining 2 transitions if the word isn't the hidden one as follows:

![three simple choice graphs](https://github.com/TheGrimTiffith/TheGrimTiffith.github.io/blob/main/images/wordle/three-choices.png?raw=true)

We are effectively repeating the same useful information 3 times; there is no meaningful difference to pick between which choice is best - as each choice has the same properties. It is critical to understand that this is also about the *happiest* scenario possible here for several reasons:
1. **Cardinality** - we're down to only 3 valid choices, higher up in the decision tree we'll have *several thousand*
2. **Choice Expansion** - this example is also *deterministic* in that each scoring leads to only a single answer. This is the recursive [base case](https://en.wikipedia.org/wiki/Recursion#base_case) eliminating the need to make further nested choices. The vast majority of the time scored branches will have multiple choices
that need to be evaluated.

To fix this? only retain the best choice - further than removes the need for the intermediate choice tree structure itself. This means we have a datastructure of ``DecisionNode`` representing the *player* optimal choice which will have up to 243 ``scoringTransitions`` representing the *game* providing a scoring of players guess vs the hidden word. [Removing the ChoiceNode](https://github.com/TheGrimTiffith/wordle-tinkering/commit/32af75ca608851c4ef430b4af4a58aab3d261b05) and the retention of all evaluated choices in memory, did indeed solve the immediate onset memory leak. So now we can start looking at making the code go faster.

## Building in speed measurement
Before we can even *start* doing the performance optimization, we need a good measure of the solver performance itself. After fixing the memory issue, the decision tree generator just *"runs for hours"*. This isn't particularly useful for operating closed loop performance improvement, so we're going to need to build a more scalable mechanism. Sonorous Chocolate's initial approach was to do a *non-exhaustive* search and to constrain the search to expanding the **Best N** at each choice. This way the solver can trade of *precision* for *processing time* and we can start to move towards an equal comparison of measurement.

So, how do we add in constraint to N best *prior* to evaluating them? [3Blue1Brown](https://www.youtube.com/channel/UCYO_jab_esuFRV4b17AJtAw)'s previously mentioned YouTube video had a great illustration of the distribution of results being ranked by the average *information* across all possible word scorings from the candidate guess word. It's a nice easy calculation to code up:

```kotlin
            for (transition in transitions.values) {
                val probability: Double = transition.size.toDouble() / doubleSum
                val information = log2(1/probability)
                averageInformation += probability * information
            }
            this.averageInformation = averageInformation
```
Introducing to the candidate generation code was done by splitting the candidate generation from the recursive build of the decision node, so the candidates could be sorted and
truncated to the Best N guesses to expand:

```kotlin
        // order choices and filter to best N candidates
        val filteredCandidates = if (candidates.size > maxCandidatesToConsider) {
            candidates.sortDescending()
            candidates.take(maxCandidatesToConsider)
        } else candidates

        // now recurse and build the deep inspection for each of the candidates
        val choices = filteredCandidates.map { buildDecisionNode(it.choice, it.transitions, similarityClusters) }
```

So, how are we doing performance wise? Well, many orders of magnitude off the state-of-the-art! 

| Max Candidates (N) | Best Start word | Guesses (SUM) | Guesses (AVG) | calculation time (ms) |
|---|---|---|---|---|
| 1 | raise | 8334 | 3.6 | 1921
| 2 | slate | 8158 | 3.523974082073434 | 1896
| 3 | slate | 8158 | 3.523974082073434 | 2238
| 4 | slate | 8157 | 3.5235421166306695 | 2974
| 5 | trace | 8153 | 3.521814254859611 | 4320
| 6 | trace | 8152 | 3.5213822894168465 | 7104
| 7 | trace | 8152 | 3.5213822894168465 | 12254
| 8 | trace | 8152 | 3.5213822894168465 | 20867
| 9 | trace | 8151 | 3.5209503239740823 | 33401
| 10 | trace | 8151 | 3.5209503239740823 | 51668
| 11 | trace | 8151 | 3.5209503239740823 | 73564
| 12 | trace | 8150 | 3.5205183585313176 | 103856

~104 seconds to do only the expansion on N = 12, that's a very *very* long way from the N = 25 in 17s and N = 50 in 40s! We have a lot of work to do, but thankfully we have
a baseline :)

## Slow is Smooth, Smooth is Fast? Expanding the Dictionary
We now have a measurement baseline and are coming up with the #3 equal best word, ``TRACE``, we're not only *non-functionally* deficient, we are also *functionally* deficient.
Specifically we're not coming up with the correct best word, ``SALET`` or the runner up ``REAST`` - further we're very significantly off the proven SUM of guesses for TRACE; 
we've whittled down to **8150**, but that's far in excess of the correct **7926**. This is because we aren't using the full acceptible word dictionary, we're just using the
*hidden word* answers file. In order to ensure we can get to the optimal guessing strategy, we need to be able to expand guess words beyond the list of answer words. This is a
relatively small change to the codebase, at least to do the naieve expansion:
1. change the precalculated matrix from being N x N, where N = 2315 hidden words, to M x N, where M = 12,953 input words.
2. pass the available guesswords list seperate from the remaining candidates list, available guesswords is kept at the full set of viable choices at any given decision, which is significantly wider than the set of remaining valid answer words.

Unfortunately, and perhaps unsurprisingly, this makes the processing even slower than before:

| Max Candidates (N) | Best Start word | Guesses (SUM) | Guesses (AVG) | calculation time (ms) |
|---|---|---|---|---|
| 1 | soare | 10070 | 3.4892584892584892 | 101490
| 2 | soare | 9948 | 3.4905263157894737 | 375036
| 3 | soare | 9881 | 3.486591390261115 | 88314

## Fixing the sort-merge inefficiency
when looking under the covers we can see that the time is dominated by the ``intersectSortedLists`` method and within that the vast majority of the time is spent *resizing* the internal array, as shown in the profile flame graph below:

![profile showing slow intersect sorted lists method](https://github.com/TheGrimTiffith/TheGrimTiffith.github.io/blob/main/images/wordle/profile-showing-slow-set-intersection.png?raw=true)

Whilst we could work through how to ensure the Kotlin ``ArrayList`` is instantiated to the correct length, or we could rewrite to use ``ShortArray`` types and pass a used length there is a significantly faster structure we can use, albeit at the cost of a little more memory, a ``BitSet``. Consider we only have 2,315 valid answer words, and we've already
encoded then with integer ordinals. As such rather than having a dense List<Short>, where we use 2 bytes to encode each intersection candidate, we can use 2,315 bits *(290 bytes)* to store the entire sparse incidence graph, and then run 37 consectutive 64 bit AND operations to conduct the intersection operations. We don't even have to write the logic ourselves as Kotlin has a helpful [BitSet](https://github.com/JetBrains/kotlin/blob/6a670dc5f38fc73eb01d754d8f7c158ae0176ceb/kotlin-native/runtime/src/main/kotlin/kotlin/native/BitSet.kt) SDK class:
        
        








