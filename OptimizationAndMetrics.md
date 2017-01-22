About optimisation and benchmarks; bottlenecks and what difference optimisation will make. 

## ASP Kestrel Benchmarks

I heard that the ASP.NET core beta running on [Kestrel Web server](https://github.com/aspnet/KestrelHttpServer) was capable of processing "[over one million requests per second](https://twitter.com/DamianEdwards/status/679743708758061056)" though [a simple benchmark](https://github.com/aspnet/benchmarks/blob/master/src/Benchmarks/PlaintextMiddleware.cs) Update: it's [nearly 1.2 million now](https://twitter.com/DamianEdwards/status/692171817327403008). [The full benchmarks are at the bottom of this page](https://github.com/aspnet/benchmarks/blob/dev/README.md)

My thoughts were:

1) Wow! That's incredible. What an achievement, that's going to get people's attention! How is that done?   
2) Wait .. that is *literally incredible*. In scenarios that I have seen, you don't get 1% of that throughput, and I [don't believe](http://www.merriam-webster.com/dictionary/incredible) that the code could do anything useful in the time slice that it implies: The code that I have already inside controller methods always takes a lot longer than that to execute. What does this benchmark mean?  

### Throughput and latency, asynchrony and bottlenecks

What does this benchmark actually imply? 
When one million requests are processed in a second what actually happens is somewhere between these two unrealistic extremes:

* Requests are processed one at a time, each taking around a millionth of a second (AKA [1 microsecond](https://en.wikipedia.org/wiki/Microsecond), 1000 nanoseconds, or a 1000th of a millisecond). This is unrealistic as the machine would have unused capacity while processing just 1 request.
* One million threads are spawned, and they all complete around a second later. And spend most of the time inbetween not doing anything. This is unrealistic as it's not useful or even feasible to run that many concurrent threads. 
 
No, the truth is somewhere in between these extremes. This benchmark was apparently done on a  six-core machine, so each request has 6 microseconds of CPU time to use, but probably not all in one uninterrupted chunk. There will be more than 6 requests being processed at any one time: Kestrel uses [libuv](https://en.wikipedia.org/wiki/Libuv) at heart, a library all about "Asynchronous" I/O. So when a request cannot be read immediately or a response written immediately the thread does not block, it yields and can be used for something else. Async code can increase *throughput* by allowing threads to be re-used instead of being blocked busy waiting for responses, but does not decrease *latency*, the time taken for an individual request. This is sometimes phrased as "you can't wait faster, but you can wait better".

There is always a bottleneck, a limiting factor that determines the capacity of software. 
A task is said to be [CPU-Bound](https://en.wikipedia.org/wiki/CPU-bound) if it's run time is determined by processor capacity, for example  calculating a cryptographic hash. A task is said to be [I/O bound](https://en.wikipedia.org/wiki/I/O_bound) if it's  run time is determined by capacity to read and write data. Web servers are usually I/O bound.

Yet we will also have to make a distinction between a web server being **I/O bound at the front end**, on reading Http requests and writing the response out again; and **I/O bound at the back end**, when the server is itself retrieving data; i.e. initiating a request for data over http or from a sql database. This benchmark is I/O bound at the front end since all it does is front-end I/O.

But code that is I/O bound at the back end is very common in the wild. Almost all of the code that I see on a regular basis in ASP.Net Web apps gets data from some back-end data store. The typical flow is that a HTTP request comes in, it is routed to the right controller or handler, it is validated and/or massaged a bit. Then data is retrieved using keys from the request, and this data is translated into a DTO for output as json or xml; or to a viewmodel for Html generation. 

e.g. in a very simple case, the url is `GET http://api.foo.com/widgets/123`, the route is `[Route("widgets/{widgetId}")]`, the action method is `public Widget Get(int widgetId)`, and in it a query is issued with `SELECT * FROM widgets WHERE id = 123`  and a response is returned that is serialised to  `{ id = 123, name = "Sprocket widget", cost = 12.34 }`.

The moving parts are the same regardless if data is retrieved from e.g. a SQL Server database, a NoSql store, or a microservice over HTTP which might or might not have a cache in front of it. In all cases, there is probably a network round trip, and the web app simply has to wait for a response -  optimisation of the code that initiates the I/O and waits for a response won't speed it up. An asynchronous HTTP GET takes just as long as a synchronous one: however long the server takes, plus a little more for the network. But it is more efficient to be async. Again, Async code can increase *throughput* but does not decrease *latency*: once the web app has sent a request, the time taken to respond is out of its hands.

Making things worse for measurement, there is often considerable *variation* in time to respond, and this is also not under the client's control or easy to predict. Some of it is due to what else is happening on that server and network, some is effectively random noise, some of it is [non-linear response to load](https://vimeo.com/145152727#t=600). Sometimes, caches added to improve the average response time do nothing for the worst case of a cache miss, thus increasing variance.

Getting data from a data store and into a http response can take as little as 10ms or as much as several seconds; but typical and acceptable times are in the range 50-750ms. The time to retrieve the data from the store is usually the main factor determining the time to respond to the http request. 

## Latency is not throughput, but they are related

Latency is not throughput, but they are related. They come about in different ways: Throughput is constrained by bottlenecks in the pipeline. Latency is the sum of all delays in that pipeline. Machine capacity is a hard cap on the maximum of (latency * throughput).

Can you do 1 million requests per second when each one takes 10ms? i.e. is 1 million requests per second thoughput and 10ms or more latency realistic on a single machine? I doubt it: That would mean there will be around 10 000 requests in flight (either actively being processed or suspended on I/O) at any point in time. This doesn't sound likely. You can do the numbers yourself for longer (i.e. more usual) request latencies. This is why a database slowdown that causes an increase in latency can have non-linear knock-on efects: it increases the number of request in flight at any one point in time, so is roughly equivalent to an increase in load. 

## Scenario planning

The "1 million requests per second" scenario is [a benchmark of simplest code](https://github.com/aspnet/benchmarks/blob/dev/src/Benchmarks/Middleware/PlaintextMiddleware.cs), so this will be the time to do the *simplest possible thing* with additional tweaking:  the minimum framework overhead to respond at all. 

This high throughput will allow the ASP framework to be used in scenarios where previously it could not be, e.g. [pushing game frames over a socket](https://twitter.com/Illyriad). I don't know much about these scenarios, so I won't discuss them in detail. But I am sure that for all the attention-getting thrill of these cutting-edge low-latency cases, they are an unusual niche and it would be a mistake to tune the general purpose ASP framework to them at the expense of the normal case. But hopefully there's a synergy instead: fixing things that break at the limit might improve the normal state. 

But when porting existing code to the latest ASP framework, your code in the controller won't run faster. The "1 million" does not apply.

That benchmark number drops to 40 000 requests per second when [benchmarking retrieving a record from a database](https://github.com/aspnet/benchmarks/blob/master/src/Benchmarks/SingleQueryRawMiddleware.cs). This is still very good, but is around 4% of the previous benchmark and you look elsewhere to improve it. The stark difference between the two is instructive: if you want to improve this second number, then the difference isn't in the ASP framework, but in the database round trip and associated code.


## The root of all evil

Knuth famously said

> "premature optimization is the root of all evil" - Donald Knuth

But [actually, there's a little more to it](http://www.codemesh.io/codemesh2015/alvaro-videla).

> There is no doubt that the grail of efficiency leads to abuse. Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We *should* forget about small efficiencies, say about 97% of the time: premature optimization is the root of all evil.
>
> Yet we should not pass up our opportunities in that critical 3%. A good programmer will not be lulled into complacency by such reasoning, he will be wise to look carefully at the critical code; but only *after* that code has been identified. It is often a mistake to make a priori judgements about what parts of a program are really critical, since the universal experience of programmers who have been using measurement tools has been that their intuitive guesses fail.   
> Structured Programming with `go to` Statements,  [Donald Knuth, ACM Computing Surveys, Dec. 1974](http://dl.acm.org/citation.cfm?id=356640) 

But this 97% rule is likely to be out probably doesn't apply verbatim to framework code. Yes, only a small fraction of the code is critical. But which code is critical will vary: different consumers with different workloads will exercise different parts of the framework in different ways. So optimisation is more important. But as Knuth says, you need to measure it first!

## Theory of constraints

[The theory of constraints](http://www.leanproduction.com/theory-of-constraints.html) is important to know as well. A web request is not exactly like [a linear production line](https://en.wikipedia.org/wiki/Theory_of_constraints#Plant_types), but there are major similarities. In a production line composed of several steps, the step with the lowest throughput is the bottleneck; and increasing capacity on other steps will have no effect on the throughput of the whole line.  In a web request, delay is cumulative, but still the process of identifying and removing bottlenecks is key, as they dominate response time as Knuth observed. 

## The effects of optimisation

So how would existing code be affected by this performance benchmark?

Lets give some numbers: suppose the request is processed in an average of 10 milliseconds. This is optimistic. Suppose that the framework takes 50 microseconds of that. How much benefit is to be had from optimising the framework code? If framework processing now takes 25 microseconds, then the time is halved! Throughput is doubled! An amazing 100% performance improvement!

Well, no, only part of it was halved. A small part. And the whole request that was was processed in 10ms, i.e. in 10 000 microseconds, is now processed in an average of 9 975 microseconds, i.e. an overall 0.25% performance improvement. This is such a small change that it would be lost in the noise. 

The part that has been optimised does not matter since it is not the bottleneck.  Ironically, the ASP.NET team could super-optimise everything that's in their purview to optimise, and it wouldn't make a difference. 

Or take another scenario, a "middlewight" MVC app that takes 200ms to respond. Because it uses more heavyweight framework parts such as routing and controllers, the framework takes 200 microseconds of that response time. This is optimised to 100 microseconds by a framework performance doubling. But this only results in an overall 0.05% performance improvement. If you really needed this response to be faster you would do much better to take Knuth's advice: do some measurement, which will most likely lead you to look at speeding up the database, or caching data to avoid it altogether for many requests. And "forget about small efficiencies" in the MVC framework. 

What does the headline benchmark of "1 million requests per second" mean? It's a slogan to get attention. It's bragging rights against other frameworks. But for ordinary production code, it means less than it seems at first. In fact not very much at all.

## Thanks to:

Thanks to the [ASP.Net](http://www.asp.net/) team for making great cross-platform software.    
[Damian Edwards](https://twitter.com/DamianEdwards) and [David Fowler](https://twitter.com/davidfowl) for taking the time to present it at [NDC London](http://ndc-london.com/).   
[MattG](https://twitter.com/portedegrange) for reading a draft of this article and constructively feeding back that I didn't really know the definition of "I/O bound".