## Against Pull Requests


The git [Pull Request](https://help.github.com/articles/using-pull-requests/) is a powerful tool. But just because you have it, [does not mean that you have to use it all the time](https://en.wikipedia.org/wiki/Law_of_the_instrument). 

Pull requests have a powerful legitimate use. Consider a team that owns on a codebase and works on it regularly. 
Pull requests give outsiders a powerful thing that they did not have before: controlled access to apply changes that they need, with review by the owners.
 Someone can come along and say "I'm not a member of your team, but I need as an outsider to get some code into your project, so have a look at my offering."

However pull-requests is not a good process to apply across the board to those same owners as a matter of course. 
Pull requests give everyone controlled access: they allow anyone to knock on the door of the room and say "Can I come in?" But if you work in the room, why would you do that?

I also think of it as like a gate in an ancient walled city: A stranger can come in, if they pass inspection at the gate.  Without the gate, they would be reduced to standing outside and shouting over the wall "We need this feature!" and hope that it happens soon. 
Possibly an ambassador will come out to negotiate terms and times. 
So the gate instead allows them to come in and get business done, while respecting the local customs. 
But would you put that checkpoint inside the city and apply it to every citizen?

Pull request give review. There are other ways to do review. Human review is inconsistent, time-consuming, not scalable or repeatable and invariably has a political aspect. 
Review helps as a bug-finding process, but it is not a good substitute for e.g. simple test coverage. Review helps as a style and architecture process, but is no substitute for pairing.

Pull requests do not scale - A tailback of unmerged PRs is always a bad sign, and not just because of having to rebase or fix merge conflicts.
An unmerged and undeployed PR is [work in progress](http://kanbantool.com/kanban-wip-limits). Your cycle time from writing code to deploying code when joining the back of that queue is much greater, and the risks are greater too. In extreme cases there can be forgotten PRs that are weeks or months old, no-one remembers exactly what their intent was, and are neither easy to safe to merge any more. This is just wasted work.

> Continuous integration (CI) is the practice, in software engineering, of merging all developer working copies to a shared mainline several times a day. https://en.wikipedia.org/wiki/Continuous_integration 

Pull requests encourage you to forget that the "Integration" in "[Continuous Integration](https://en.wikipedia.org/wiki/Continuous_integration)" literally means "Merge to master". A branch or PR is by definition not integrated.  
You can test a PR, but [you can't continuously integrate without closing it](https://www.infoq.com/news/2015/10/branching-continuous-integration).

If I was starting from scratch with setting up a process, I would work hard to avoid mandatory Pull Requests and encourage [Trunk-based development](https://dzone.com/articles/organisation-pattern-trunk-based-development). 
It might be that the lack of PRs would force you to do other things: good, those other things are things worth having. 

This all supposes that code has an owner. If you don't have such ownership, i.e. nothing but feature teams, you are going to have a bad time in the long run because of this lack of ownership.


## Practices for using Pull Requests

If you are going to use PRs, consider these guidelines to make the experience less painful:

* Extra process that you use but do not need is counter-productive: it causes extra cognitive load, extra work, slower delivery and can even cause errors. Try to discard it.

* The PR workflow is most appropriate with public open source where anyone can view and submit a PR, but only a few can write to the master branch. It is fairly appropriate for "internal open source" or "shared code" that lives within your organisation but is worked on and used by a large number of teams, it is possibly appropriate when you have a remote workers who are also working on a codebase, and it is least appropriate for a co-located team who own a codebase and are the main committers.

* Even for a co-located team who own a private codebase and commit freely to it, allow pull requests from elsewhere in the organisation.

* Make PRs are short-lived as possible. Try not to have lots of outstanding PRs. A long queue of old PRs is a warning sign. 

* As a general rule, the longer the PR author can be involved, the better. Hopefully they can be involved until thier change is deployed to production and verified to work as expected. They should not consider it "done" and have no further involvement once the PR is submitted.

* It is up to the PR author to keep the PR up to date with the `master` branch  by merging or rebasing and fixing merge conflicts.

* It is up to the PR author to follow  up with the owners if their PR has not been reviewed or merged, to make changes as requested, and to explain why it is needed or to close it if it is no longer needed.

* It is up to the repo owners to review PRs and merge the good ones, and ask for clarification and offer suggestions; or close the ones that are not good. Do not let an unmanageable PR backlog build up.

* Deal with the oldest PRs first, as a general rule. This has several benefits: PRs are merged in a predictable order, not as favours. And it reduces merge conflicts or at least makes them the responsibility of the person who opens a new PR when they can see that it will clash with a existing PR.

* If a PR is obsoleted by other commits that make some of the same changes or otherwise fix the same problem, then anyone should close it. The author can assess against the current master if there are any remaining parts that are worth extracting as a new PR.

* Consider your release process. Does merging four PRs mean four separate deployments to production? How long does each take? Can and should you group PRs in one deployment, and does this depend on if they are major or minor changes? Where is the bottleneck in the system? 

* Consider your testing process. Running unit tests against PRs as well as on master can be very useful, but is not in itself Continuous Integration. How long does this take, and is it a bottleneck?



