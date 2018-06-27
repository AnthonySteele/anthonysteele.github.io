# The Single Return Law

I wrote this a few years ago (before 2012), about the idea that a method or function should have only one "exit point", i.e at most one return statement. It is now hosted here for reference.  Fortunately, this "law" seems to becoming less common. The article is still generally my opinion. To sum up:

* Do you code in C or similar old-school, low-level language? If so, stop reading now because the rest of this article does not apply to your practices. The arguments in favour of the single-return style originated in the C programming language for reasons of manual resource management. But these reasons are irrelevant to Java, C#, JavaScript, Ruby, Python etc. 

* There is a high bar to clear to call something a "law" and the idea that "a method should have at most one return statement" does not meet it for modern languages. There is no formal study that shows that this rule leads to safer, more readable or otherwise better code in these languages. It is therefore just a style. 

* It is not a useful style when applied as a blanket rule. There are cases where a single return is more readable or simpler, and cases where it isn't.  If you learn when and how to use multiple returns, you can write more expressive code. 

* Do not blindly follow cargo-cult rules. 

## Objections

At the time the blog post attracted some comments. So this style must be an "important" issue to coders, like [tabs vs. spaces](https://www.youtube.com/watch?v=SsoOG6ZeyUI).

* Most of the outraged, dogmatic replies that I have received are along the lines of "but in _this_ case a single return is better." No doubt, but that is no contradiction to what I am saying: Both styles have uses, so learn to apply both, and choose which best fits your case. If there _is_ resource de-allocation or logging at the end of the method that cannot be handled by a block structure of the language then you might prefer a single exit point _in that case_.

* Examples of long, confusing methods are not an argument either way. You can write them in either style, and the cure is the same: refactor to extract methods. 

* If you somehow still think that the single-return rule is applicable across all languages, please read about [match expressions in F#](https://docs.microsoft.com/en-us/dotnet/articles/fsharp/language-reference/match-expressions), [Erlang's case expressions](http://erlang.org/doc/reference_manual/expressions.html#id80767) or [Haskell's pattern matching](https://en.wikibooks.org/wiki/Haskell/Pattern_matching) and get back to me. In those constructs, you cannot avoid using multiple return values. A [Rust `match` expression](https://doc.rust-lang.org/book/match.html) works similarly too.

## Original blog post

It is sometimes said that a method should have only one return statement (i.e. one exit point) and that to code using more than one return per method is bad practice. It is claimed to be a risk to readability or a source of error. Sometimes this is even given the title of the "single exit point law".  Actually, if you want to call something a law, I'd expect some evidence for it. I do not know of any formal study that back this "law" up, or of any study of the multiple-return pattern at all. This makes it a "preference" not a "law". And it's a preference that I do not hold for c# code. Or java, ruby or python code either.

This coding rule dates back to [Dijkstra's structured programming](http://en.wikipedia.org/wiki/Structured_programming). However it is outdated, if it ever ever was a "law" or "principle". The single return rule originates in the era of C, FORTRAN or assembler, where it was common to allocate resources (most frequently memory, file handles or locks) at the start of the procedure, and to de-allocate them at the end of it. An early return can lead the programmer  to either forget to do the cleanup code and cause a memory leak or locked file, or to maintain cleanup code in two places. So it makes some sense to stick to just one return right at the end of the method.

But in the modern world, this is no longer so. Firstly, garbage collected languages make explicit deallocation unnecessary in most cases - memory is reclaimed automatically. Secondly, `try...finally` blocks and `using` statements allow release of other resources to happen with greater certainly at the end of any block of code when it is needed.

In any event, a function with multiple exit points is a far lesser issue than [a goto](http://www.u.arizona.edu/~rubinson/copyright_violations/Go_To_Considered_Harmful.html). In some cases it is the simplest way to code now that we have control structures to deal with it.  If your method is long, complex and has multiple returns, consider splitting it up into smaller, well-named methods with a single-responsibly each. Also consider doing that if it's long, complex and has only one return.


My summary of the matter is:


* There are cases where a single return is more readable, and cases where it isn't. See if it reduces the number of lines of code, makes the logic clearer or reduces the number of braces and indents or temporary variables.

* More than one return style may be a bad habit in C code, where resources have to be explicitly de-allocated, but in languages that have automatic garbage collection and `try...finally` blocks and `using` blocks, this argument does not apply - in these languages, it is very uncommon to even need centralised manual resource deallocation.
 
* There is no law requiring only one exit point for a method, and no evidence for it as a rule of thumb. Some people have an unsubstantiated belief in the superiority of this style.

Therefore, use as many returns as suits your artistic sensibilities, because it is a layout and readability issue, not a technical one.

There's another issue at play here, about rules being adhered to without people grasping the reason why they are adhering to them, and thus keeping the rule in force after the need for it has evaporated. When you give a rule, also give the reasons for it, so that people can apply their own judgement as to if it is still applicable.

## Common patterns of early return

### The guard clause
```csharp 
public Foo merge(Foo a, Foo b)
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
```

It is fine, even in C code to use early return as a first check on the parameters after entering a function, before allocating resources and doing the main work of the routine. It's more readable than embedding the rest of the function in an if-block. Bear in mind that if the parameters are actually in error, throwing an exception is probably better than returning a default value.


The pattern is [well explained here on Ward Cunningham's wiki](http://www.c2.com/cgi/wiki?GuardClause). The discussion there diverges into a more general on on "early return vs single return", with the interesting point that single return requires mutable state in the form of a temporary "result" variable, thus early return, not requiring this piece of mutable state, seems "cleaner" to a functional programming perspective.

### Found item in loop

Finding an item in a loop using returns was common in C#, but is often now giving way to the more concise LINQ version.  However, beware of returns in the middle of highly nested code or in methods that are more than just a loop (if posible, factor out the loop into a seperate method).  All other things being equal, avoid them, since they are less readable and predictable. However, the following is a valid use of multiple returns:

```csharp 
public Customer FindLuckyCustomer(IEnumerable<Customer> customers)
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
```

Note how `FindLuckyCustomer` does not require a result local variable, just the `customerItem` for the loop. The local variable would be "mutable state" for the method. If you have looked at all at functional programming, you can appreciate how eliminating unnecessary mutable state can be a good thing. Here's the code that uses the same loop but avoid multiple returns:

```csharp
public Customer FindLuckyCustomer(IEnumerable<Customer> customers)
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
```

This version has more lines of code, more state and more complex flow control. To me it's more complex than the version with multiple returns. We have removed one complexity, and introduced a different one which turns out to be worse.


However, the functional, LINQ version is even shorter and has no local state at all:

```csharp
public Customer FindLuckyCustomerWithLinq(IEnumerable<Customer> customers)
{
    return customers.FirstOrDefault(c => IsLucky(c));
}
```

### Fallback

This pattern has more variation, but the principle is to try and find a result from a first source, falling back to a second or futher sources. The best example of this is data from either cache or database. Early return if a item is found an a cache fits into this pattern.

```csharp
public Customer GetCustomer(int customerId)
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
```

This example with the fallback is interesting: if you re-write it with a single exit point you must introduce a temporary variable to hold the result. I think that you *cannot* do this code without using one of two things: a temporary variable to hold the value to return at your single exit point, or multiple exit points. The single return increases the amount of temporary mutable state and thus the complexity. It's a trade-off.

## Links

*  There's questions on this topic [on Stackoverflow](http://stackoverflow.com/questions/36707/should-a-function-have-only-one-return-statement) and on [Software Engineering StackExchange](http://softwareengineering.stackexchange.com/questions/118703/where-did-the-notion-of-one-return-only-come-from).
* See also [Ward Cunningham's wiki on Single Function Exit point](http://c2.com/cgi/wiki?SingleFunctionExitPoint).
* [Codinghorror talked about Spartan programming](http://www.codinghorror.com/blog/2008/07/spartan-programming.html), and mentioned "frugal use of control structures, with early return whenever possible."
*  One of the few links on the topic that I could find was [Return statements and the single exit fantasy](http://www.leepoint.net/JavaBasics/methods/method-commentary/methcom-30-multiple-return.html)
* [Single exit point from function, on 
The Joel on Software Discussion Group](http://discuss.joelonsoftware.com/default.asp?joel.3.325456.34)


## Second blog post

My blog post on multiple returns and why they are actually quite useful has a small but enduring popularity (some of it negative) among those who [obsess over minutia](http://en.wikipedia.org/wiki/Parkinson's_Law_of_Triviality). Yes, that includes me sometimes.


For instance [this blog post](http://npclaudiu.blogspot.co.uk/2012/04/my-two-cents-on-one-return-only-policy.html) which argues for single returns, with an example where a single return is better.


What's interesting about the example is that it introduces a step that is taken at the exit point. This is the main reason for having a single exit point - to better control what happens there. Given that Java, C# et al seldom need cleanup on exit (and have constructs for this - `try ... finally` and `using`) the author of that piece has found another case where something happens on exit - logging.


I'm sure there are cleverer ways to do the logging on exit - maybe something based on attributes and/or [postsharp](http://www.sharpcrafters.com/) to inject the right code. But if you don't already have those in place, the code given is a reasonable way to do a reasonable requirement without over engineering it.


The assertions that the single return is "more readable" in all cases though, are something that I disagree with. So often, what a person finds more readable is just a matter of what they are used to reading. If it's not backed up by concrete measurements (e.g. lines of code or [cyclomatic complexity](http://en.wikipedia.org/wiki/Cyclomatic_complexity)) or statistically significant studies on third parties who start out without a preference, then it's very susceptible to bias in favour of what's familiar to the reader. I think this is one of those times.

One return is more readable than two in the sense that one statement is easier to read than two, but I'm in favour of multiple returns in cases where it does away with other complexity and reduces the overall line count. Keeping track of multiple returns is harder than keeping track of one at the end (especially if you're not used to it), but keeping track of local variables also has a cost.  You trade one for the other based on what you prefer.


How would I do this code?


Suppose the code at first does not have the requirement for logging. Three ways of coding it are given. I don't like big nested `? :` blocks - my personal opinion is that they are too terse for readability. So I'd start with #2 , but removing some superfluous else keywords:

```csharp
// simple verson without logging
public Seniority GetSeniority(int yearsOfExperience)
{
  if (yearsOfExperience>= minSeniorYears)
  {
    return Seniority.Senior;
  }
            
  if (yearsOfExperience>= minMiddleYears)
  {
    return Seniority.Middle;
  }

  return Seniority.Junior;
}
```

You could argue that the single return version is more extensible (as it supports the addition of logging). But [YAGNI](http://www.codinghorror.com/blog/2004/10/kiss-and-yagni.html).  Don't add in extra lines of code to support some requirement that you might have at some time. You can add that when you do need it, it's not hard to do so.


If you do get the requirement for logging, going to the version with logging and one exit point is good code, in my opinion. It does what it should do, with little fuss. I'd let it stand with minor simplification.

```csharp
// one-return verson with logging
public Seniority GetSeniority(int yearsOfExperience)
{
   Seniority result;
             
   if (yearsOfExperience >= minSeniorYears)
   {
     result = Seniority.Senior;
   }
   else if (yearsOfExperience >= minMiddleYears)
   {
     result = Seniority.Middle;
   }
   else
   {
     result = Seniority.Junior;
   }
             
    Log("GetSeniority", yearsOfExperience, result);         
    return result;
}
```

But there is another way. This method does two things - it calculates the value, and as a side-effect, logs it at the end. Which got me thinking of the trade-offs. Multiple returns is best done when the method is a function with no side-effects and minimal state. Encouraging methods like that is a good thing. I find it easy to read the multiple returns because what is returned is all you need to know about the method; it doesn't do anything else.

But side-effects are necessary, so where they occur, it may be a reason to avoid multiple returns in that code. This leads me to another way to do it:

```
// separating the concerns of calculation and logging
private Seniority CalcSeniority(int yearsOfExperience )
{
  if (yearsOfExperience >= minSeniorYears)
  {
    return Seniority.Senior;
  }
     
  if (yearsOfExperience >= minMiddleYears)
  {
    return Seniority.Middle;
  }
     
  return Seniority.Junior;
}
     
public Seniority GetSeniority(int yearsOfExperience)
{
  Seniority result = CalcSeniority(yearsOfExperience);
  Log("GetSeniority", yearsOfExperience, result);
  return result;
}
```

So the part of the code that can be just a function is that, and where there has to be state and a side-effect, this state is kept for only three lines of code with only one return. Also good. It's one line longer so no improvement there. But we do get to name something that we didn't get to before.

Does this really matter? In the code given, no. It's just a simple example. But picking single return or multiple returns as need be is a tool for keeping unnecessary complexity down in longer code.


