# Parameter Types

I want to look at a particular pattern in C# code, about collection parameter types. We are reminded that if we e.g. have a method such as:

```csharp
public List<ProcessResult> ProcessCustomers(List<Customer> customers)
```

This would be more useful if `List<T>` is lowered to a base type, typically `IEnumerable<T>`. So this is often turned into:

```csharp
public IEnumerable<ProcessResult> ProcessCustomers(IEnumerable<Customer> customers)
```

Is this more useful? Yes and no. The rule does not apply to the return value, it has been _over-applied_.  IMHO, It should be:

```csharp
public List<ProcessResult> ProcessCustomers(IEnumerable<Customer> customers)
```

When we change the type of the parameter to the method from `List<Customer>` to `IEnumerable<Customer>` we make the method more useful,
 as the weaker contract is easier to meet. So when I have code like this, the final `ToList()` becomes optional:

 ```csharp
  var customersToProcess = someCustomers
    .Where(c => SomePredicate(c))
    .ToList(); // not needed any more

  // I can now process a list, an array or a LINQ query iterator
  // as all of them are IEnumerable<Customer> 
  var results = ProcessCustomers(customersToProcess);
 ```

Then we change the return value  `List<ProcessResult>` to `IEnumerable<ProcessResult>` and it's not the same. We might need to do:

```csharp
  var results = ProcessCustomers(customersToProcess)
    .ToList(); // I need a list, thanks
```

The weaker returned type means that the caller can do fewer things with it. It's the reverse of a parameter!

## Inversion

If I have a method typed:

```csharp
ProcessResult DoTheThing(ParamType someParam)
```

then I can call it with `ProcessResult aValue = DoTheThing(paramValue);`.

But I can also use a _subtype_ of `ParamType` for the input parameter and - note the inversion - assign it to a _base type_ of `ProcessResult`. e.g. `object resultValue = DoTheThing(subTypeParamValue);`.

So I get the most value when the input types are a more base type (with more sub types) and the return type is a more derived type (with more base types).

## Variance

The "smell" that alerted me to this difference was an abundance of methods that made a `List<T>`, return it typed as an `IEnumerable<T>`, to callers who immediately call `.ToList()` on it,
because they need the list functionality back, or have the warning about "Possible multiple enumeration of IEnumerable".

This is a bit like [Covariance and Contravariance](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/covariance-contravariance/).
C# uses the keywords [`in`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/in-generic-modifier) and [`out`](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/out-generic-modifier) for these, 
because `in` (contravariant, using a less derived type) is used for input values, but  the inverse, `out` (covariant, using a more derived type) is used for returning values.

## Conclusions

The practice of using a base type (with a weaker contract) for input parameters to a method where possible, does not also apply to the return value. It is inverted.

The input parameters to a method should be the weakest type that the method can easily use. e.g. if we lower the parameter type from e.g. `List<Customer> customers` to `IEnumerable<Customer> customers` and there are no issues with the method's code, then do that. This gives the caller maximum flexibility of values to give to the method.

However, the return value to the method can be the strongest type that the method can easily return. e.g. if the results are a list, then return it as a list e.g. `List<ProcessResult>` instead of typing it as  `IEnumerable<ProcessResult>`. If we can do this without changing to code or introducing a `.ToList()` that you might not want, consider doing it. This gives the caller maximum flexibility of what to do with the results.

**Intent** matters too, once you are no longer always returning the lowest type possible. Are the results a list generated in the method? Then return it as a list. Is the list actually part of the object state? Then you probably don't want the caller using the list methods `.Add()` and `.Remove()` on it, so you should return a copy, a `IReadOnlyCollection<T>` or just an `IEnumerable<T>`.
  
  But, don't force the method return type if it can't "easily return" as said above. If the method is e.g. part of a LINQ pipeline, then leave it as an `IEnumerable<T>` return and allow the caller choice of when to reify it.
