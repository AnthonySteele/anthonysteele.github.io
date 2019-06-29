# The bug that took a year to fix

Here is another war story.

One of the issues that was raised to the team that I was on, was an order that could not be processed. Just one order at a time. Once in a while, for no clear reason. Everything looked good, and yet it failed.

The cause was eventually tracked down, and the fix applied, but that took the best part of a year of intermittent effort.

## The Bug

The team that I was on, if I remember it right, had a service called `OrderPartnerWorker`, that received messages over SQS on AWS. The affected message was called `OrderAccepted`, and it contained an order id for the new order. We processed it, eventually forwarding it to the correct third party.

Sometimes, around once per week, this failed. The team sending the message insisted that all was well at their end.  We didn't have much to go on, so simply had to improve logging around the `OrderPartnerWorker` code and wait for it to recur. Logging was improving at this time, from an initial low standard.

We eliminated many things: it wasn't some resource exhaustion or concurrency thing, correlated with high load - it was only around twice as likely to occur when there were twice as many orders. SQS retries were working normally.

After a while the logging was good enough to see what was failing, and the investigation turned to timing issues. This was finally the right track.

## The Friction

 Losing orders is about the most serious failure that could happen in this business: the company doesn't get to make money on the transaction, and the customer is annoyed. An outage when orders aren't being processed at all was taken very seriously, but this wasn't an outage, just a recurring blip.

There were two teams involved, so there is a political, human dimension. The message exchanged literally formed the point of contact between their systems. JIRA tickets formed the point of contact between the people.

I'm sorry to say that in this point the teams, who were seldom in the same room, did not see eye-to-eye. The downstream team knew that there was a real issue. The upstream team claimed that "if there is an issue, it's not ours" and had more urgent things to do than investigate someone else's "one order a week" issues. The downstream team wanted correctness, not ongoing late-night call-outs; and we were worried that it happened twice this week and once the week before; there was nothing to stop it happening hundreds of times next week, in which case the CTO would be breathing down our neck.

Loud conversations were had in meetings, embarrassing bystanders. Fingers were pointed. People were rudely cut off mid-sentence. There are still memories.

## The Reasons

We eventually found and fixed the issue. And it was more a design issue than anything else. Both the upstream and downstream systems were actually working as designed all the time, the fault lay in the pipes between them, and in a third system.

SNS and SQS are extremely reliable and scalable systems, I would happily use them regardless of if I am handling 1 message per week, or if I am handling millions of messages per day. The consumers would look different: a lambda in the first case vs. multiple instances of a full app in the second, but that's a another story.

_Almost all_ messages are delivered over SNS once, very quickly; but there are edge cases, outliers and failures, and at scale they happen. Given e.g. a million messages per day, a "one in a million" event will happen daily. SNS and SQS tend to fail in the direction of delivering a message late, or delivering a message multiple times instead of dropping the message. It is "at least once" delivery.

What was happening in this case is that the `OrderAccepted` message was received, the `OrderPartnerWorker` immediately went to the order API and tried to get details of the accepted order over HTTP. The order API replied "What accepted order? It doesn't exist!".  But the order API was not the `OrderAccepted` message sender. The order API also listened to other (further upstream) messages about orders and put data in in its store, to respond to later GET requests. Once in a while the message to it arrived tens of seconds late there, and it literally didn't know about it yet when `OrderPartnerWorker` tried to process it.

## The Learnings

It was an "eventual consistency" problem, a race condition. In a distributed system if your service is told that "order 12345 has just been accepted", you can't always assume that other services are in that same state right then. They might not have got there yet, or they might have moved on to another state. And the order in which it's designed to happen is not always the order in which it does happen.

The fix was a design change. The `OrderAccepted` message was revised to contain not just an order id, but was a "fat message" containing a JSON representation of the order. The call to the order API was eliminated, and the problem went away.

The consequences were that, in general, calling back for details about the message just received started to be rightly regarded as bad idea. Aside, from eventual consistency concerns, even when things go well it's not the best design: As part of the push for reliability, some systems were designated "mission critical", i.e. we wanted them to be running always, more so than other systems. In the original design, if `OrderPartnerWorker` was mission critical, then _so was the order API_ because `OrderPartnerWorker` could not function without it. `OrderPartnerWorker`'s uptime is constrained by the order API's uptime.

The "fat message" decoupled them; it eliminated that dependency and improved reliability. A while later we learned that the common technical term for this pattern is (unsurprisingly) not "fat message", It is [Event-Carried State Transfer](https://martinfowler.com/articles/201701-event-driven.html). This talk explains more: [Event Driven Collaboration, by Ian Cooper at NDC](https://www.youtube.com/watch?v=PreAnSofAsA&feature=youtu.be&t=1819)

The other thing is that distributed systems are hard, the edge cases will surprise you. We knew some things about SQS's edge cases, and so we checked that our message handlers were idempotent (they were), But still the issue was not anticipated. You can't plan for everything that happens at scale so good logging and monitoring are important. As is learning from others best practices.

Another thing is the hard parts can be at the seams, which can be [both technical and political seams](https://en.wikipedia.org/wiki/Conway%27s_law). When teams disagree on how severe the issue is; when one team feels the consequences (in the form of on-call issues), but another team owns the fix, this is a recipe for friction.