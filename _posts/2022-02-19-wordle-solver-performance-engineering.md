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

**GAME ON**
