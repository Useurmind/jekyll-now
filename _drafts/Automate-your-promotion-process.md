---
layout: post
title: Automate your promotion process
tags: rfluxx routing stores nested hierarchy list
---

In this Post I want to discuss the automation of promoting your software through different environments. From your development environment over e.g. QA/staging to production.

## The tradition

Traditionally people let developers do whatever they want on the development environment. At some point someone will deploy a software package to staging be it by hand, or scripted or automated via a CI/CD pipeline. Then a lot, or just a few people will have a look at the quality of what was deployed. This usually comes in the form of exploratory testing. Believe me when I say that you can even test APIs exploratory. Often no structured tests are applied. In the worst case no one will even care and everything is accepted for deployment into production without any checks.

After that the deployment to production will be performed, hopefully in the exact same way as the deployment to staging, again either by hand, by script or automated via a CI/CD pipeline. But again nobody will perform any meaningful tests here. Perhaps some smoke or exploratory tests. The thinking here usually goes like "come on we already tested staging and it worked and its boring". That does not mean people are to blame for this thought. I did my share of this type of testing too and the quote above perfectly reflects my thoughts after the second or third time I am testing the exact same thing.

Finally the first real tests are performed. The actual users of the system start using production. And believe me when I say that they will care if something does not work.

## Escaping the cycle

The problem of escaping this approach is also the following. You are developing software and you need to test it at the end of your sprint (if you are working in SCRUM). So for example the proportion in the first sprint may be 90% development and 10% testing. In the next sprint you would have to test the stuff from the first sprint too so it becomes 80% development and 20% testing. The next sprint will add more stuff that needs to be tested and the cycle repeats until development comes to crawling hold. 

Don't fret about the numbers too much but your development will become slower as your software becomes richer and richer in features. 

One way to escape this is to not let the developers test. There are reasons to have separate testers in your team like a different approach of using the software and less prudence when testing it. But in my opinion you should involve developers in testing because it makes them see posibilities to make their work better. Not involving them in testing will take away one feedback channel from which they are able to grow. Also the usually one or two testers will never be able to fully test the software either. And lets say you have a development team of around five developers. Then you will not be able to afford more testers to accomodate for the increasing numbers of features to test.

The question now is how to involve them in testing, and how to not block more and more of your development capacity for testing. And the magical answer you probably already guessed is __automated testing__. Automated testing will make the development slower by a certain percentage lets say 50%. But in constrast to manual testing that will eat up almost all of your development capacity over time the share of time for automated testing stays constant at 50%. You will be able to keep your pace. And you will be able to keep the quality high. Win and win I would say (by the way I think the 50% can be more or less accurate in case you wondered).

## New possibilities

This is no post about automated testing but still I wanted to point out how important automated testing is. And also the automated tests give us some new possibilities that we could not see before. 

There is this thing called approval during deployment. A manager needs to approve that a deployment to staging or production may be done. He needs to take responsibility for the work of the developers that he cannot even fully judge. So he delegates the judgement to testers. They too cannot fully judge the work because it builds up over time. So in the end an automated test suite judges if the software is working.

But if the automated tests judge if the software is working why not let them decide whether to promoto the software to the next stage. That is the core idea of automated promotion. 

Now one could say: "I do not trust the test suite to be accurate". Well trust builds over time. So if you do not just start with automating promotion you probably never will. You can always invest in making the test suite better if you encounter problems. And do you really trust a person to test a software that was developed over two years by 5 developers in two days? Even if the same job could be done by a test suite in 30 minutes?

Also compare the following: investing in the test suite gives you a permanent increase in test speed for a fix amount of money. Investing in a tester gives you temporary increase in test speed for the same amount of money. Lets say a developer needs one day to automate a test that takes 20 minutes to perform. That means the tester could perform this test 24 times. If you have two week sprints that means you can pay the tester for a year before the odds turn in favour of the automated test. Obviously these numbers are totally made up, but the idea stays the same. In the long run you will benefit from automated tests.

If a meanigful decision is not automated in your promotion process then you probably need to think about how to automate it.

So hopefully you come to the same conclusion as me: an automated test suite will do the testing faster, more accurate and cheaper over a long period of time. Therefore, it should be the main decision-maker about promotions of software between stages.

And if you are the manager of a project, let me tell you: you don't loose your job! Just make sure the developers spent enough time creating proper tests for your software. Because if they don't you will either pay in development speed or quality in the long run.

## Strategies/concepts for automated promotion

Every project is different. That is a core rule I learned over the past ten years. I actually never encountered the same in most cases not even a similar setup in two projects. So it does not make sense to explain an exact technical implementation of an automated promotion. Therefore, I would rather like to talk about strategies and concepts that can be employed to solve common problems in the automation.

### Deployment Artifact

### Bootstrapping

Every automated promotion process needs a way of bootstrapping it. The simplest form of bootstrapping is a push to a git repository. The push will trigger a pipeline that will lead to promotion

### Stage Diff

