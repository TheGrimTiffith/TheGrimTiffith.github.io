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

Why is it retaining *so* much memory? well because I made a bone-headed retention mistake of course! The code is structured to recursively evalute all permutations for each *scoring* given the discovered information so far, however once it's done it's search of the child space it doesn't *"take the 'best' result"* - it retains all the options, and then picks the *"best"* on read/access as shown in the following snippet [code permalink](https://github.com/TheGrimTiffith/wordle-tinkering/blob/5c9984c2d5cdfd7187b93b72086a16f58fd0b6d0/src/main/kotlin/com/fafflegriff/wordle/graph/DecisionTree.kt#L141):

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
