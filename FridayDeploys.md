# Who's afraid of Friday Deploys?

_September 2026_

How often have you read or heard someone say "We have a rule to never deploy on Fridays. This is a best practice!"

I do not really agree. It's a practice, it might in context be a good practice, and it can be a process improvement. But don't mistake it for always the "best", the pinnacle for all people.

See [Charity Majors, 2019: Friday Deploy Freezes Are Exactly Like Murdering Puppies](https://charity.wtf/p/friday-deploy-freezes-are-exactly-like-murdering-puppies)

## Accelerate deploys

I will start by referring to [the _Accelerate_ book](https://www.goodreads.com/en/book/show/35747076-accelerate) (Nicole Forsgren, Jez Humble, Gene Kim, 2018).

I take it as given that large, infrequent deploys are a bad thing. Very frequent deploys have higher productivity and lower defect rate.

But "guardrails" are needed: Rigour to deployment.

The issue is that there are many kinds of guardrails. What comes to mind first? Different things to different people.  Some "guardrails" slow things down, some do not.

Some read that as "It needs sign-off" after manual test by an external team. With the inevitable slowdown and batching.

some read that as "It needs automation", preferring test coverage and automation that makes deployment faster and simpler.

Neither sees the other as having actual rigour.

**But short cycle time is itself a key guardrail. All other things being equal, prefer additional guardrails that keep the cycle time low, or actively lower it**.

Waiting two days for a signoff from a person who isn't close to what the software actually does, doesn't seem like it's worth the delay to me. Waiting five minutes for  100 test cases to run does seem worthwhile.

We know that larger, less frequent release batches cause issues, and if the organisation's only response to that is to add in process that also takes time - sign-off, manual test cycles, "staging soak", then that delay compounds the problems that it aims to solve. It is digging the hole deeper, moving in the wrong direction. The doom-loop sets in.

"If it hurts, do it more frequently" is an agile saying for good reason, and it applies to deploys as well. You need to "grease the wheel" to make it painless. Not stop because of the pain.

This means that automation must be preferred over "wait for sign off" guardrails.

It is never good to increase cycle time without a very good reason. Yes, rules are "a discipline", "a guardrail", but so is better faster test and deploy automation. The same goal, exactly opposite ends. An engineering not management fix to an engineering not management problem.

## Maturity level

I sometimes say, only partly joking that there are 3 maturity levels of deployment.

### Level 0

No process. We can deploy at any time, and we do. Oops, we caused an outage. Oh well, YOLO.

### Level 1

Process. To  prevent costly outages, we have put down rules on when we can deploy. This is where the "No friday deploys" rule lives.  It is a "better practice" than level 0 when the organisation has come to depend on the system being up and running.

### Level 2

Invisible process. We can deploy at any time, and we do. Our automation is good enough that deploys have been de-risked. We have done the work and have the discipline. Deploying at the end of day or week has the same very low risk as any other time. The rules and the consequent increase in cycle time are no longer needed or beneficial.

We can safely deploy at any time, and we do.

_A one-sentence description of level 0 and of level 2 can sound the same._ And on a good day the result is the same! But the process is quite different, in the work that has gone into building tests and tools that execute during the level 2 deploy, and halt it when a potential issue is detected.

## Automation up the ladder

Automation is mostly tests of various kinds, but also covers automated blue-greening, monitoring and automated rollback.

If your organisation is at level 1 and you are telling management "deploy more frequently" you run the risk of being seen as a cowboy who wants to go down to level zero, a risk-taker who wants outages, rather than pushing for productivity and quality. The idea that short cycles and small batches are just safer and more productive is still not widely accepted.

Code freezes are not good. They are sometimes necessary, e.g. for a shopping site on the busiest day of the year. But this should be limited and infrequent.

This applies to "deploy once per week" being better than "deploy quarterly". But it also applies to "deploy today" being better than "don't deploy today".

The end point of course is to deploy one change at a time as soon as it's ready, and monitor afterwards. And then work on making changes right-sized - smaller Pull requests merged more frequently! too often they're just too large and to infrequent.  

Deployment Risk is in no way limited to one day of the week. What makes deploys on Thursday safe? De-risking deploys by avoiding a specific day is just superstition.

No test automation is perfect, that's why level 1 can contextually be a sensible option. But you can decide if your automation is good enough to de-risk deploys. You can also move the dial on lowering the risk.

If you're not deploying new code on Friday, why not work on your tests on Friday? Make your deployment verification wider, faster, more robust, etc.

Nobody's tests are perfect, nobody's test coverage is perfect. The question is, it it good enough? It's not about the absence of issues, it's about what do you do when there is an issue - do you increase the cycle time or do do you look for a way to de-risk the change and automate the check? Whichever you choose, you are going to do again.

Fear of deploying on Friday where you are may be justified, but it _is not_ a signal that "Friday deploys are a bad practice", for everyone, always, fullstop. It is a signal that you may not have the maturity of automation to do it safely. It would be mistake to elevate your lack of capability to a universal principle.

So, work on that capability instead.

This is not about "low-risk front-end" teams having more latitude to deploy, except in that they might be fortunate in needing less effort than others to get to level 2. Even crucial back-end teams can get there.

## The opposite of rules

The opposite of "A hard rule of no friday deploys" isn't "Just go live with a huge feature late in the day, and head off to the pub" although that is a common straw man.

No, the opposite of "That's a rule" is "Use _judgement_. Do so at your _discretion_." The capability that "you _can_ deploy on Friday" is not "you _must_". It is completely compatible with "I will merge this minor update to metrics on Friday morning, so that we have good data by Monday" and also "This big feature? Let's hold off on that until Monday".

This is scary as it requires trust.

But there is something liberating in looking a person in the eye and saying "Yes, I chose that my update went live today, just like any other day. It's Friday, what of it? Our process is robust. We have the discipline to do this, do you? Why are you afraid, and what are you doing about it?"

But once you achieve Friday deploys, your work to keep the deploy safe and productive is ongoing. Then some myopic "best-practice" merchant will confidently and incorrectly tell you that you're doing it wrong, and that they chose better, because they do not have confidence in a specific time of week only.

## Obligatory Mention

This is becoming ever more relevant currently, as senior people are ever more hypnotised by LLMs, and have the simplistic belief that "whatever the problem, AI is the tool for it". Continuous delivery does not flow from a chatbot. Proven best practices should not be exchanged for a chatbot.
