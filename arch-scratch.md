
# Architecture

Look at a poor quality codebase, one that you don't like to work on, and it might have these hallmarks:

It is littered with half-completed migrations, where to different technologies or styles are used side by side.
There are vestigial features that were brought in, and never reached sufficient utility to justify the setup costs. And yet they remain.
There are things that don't serve any purpose any more, and could have been removed a while ago, but have not been.

This is once kind of tech debt. These are failures of planning.

Finishing, and follow through is important. This kind of iterative work that is not captured well by a transactional model where "this PR fulfils this Jira ticket and it's done". Maintaining software just isn't always _like that_.
Taken to extremes, this model gives rise to a "drive-by" coding style, where to fulfil the ticket, you pull up, dump a load of code and exit ASAP, with no concern to maintainability.
Tickets will of course be fulfilled at a slowly decreasing rate, as it becomes harder and harder to work with.

You need to have confidence that there will be follow-through, and it will not be prevented by process. changing focus halfway through a feature will remove this confidence, and result in work done in a less iterative style, as there is less expectation that there will be time for follow-through. Or that the next feature will be completed  before focus changes again.
