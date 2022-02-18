---
title: "Learning Github Pages (Learing Lab)"
date: 2022-02-18
---
Despite being in tech for the last 15 years, I've always worked on back-end systems for organizations that didn't *encourage* open source and community participation. 
As such, on **"Day 5 of 21 between roles"** I wanted to find a place to capture some publically shareable technical thoughts, both on things I'm experienced in as well as 
those things I'm learning from scratch. This post walks through why I chose Github Pages as the technology, the learning experience and a light inspection of the Github Actions tooling that powers both the static website build and the learning labs.

## Options I considered for Blogging solutions
With that in mind, and with no interest in looking for an optimum solution, but rather an acceptible one that I could make a quick *( < 10 minute)* decision on. I looked through a few options:
1. **self-hosted** - Maximum control, but also a more significant maintenance commitment. Last time I built a website was for my wedding in 2014. It's great for learning
new rendering stacks and rebuilding *front-end empathy* on *"is CSS cross-browser and form-factor still a nightmare"*. However neither of those were a goal for me today.
2. **Facebook pages** - looking through tutorials, [such as this one](https://startbloggingonline.com/facebook-page-for-a-blog/), these looked nicely low barrier / accessible 
and would drive me to engage more with a product that I'll need to understand professionally going forward, however are *(unsurprisingly)* very much designed for non-technical
content. The idea of having to do code snippets as screenshots in order to get appropriate markup was not appealling at all.
3. **Github pages** - pages written in [markdown](https://www.markdownguide.org/basic-syntax/) with great support for code, and static webpage generation and hosting are both
huge pluses. Having to do changes via a [Distributed Version Control System](https://en.wikipedia.org/wiki/Distributed_version_control) (DVCS) seemed a little overkill, 
given  the grand total of contributors will be between 0 and 1 people.

Given refreshing on tech stack learning that self-hosting would provide wasn't of great interest to me this time around, my lightweight and distinctly non-exhaustive consideration
of the space, I decided that Github pages was the middle balance between being technical depth and content focus.

**NOTE** I've know I've ommitted many sensible, perfectly reasonable choices; such as the classic [Wordpress Blog](https://wordpress.com/blog/), [Medium](https://medium.com/), etc - the main reason for choosing
the tiny subset I did was *"tangential but serendipitious learning"*; I want it to be separate from my day job - but still develop my learning in a way that has a non-trivial
chance of providing indirect benefit to my day job.

## Learning Github Pages via the Learning Lab
Getting started with Github pages was very easy, whilst I'd normally learn [based on documentation](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll), my top google 
result pointed me to the interactive [Github Pages Learning Lab](https://lab.github.com/githubtraining/github-pages) format. This format consists of guiding the user/developer
through the steps of the project via interactions with the artifacts you generate.
1. **upon generation of the project** - the lab agent [creates an Issue](https://github.com/TheGrimTiffith/github-pages-with-jekyll/issues/1) with details of the next activity; 
changing the project settings to enable Github pages for the site
2. **upon changing the setting** - the lab agent [adds a new comment](https://github.com/TheGrimTiffith/github-pages-with-jekyll/issues/1#issuecomment-1044977593) to the Issue, with the
next set of instructions. 

The lab proceeded this way until you have a working blog in < 15 mins, which was pretty awesome. However there *were* a few bumps in the road for me ... as the comment and commit history shows:
* **asyncronicity** - some of the steps take non-trivial time and have no sense of progress built in, in particular once the **Source Root** does it start working to publish the
website straight away, or need something else? How long does it *take* to publish? Can I *view* the progress in a deployment pipeline or similar?
* **Github pages settings** - ``Settings > Github Pages`` may have changed since the tutorial was written, it's *unclear* whether or not the **Theme Chooser** is *required* 
or *optional* and the lab instruction doesn't comment on ``theme`` at all until the next instruction, which is only posted upon the setting being changed.
* **Where are my posts?** - As a result of fumbling the start of the tutorial, when I finished - my blog looked *even more bare than intended* - it was lacking the posts from the 
``_posts`` folder. A quick search yielded a [post with response from Github staff](https://github.community/t/why-the-github-page-doesnt-show-my-post/13277/11) from an earlier (2019)
issue with the lab, and after updating the theme *back* to ``theme: minima`` and *patiently* hitting ``Ctrl + F5`` for what felt like a minute; **Success**!!
* **Formatting transclusion** - finally I learned the format of the ``minima`` theme by trial and error, namely removing the duplicative titles for both the ``index.md`` and each blog post!

## How does it work?
Github provides an ``onEvent`` style callback mechanism call **[Github Actions](https://docs.github.com/en/actions)** for a [wide range of events](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#available-events) *(e.g. Code is pushed, Issue is Created, etc)*. These are used both for the deployment of Github Pages *and* building the Learning Lab transitions. It's worth taking a peek under the covers of both of these to learn a little more.

### Deployment of the Webserver
The webserver uses an action on content change to trigger a build and deployment of the static website. It doesn't appear the workflow code for this is held locally, or if it is it's in some form of hidden folder that I haven't worked out how to make visible, but we can gleen a few interesting tidbits:
* here we can see a [workflow run from the blog update](https://github.com/TheGrimTiffith/github-pages-with-jekyll/actions/runs/1866164169) was triggered by the **githubpages-bot** and took 46 seconds from trigger point to completion. 
* Given a build time of 19s and a deployment time of 5s, it's unclear where the other 22s went. A reasonable guess is this internal mechanics of the bot and/or Github Actions such that we're witnessing *"polling interval"* and/or *"queuing to be scheduled to build resources"*, however these are very much guesses from experiences with other [Continuous Delivery](https://en.wikipedia.org/wiki/Continuous_delivery) (CD) solutions.

### Github Learning Lab workflow
Whilst the code isn't directly linked from the Learning Lab workflow *page* - the page name of ``githubtraining/github-pages`` fits the archetype of ``<user|organization>/<respository-name>`` and so, sure enough the code can be found at https://github.com/githubtraining/github-pages/. 

looking at the project we can see the following:
1. The actions are defined in the [config.yml](https://github.com/githubtraining/github-pages/blob/main/config.yml) file. I was hoping to [use the Actions visualizer](https://docs.github.com/en/actions/monitoring-and-troubleshooting-workflows/using-the-visualization-graph) to look at the flow visually, but [no actions show up for me](https://github.com/githubtraining/github-pages/actions). Given this doesn't follow the [entry level explanation](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions#understanding-the-workflow-file), but rather starts with step declarations and has a seemingly novel concept of ``before`` I'm expecting there is a common framework shared across all the github training pages.
2. There are two supported *localizations* of the tutorial; English (EN) and Spanish (ES)
3. There is no code outside the workflow definition in the config.yml - the [responses folder](https://github.com/githubtraining/github-pages/tree/main/responses) contains a series of response payloads written in markdown, with a subfolder for the spanish language versions. 

From a no/low code perspective this is pretty impressive; 296 lines of Yaml all up! As per #1 though I would suspect there has to be another extension written that's common / re-used for github learning. This is worth some further digging.
