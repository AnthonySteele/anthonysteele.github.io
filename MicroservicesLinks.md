Links

* [The twelve-Factor app](https://12factor.net/)
* [You need to be this tall to use micro services](https://news.ycombinator.com/item?id=12508655)
* [Microservice Prerequisites](http://martinfowler.com/bliki/MicroservicePrerequisites.html)
* [10 Modern Software Over-Engineering Mistakes](https://medium.com/@rdsubhas/10-modern-software-engineering-mistakes-bc67fbef4fc8)

Microservices have a dual nature: On the one hand, they give independence of code, testing, deploy times, versions, tools, techniques, etc. The ability for a team to work on and deploy an isolated unit of functionality when ready is great. It really is an archetectural style for the agile and Continous Deployment era.

On the other hand, Microservices imposes a strong contract with how a service behaves. That service will have to be a "good citizen" of the ecosystem, even while it has freedom of how it implements that. For instance, it will need to participate in centralised logging and monitoring systems. It will need health checks for load balancers. Those load balancers can create and terminate instances at will, so work-sharing is a concern. Coordinating all this across many teams' code is different to doing it to one monlithic app.

Because of the overhead, without scale you don't get much benefit from Microservices. 


Links

* [Beware the share](http://programmer.97things.oreilly.com/wiki/index.php/Beware_the_Share)
* [The Seven deadly sins of Microservices](https://opencredo.com/7-deadly-sins-of-microservices/)
*  Ben Christensen: [The Distributed Monolith](https://www.infoq.com/news/2016/02/services-distributed-monolith)
* [Don't share code](https://www.infoq.com/news/2015/01/microservices-sharing-code)

There's a point repeated here, that **Dont Repeat Yourself** is an excellent rule, but it is not the only rule and it is contextual. Beware of taking DRY too far; especially at larger scales where it breaks down, e.g. across the boundaries of one executable. You remove duplication but you create coupling and the costs can outweigh the benefits.  **[Duplication is better than the wrong abstraction](http://www.sandimetz.com/blog/2016/1/20/the-wrong-abstraction)**.

Microservices, of course, creates lots of these boundaries. Be wary of code that is required to make these services interoperate and is not common enough to form part of the language framework or an open-source public package. Microservices should be mostly independent.
