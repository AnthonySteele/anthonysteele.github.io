# An On-call War Story

On-call is a valuable practice, a closing of the feedback loop of "you code it, you deploy you operate it, you learn and improve the code".
What happens in production is the truth of your application, and it's a huge benefit to learn from that.

But it's not a combat zone. There are rules to making it work. Here is my "war story" of when it went wrong.

## The Story

I had been at the company for a few months, was getting to know the systems, I was added to the on-call roster.
This was my first experience of on-call, and I wanted that experience in "DevOps" practices. I get a kick out of finding issues in the production logs, reducing noise and increasing reliability, and this seemed like the right way to get there.

it was going OK, I learned how to operate the production systems, e.g. how and when to scale up. Minor issues seen in production got fixed along the way.

But then a few months later, one person moved to a different team and one person resigned. On-call went from once every 5 weeks to once every 3 weeks.
At the same time load on the system  was ramping up, and there were more and more capacity-related issues. Weekends and evenings were busy times for this company. At the same time there was a big push for more features.

Call-outs became very common over the weekend. It became very unpleasant, and bad for my general well-being. In retrospect, the stress that was feeling was progressing to burnout.

Rota flexibility disappeared. I could not swap any on-call days any more.
Not only were there simply fewer people to ask, they were limited to the one who was on last week and the one who is on next week; and neither was happy about taking on more on-call.

Management were not allowing sufficient time for alerts to be squashed, focusing on all-in for features. More than anything else it stressed me out to come in on Monday and try to raise the issues in the face of the features.

I raised to my line manager that this was too hard. I got the pep talk in response: It was fine and normal, things were functioning as intended, and that the stress was sustainable. But it was not.

If I had refused to be on that rota, it would likely have collapsed as the other two people were under the same pressure and would not have coped with it.
A big reason why I didn't quit on-call was to not leave them in the soup. In retrospect, I think that this gave me more leverage than I realized at the time: If I had quit it, that rota could not have continued to function.

Eventually I contacted the manager's manager, who got three more people onto that rota. This eased the pressure.

Shortly after that, due to the ramp-up of issues and outages, upper management called a complete release freeze. This was a complete reversal of development direction, and IMHO a huge over-reaction.
For one thing, it severely bottlenecked the ability to deploy more stable and operable code.
But that's a different war story. The on-call stress largely went away, but a different stress replaced it.

## The Learnings

Three people on a rota that experiences significant numbers of call-outs is not enough. I feel that I know more about what is acceptable for on-call now, and what is a danger sign, and would flag up that situation earlier.
It was untenable both in respect of not enough number of people on the rota, and what is a too-large volume of call-outs. Burnout was a risk.

I don't hate on-call on the whole, and would not hesitate to recommend the practice others.
But I look back on my experience of on-call during that period of time with loathing. On call done well gives a lot of benefit, but done badly can be very bad, so setting it up well is important.

On-call is a necessary part of closing the feedback loop, but it's just a part.  If the other parts aren't there, it won't work and you will feel the pain.

I'm  in my case particularly to listening to the on-call issues, allocating more time to fixing them - knowing when to focus more on reliability, operability, scalability, rather than features, and a sustainable mix of both with minor course corrections, rather than sudden changes of direction. See for example having an [error budget](https://danlebrero.com/2017/07/16/error-budget-google-solution-for-innovating-at-a-sustainable-pace/).

## Other things to call out

* [On call is the ambulance crew doing first aid, not the doctors curing the disease](https://www.youtube.com/watch?v=7tTsxfsxw3Y&feature=youtu.be&t=733). The comprehensive fix starts with the rest of the team, the next day.
* [Edge-cases will happen: e.g. If you have a million transactions a day, then one-in-a-million things will happen daily](https://www.youtube.com/watch?v=7tTsxfsxw3Y&feature=youtu.be&t=1997). And you can't anticipate everything, production will surpise you from time to time.

## Links

* [Tips on running an on-call rota](https://blog.hinterlands.org/2010/07/running-an-oncall-rota).
* [You build it, you run it, Chris O’Dell - Agile on the Beach 2018. Slides](https://speakerdeck.com/chrisann/you-build-it-you-run-it-1).
* [You build it, you run it, Chris O’Dell - Agile on the Beach 2018. Video](https://www.youtube.com/watch?v=7tTsxfsxw3Y).
