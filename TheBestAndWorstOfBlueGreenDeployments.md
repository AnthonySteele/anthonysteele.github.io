
# It was the best of blue-green deployments, it was the worst of blue-green deployments

Let me tell a tale. I am talking about a previous employer, no name need be mentioned here.

We worked on an API that was in production, and it processed transactions over HTTP. It was part of a key process that made money for the company, so it should _never_ be out of service. The load varied but it was never idle, there were always at least a few transactions per second, 24-7. Most transactions completed in under 2 seconds.

Meanwhile, development work on new features was ongoing, and updates due to new code happened about once a week. So how did this update happen without downtime?

The update process was simultaneously the best and the worst of [Blue-Green deployment](https://martinfowler.com/bliki/BlueGreenDeployment.html).

It was the best because it was gradual, zero-downtime, and checked, and had the ability to back out if any error was noticed.

It was the worst because of the sheer manual overhead and the resistance to automation, and the unaddressed impacts of that.

More people should embrace the best aspects of this deployment process, more people should shun the worst of it.

## The good

There were at one point 16 servers, in 2 groups of 8 each. The groups were of course named "Blue" and "Green". If a transaction failed outright with a `HTTP 500` error, the upstream gateway would retry it up to 3 times. Load was usually evenly balanced between all servers, but this was configurable. Average load could be sustained by 1 group of 8 servers.

To deploy smoothly, all load was directed over to the green servers, then when blue was drained and idle, a deployment of a new build was done to the idle blue servers. Or vice versa. Thus [we had deployment, without actual release to users yet](https://www.thoughtworks.com/en-gb/radar/techniques/decoupling-deployment-from-release).

Then tests were run on the new build on blue, and when passed, 1% of traffic was then sent to blue, and monitoring of logs and dashboards happened until we were happy with the outcome of [this canary](https://semaphoreci.com/blog/what-is-canary-deployment).

This process is relatively safe: consider the worst case ; if the new deployment is "bad" and has some defect that somehow slipped through all testing, and it is throwing errors in common cases with real transactions. Then these transactions would occur once every 2 or 10 seconds, fail, and be retried with only a 1% chance of going to a blue server again on the second retry, and same again on the third retry. It's possible to back out of a deployment before any transaction has the probability of more than an inconvenient retry delay to a few tens of transactions.

If there were no errors, load was ramped up, with a pause for monitoring after each change, and if necessary, looking for specific transactions that have the characteristics to activate new code paths. A typical ramp up sequence was: 1%, 2%, 5%, 10%, 25%, 50%  and then if all is going well, complete it: 100% load on the new code on blue, a deploy to green, test on green and balance out again at 50-50 with the new version on both glue and green.

This is IMHO hard to improve on, as a gradual, testable, safe rollout of a live system with no downtime.

But it was also terrible.

## The bad

Adding 1% traffic to blue meant an engineer entering "1" and "99" into 2 text-boxes in a well-known deploy tool, then clicking about 4 times to save, confirm, deploy, confirm. The total number of important keystrokes involved in a deployment was not small.

"Monitoring" was the people on the call with dashboards open and waiting a minute or several.

This was tedious and error prone and required keeping a lot of context in mind, with painstaking concentration. So the solution to that was to have more people on the call to check and confirm. Testing was manual too, so a Tester had to be on the call.

It typically took around 4 people more than 90 minutes to go through the whole thing.

The upshot is that releases couldn't really happen more often than once a week on average - that is, some week there were 2 releases but it would slow down again afterwards when people stopped doing extraordinary effort. There just wasn't capacity due to the manual test cycle, the multiple sign-offs, and scheduling and running the meeting. More often, and most of someone's time would be spent on organizing and running releases.

It's not just a "releasing was hard work" complaint that I have. It impacted the cycle (The length of the feedback loop for code from Pull Request to production) a lot. This in turn made the delivery batches larger, which unless you push back hard, makes the cycle time larger again. So releases always rolled up about a week's worth of work for a team, which also makes the release harder.

In terms of [the 4 DORA metrics](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance): Lead time for changes could not go down, Deployment frequency could not go up. Change failure rate in terms of changes that caused issues in production was negligible due to the diligent blue-green process, but deployments were occasionally backed out due to issues in one of a batch of changes.

The fear of change given relatively infrequent and expensive releases was noticeable. Quality and incremental change was starved out in favour of hacking in new features that had a JIRA and a manager wanting them, as these were much easier to justify to the test team.

The test team and it's multi-day process going through each release was viewed as vital. And they had no incentive to automate themselves out of their current job.

It was around then that I started to say "These problems cannot be solved by adding process; the problems that can be solved by adding process have been  thus solved; so the remaining problems are those caused by too much process." I don't think that this was ever even taken seriously. The impacts were either not understood or not deemed important.

The thinking was of "why make deployments easier when you could be working on features?" or "Why fight against The Way We Do Things?", "Why release more often than once a week, that would just be much more work doing releases?" and the idea that less human-in-the-loop processes must be "cowboy yeeting code" rather than "repeatable, consistent automation that removes human error and never takes short-cuts". The incremental refactoring necessary for building quality in, never happened.

This is a big reason why I don't work there any more.
