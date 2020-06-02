# DevOps for Developers

About pipelines and feedback loops.

## About me

 I am a fan of [DevOps](https://en.wikipedia.org/wiki/DevOps), but I don't describe myself as a "DevOps Engineer" as that gets misinterpreted -  I'm primarily a Developer not an System Administrator and I still don't understand how to configure DNS. But I am a developer who wants their software to be run (otherwise what's the point of it?), and even better if it runs at scale and under load. I am cynical enough to know that there's a good chance that it will run badly the first time, but optimistic enough to think that I can get it to work well eventually.

 I want to know the truth of how it ran, for better or worse. Otherwise, how can I improve it? In order to run software well in production, we need be prepared to invest in build, test and deploy automation, and live monitoring and logging for after that.

 I'm also a fan of "Agile" and "Continuos Delivery" methods that I encountered before DevOps". And I think that iteration is key to them. The values of the [Agile Manifesto](https://agilemanifesto.org/) do not specifically say "feed back and iterate" but I think it's present there - especially in "Responding to change". The idea likely dates back to  [The OODA Loop](https://en.wikipedia.org/wiki/OODA_loop).

## About DevOps

What is "DevOps"? It's not a product, it's not a team. It's not rebranded System Administration. It's not a particular deploy pipeline. It's not only about deploy pipelines in general, although you will need them. It's a way of thinking.

The clue to "DevOps" is right there in the word itself, which joins "Development" and "Operations" into one thing. Maybe "DevelOperations" was considered and rejected as too hard to say. Before I'd heard of "DevOps" I had heard of "**[You build it, you run it](https://www.youtube.com/watch?v=UNxhm89DwlY)**", which I think is as good a summary as you'll get in six words. Consider that you can also phrase that as "You _Develop_ it, you _Operate_ it".

Running the software and being on call for it taught me that what happens in production is truth, and you're better off knowing the truth sooner instead of later. It taught me that Ops is hard when things go wrong, and that you need lots of good data coming out of the running application, in order to understand what's actually going on.

## Deployment

> DevOps focuses heavily on establishing a collaborative culture and improving efficiency through automation - Collab.Net

DevOps is now often taken to mean automated deployment. But all successful software methods ever, had deployment.  It's not optional for success: without deployment, there is no running software. However, maybe in the old times we did deployment infrequently, manually, slowly, ad-hoc, with downtime, or poorly in other ways.

Emphasis on painless deployment is not that new either: [Continuous Delivery](https://en.wikipedia.org/wiki/Continuous_delivery) (and Continuous Integration and Continuous Deployment) was partly about making the deploy process repeatable, fast, verified and safe, by automation. It pre-dates "DevOps". So DevOps is not merely "having a build pipeline", and having a build pipeline may be necessary but is not sufficient to have some "DevOps".

## What's new in DevOps

> DevOps is just adding the operations' mindset and maybe a team member with some of those responsibilities into the agile team. ... To achieve this, Dev and Ops must break down the silos and collaborate with one another, share responsibility for maintaining the system that runs the software, and prepare the software to run on the system with increased quality feedback and delivery automation - Collab.Net

There is a theme of "breaking down divisions between different capabilities that might have different schedules and different priorities". With Continuous Delivery methods, development and testing and deploying are merged into working together.

With DevOps it's (you guessed it) Development and Operations that are working together.

### The circular pipeline

Let's leave aside the "infrastructure as code" aspects, and ask what "DevOps" means for a developer doing an everyday change to existing code.

If we already highlighted the "Dev ⇒ Ops" pipeline that metaphorically moves software from you to production. But this is linear. What about "Ops ⇒ Dev"? It is only a feedback loop if both sides are present. "Dev ⇒ Ops" requires the deploy pipeline, "Ops ⇒ Dev" requires that you know what's happening in production, therefore it requires observability, logs, metrics, monitoring, alerting etc.

The feedback from production always did happen in some form, even if badly: even if you got told that your site was down by a customer, even if you had to log into a server to view a log file, even if it was hard to find out how much CPU or disk space your app was using until it ran out. But now we are interested making this feedback much better than that.

Ops and "you run it" are things that happens after deploy succeeds. So as well as build pipelines, [the set of tools and techniques that work production](https://medium.com/@copyconstruct/testing-in-production-the-safe-way-18ca102d0ef1), and covers observability, logs, metrics, monitoring, alerting, is also key to success at DevOps.

So I believe that what is new in emphasis in developing with DevOps, is the *Ops* feeding into the *Dev*, the feedback from production, the closing the loop, the fast feedback all the way through, the info from running it that informs what you build next. That is what's new in DevOps.

And that we can and should pay even more attention to this, and in making that process repeatable, fast, verified and safe, by automation.

The fact that this is a feedback loop, which we are seeking to make faster means that this is an agile process. "Faster" meaning we can go around the loop in less time. Which also means that we can afford to instead take smaller increments of change around the loop more often, and observe what effect they have in isolation; or verify that a small refactoring code change or library update has no observable effect and is thus safe. This means more confident refactoring, more incremental change and therefore more safety and learning.

## You build it

"You build it, you run it" also means "You build it, you test and deploy it, you run it, you monitor it. You feed the insights gained from that back into building the next cycle around this feedback loop. You do this autonomously and iteratively. You work in small increments that go all the way through from idea to live monitoring. You do each increment quickly, confidently and safely, assisted by automation in all steps."

## Links

* [What is DevOps - Collab.Net](https://resources.collab.net/devops-101/what-is-devops)
* [DevOps Culture - Martin Fowler](https://martinfowler.com/bliki/DevOpsCulture.html)
