##The Single Return Law.

I wrote this a few years ago (before 2012), and it is now hosted here for reference. This is still my opinion. To sum up:

* There is a high bar to clear to call something a "law" and the idea that "a method should have at most one return statement" does not meet it. It is just a style, and one that works well in some cases and less well in others. If you learn when to judiciously use multiple returns, you can write more expressive code. 

* The arguments in favour of the single-return style originated in the C programming language for resource management reasons. They are irrelevant to Java, C#, Javascript, ruby, python etc. Do not blindly follow cargo-cult rules.

* Most of the outraged, dogmatic replies that I have received are along the lines of "but in _this_ case a single return is better." No doubt, but that is no contradiction to what I am saying: Both styles have uses, so learn to apply both, and choose which best fits your case. 

* If you somehow still think that the single-return rule is applicable across all languages, please read about match expressions in F#, Erlang's case expressions or Haskell's pattern matching and get back to me. In those constructs, you cannot avoid using multiple return values.

## Original text


It is sometimes said that a method should have only one return statement (i.e. one exit point) and that to code using more than one return per method is bad practice. It is claimed to be a risk to readability or a source of error. Sometimes this is even given the title of the "single exit point law".  Actually, if you want to call something a law, I'd expect some evidence for it. I do not know of any formal study that back this "law" up, or of any study of the multiple-return pattern at all. This makes it a "preference" not a "law". And it's a preference that I do not hold for c# code. Or java, ruby or python code either.


This coding rule dates back to Dijkstra's <a href="http://en.wikipedia.org/wiki/Structured_programming">structured programming</a>. However it is outdated, if it ever ever was a "law" or "principle". The single return rule originates in the era of C, FORTRAN or assembler, where it was common to allocate resources (most frequently memory, file handles or locks) at the start of the procedure, and to de-allocate them at the end of it. An early return can lead the programmer  to either forget to do the cleanup code and cause a memory leak or locked file, or to maintain cleanup code in two places. So it makes some sense to stick to just one return right at the end of the method.


But in the modern world, this is no longer so. Firstly, garbage collected languages make explicit deallocation unnecessary in most cases - memory is reclaimed automatically. Secondly, try...finally blocks and using statements allow release of other resources to happen with greater certainly at the end of any block of code when it is needed.


In any event, a function with multiple exit points is a far lesser issue than <a href="http://www.u.arizona.edu/~rubinson/copyright_violations/Go_To_Considered_Harmful.html">a goto</a>.  In some cases it is the simplest way to code now that we have control structures to deal with it.  If your method is long, complex and has multiple returns, consider splitting it up into smaller, well-named methods with a single-responsibly each. Also consider doing that if it's long, complex and has only one return.


My summary of the matter is:


&amp;bull;   There are cases where a single return is more readable, and cases where it isn't. See if it reduces the number of lines of code, makes the logic clearer or reduces the number of braces and indents or temporary variables.
 &amp;bull;   More than one return style may be a bad habit in C code, where resources have to be explicitly de-allocated, but in languages that have automatic garbage collection and <strong>try..finally</strong> blocks and <strong>using </strong>blocks, this argument does not apply - in these languages, it is very uncommon to even need centralised manual resource deallocation.
 &amp;bull;   There is no law requiring only one exit point for a method, and no evidence for it as a rule of thumb. Some people have an unsubstantiated belief in the superiority of this style.


Therefore, use as many returns as suits your artistic sensibilities, because it is a layout and readability issue, not a technical one.


There's another issue at play here, about rules being adhered to without people grasping the reason why they are adhering to them, and thus keeping the rule in force after the need for it has evaporated. When you give a rule, also give the reasons for it, so that people can apply their own judgement as to if it is still applicable.

<h2>Common patterns of early return</h2>
<h3>The guard clause</h3>
<pre class="brush: csharp">public Foo merge(Foo a, Foo b)
{
    if (a == null)
    {
        return b;
    }

    if (b == null)
    {
        return a;
    }

    // complicated merge code goes here.
}
</pre>

It is fine, even in C code to use early return as a first check on the parameters after entering a function, before allocating resources and doing the main work of the routine. It's more readable than embedding the rest of the function in an if-block. Bear in mind that if the parameters are actually in error, throwing an exception is probably better than returning a default value.


The pattern is <a href="http://www.c2.com/cgi/wiki?GuardClause">well explained here on Ward Cunningham's wiki</a>. The discussion there diverges into a more general on on "early return vs single return", with the interesting point that single return requires mutable state in the form of a temporary "result" variable, thus early return, not requiring this piece of mutable state, seems "cleaner" to a functional programming perspective.

<h3>Found item in loop</h3>

Finding an item in a loop using returns was common in C#, but is often now giving way to the more concise LINQ version.  However, beware of returns in the middle of highly nested code or in methods that are more than just a loop (if posible, factor out the loop into a seperate method).  All other things being equal, avoid them, since they are less readable and predictable. However, the following is a valid use of multiple returns:

<pre class="brush: csharp">public Customer FindLuckyCustomer(IEnumerable&amp;lt;Customer&amp;gt; customers)
{
    foreach (Customer customer in customers)
    {
        if (IsLucky(customer))
        {
            return customer;
        }
    }

    return null;
}
</pre>

Note how FindLuckyCustomer does not requre a result local variable, just the customerItem for the loop. The local variable would be "mutable state" for the method. If you have looked at all at functional programming, you can appreciate how eliminating unnecessary mutable state can be a good thing. Here's the code that uses the same loop but avoid multiple returns:

<pre class="brush: csharp">public Customer FindLuckyCustomer(IEnumerable&amp;lt;Customer&amp;gt; customers)
{
    Customer result = null;

    foreach (Customer customer in customers)
    {
        if (IsLucky(customer))
        {
            result = customer;
            break;
        }
    }

    return result;
}
</pre>

This version has more lines of code, more state and more complex flow control. To me it's more complex than the version with multiple returns. We have removed one complexity, and introduced a different one which turns out to be worse.


However, the functional, LINQ version is even shorter and has no local state at all:

<pre class="brush: csharp">public Customer FindLuckyCustomerWithLinq(IEnumerable&amp;lt;Customer&amp;gt; customers)
{
    return customers.FirstOrDefault(c =&amp;gt; IsLucky(c));
}
</pre>
<h3>Fallback</h3>

This pattern has more variation, but the principle is to try and find a result from a first source, falling back to a second or futher sources. The best example of this is data from either cache or database. Early return if a item is found an a cache fits into this pattern.

<pre class="brush: csharp">public Customer GetCustomer(int customerId)
{
    Customer cachedCustomer = customerCache.GetCustomer(customerId);
    if (cachedCustomer != null)
    {
        return cachedCustomer;
    }

    Customer loadedCustomer = dataLayer.GetCustomer(customerId);
    customerCache.Add(customerId, loadedCustomer);
    return loadedCustomer;
}
</pre>

This example with the fallback is interesting: if you re-write it with a single exit point you must introduce a temporary varaible to hold the result. I think that you <strong>cannot </strong>do this code without using one of two things: a temporary variable to hold the value to return at your single exit point, or multiple exit points. The single return increases the amount of temporary mutable state and thus the complexity.

<h2>Links:</h2>

&amp;bull;  There's questions on this topic <a href="http://stackoverflow.com/questions/36707/should-a-function-have-only-one-return-statement">on Stackoverflow</a> and on <a href="http://programmers.stackexchange.com/questions/118703/where-did-the-notion-of-one-return-only-come-from">programmers.Stackoverflow</a>.
 &amp;bull;  See also <a href="http://c2.com/cgi/wiki?SingleFunctionExitPoint">Ward Cunningham's wiki on Single Function Exit point</a>.
&amp;bull;  <a href="http://www.codinghorror.com/blog/2008/07/spartan-programming.html">Codinghorror talked</a> about "<a href="http://ssdl-wiki.cs.technion.ac.il/wiki/index.php/Spartan_programming">Spartan programming</a>", and mentioned "frugal use of control structures, with early return whenever possible."
&amp;bull;  One of the few links on the topic that I could find was <a href="http://www.leepoint.net/JavaBasics/methods/method-commentary/methcom-30-multiple-return.html">Return statements and the single exit fantasy</a>
&amp;bull;  See this discussion on <a href="http://discuss.techinterview.org/default.asp?joel.3.325456.34">Single exit point from function</a> from Joel on Software
&amp;bull;  Take a look at some of the comments on this post <a href="http://thedailywtf.com/Articles/Refactoring_to_the_Max.aspx">"Refactoring to the Max" on the Daily WTF</a> where this issue is raised, particularly this comment:
 
<em>Re: Refactoring to the Max
2006-04-05 02:25 &amp;bull; by John Hensley
            
The main reason for the single return "law" in C is to make sure you clean up memory, locks, handles, etc. before you leave a function. 
This, on the other hand, is Java code.</em>
" />
