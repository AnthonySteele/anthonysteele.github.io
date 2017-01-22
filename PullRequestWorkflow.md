The git [Pull Request](https://help.github.com/articles/using-pull-requests/) is a powerful tool. But just because you have it, [does not mean that you have to use it all the time](https://en.wikipedia.org/wiki/Law_of_the_instrument). 

I am thinking of a team that owns on a codebase and works on it regularly. Pull requests give outsiders a powerful thing that they did not have before: controlled access to apply changes that they need, with review by the owners. If you don't have such ownership, i.e. nothing but feature teams, you are going to have a bad time in the long run because of this lack of ownership.

However pull-requests is not a good process to apply across the board to those same owners as a matter of course.  Put it this way: Pull requests give everyone controlled access: they allow anyone to knock on the door of the room and say "Hey, I have some code for you, please have a look?" But if you work in the room, why would you do that?

I also think of it as like a gate in an ancient walled city: A stranger can come in, if they pass inspection at the gate.  Without the gate, they would be reduced to standing outside and shouting over the wall "We need this feature! And you need to update some urls!" and hope that it happens soon. So the gate allows them to come in and get business done, while respecting the local customs. But would you put that checkpoint inside the city and apply it to every citizen?

Pull request give review. There are other ways to do review. Human review is inconsistent, time-consuming, not scalable or repeatable and invariably has a political aspect. Review helps as a bug-finding process, but it is not a good substitute for e.g. simple test coverage. Review helps as a style and architecture process, but is no substitute for pairing.

Pull requests do not scale - you can wait for one PR to be merged before the next one, or you can anticipate trouble. They increase cycle time.

> Continuous integration (CI) is the practice, in software engineering, of merging all developer working copies to a shared mainline several times a day. https://en.wikipedia.org/wiki/Continuous_integration 

Pull requests encourage you to forget that the "Integration" in "[Continuous Integration](https://en.wikipedia.org/wiki/Continuous_integration)" literally means "Merge to master". A branch or PR is by definition not integrated.  You can test a PR, but [you can't continuously integrate without closing it](https://www.infoq.com/news/2015/10/branching-continuous-integration).

If I was starting from scratch with setting up a process, I would work hard to avoid mandatory Pull Requests and encourage [Trunk-based development](https://dzone.com/articles/organisation-pattern-trunk-based-development). It might be that the lack of PRs would force you to do other things: good, those other things are things worth having. 
