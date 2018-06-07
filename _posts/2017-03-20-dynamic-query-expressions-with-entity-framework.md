---
layout: post
title: "Dynamic Query Expressions With Entity Framework"
date: 2017-03-20T19:23:27+01:00
tags: ["Expression trees", "Entity Framework", "LINQ", "Runtime", "C#"]
---

When you use Entity Framework for data access and you need to retrieve data
from data store, you usually write LINQ expression. LINQ provides fluent syntax
for expressing what data we are interested in. And since syntax is the same for
querying in-memory collection or external data store, I never really look into
what the difference is and how queries are actually constructed. Until the day
that I was designing API which allowed client to sort on any field. While I could
create if/else or switch statement covering all possibilities, this would be
boring, error prone and unmaintainable. And there is a better way.

<!--more-->

Let us start with simple LINQ query which has order by clause applied. This might
look something like this:

```csharp
IQueryable<Person> = context.Persons.OrderBy(p => p.FirstName);
```

_System.Linq.IQueryable&lt;T&gt;_ represents query definition. To build/change definition
we use extension methods from _System.Linq.Queryable_. I will not go into more
details what _IQueryable&lt;T&gt;_ exactly is and how it differs from _IEnumerable&lt;T&gt;_ --
you may want to check out [this Stack Overflow answer][3]. What we are interested
in is definition and implementation of _[System.Linq.Queryable.OrderBy][1]_ method:

```csharp
public static IOrderedQueryable<TSource> OrderBy<TSource, TKey>(
    this IQueryable<TSource> source,
    Expression<Func<TSource, TKey>> keySelector)
{
    if (source == null)
        throw Error.ArgumentNull("source");
    if (keySelector == null)
        throw Error.ArgumentNull("keySelector");
    return (IOrderedQueryable<TSource>) source.Provider.CreateQuery<TSource>(
        Expression.Call(
            null,
            GetMethodInfo(Queryable.OrderBy, source, keySelector),
            new Expression[] { source.Expression, Expression.Quote(keySelector) }
          ));
}
```

There are two important things to see here. First is that method expects parameter
of type _Expression&lt;Func&lt;TSource, TKey&gt;&gt;_ instead of _Func&lt;TSource, TKey&gt;_.
What this means is that compiler generates expression tree automatically for given
argument instead of compiling it to executable code. This way developer does not
have to generate expression trees by hand. But they can still be created in code
if desired and we will use this with dynamic order by statement. If you want to
know more about expression trees I recommend reading [Expression Trees (C#) -- MSDN][2].

Second important thing to notice is that this method creates new expression tree
out of existing expression to which _OrderBy_ call is applied. This expression tree is
then used as argument to _System.Linq.IQueryProvider.CreateQuery&lt;TSource&gt;_.
We can reuse this logic in our function for dynamic order by.

## Building property accessor expressions

If we analyze lambda expression that is being passed to _OrderBy_ function, we see
that it is constructed from:
1. Parameter with specific type and name _p_
2. Member access expression to member with name _FirstName_
3. This is wrapped into lambda expression with single parameter _p_

Since we want to be able to sort by any property for specific type it is best to
cache constructed expression trees so that we don't have to construct them every
time:

```csharp
public static class PropertyAccessorCache<T> where T : class
{
    private static readonly IDictionary<string, LambdaExpression> _cache;

    static PropertyAccessorCache()
    {
        var storage = new Dictionary<string, LambdaExpression>();

        var t = typeof(T);
        var parameter = Expression.Parameter(t, "p");
        foreach (var property in t.GetProperties(BindingFlags.Public | BindingFlags.Instance))
        {
            var propertyAccess = Expression.MakeMemberAccess(parameter, property);
            var lambdaExpression = Expression.Lambda(propertyAccess, parameter);
            storage[property.Name] = lambdaExpression;
        }

        _cache = storage;
    }

    public static LambdaExpression Get(string propertyName)
    {
        LambdaExpression result;
        return _cache.TryGetValue(propertyName, out result) ? result : null;
    }
}
```

All work of constructing possible property accessors is done inside static
constructor which is executed only once for each type _T_. We store constructed
accessors into a dictionary so that we can access them by property name. In above
helper this access is case sensitive -- if you wish to have case insensitive API,
then specify _StringComparer.OrdinalIgnoreCase_ as argument to dictionary constructor.

## Constructing final expression tree for ordering

Now that we have accessor expressions for all properties we can write
extension method for applying order by statement by property name:

```csharp
public static IQueryable<T> ApplyOrderBy<T>(
    this IQueryable<T> source,
    string propertyName,
    out bool success) where T : class
{
    success = false;
    var expression = PropertyAccessorCache<T>.Get(propertyName);
    if (expression == null) return source;

    success = true;
    MethodCallExpression resultExpression = Expression.Call(
        typeof(Queryable),
        nameof(Queryable.OrderBy),
        new Type[] { typeof(T), expression.ReturnType },
        source.Expression,
        Expression.Quote(expression));
    return source.Provider.CreateQuery<T>(resultExpression);
}
```

Since property type is not known at compile time, we have to use another overload
for _Expression.Call_ where we let factory method to locate appropriate method by
explicitely providing type arguments. If this code is frequently executed it may
be better to find _System.Reflection.MethodInfo_ explicitely and cache it for later
use.

## How about dynamic filtering

Dynamic filtering is more complicated than order by statement since we have multiple
possible operators and we have to bring parameter value from outside. In this example
I will present solution using equality operator. Others can be trivially added with
_if/else_ or _switch_ statement.

We will be using _PropertyAccessorCache&lt;T&gt;_ utility class defined earlier in
this post. Cached member access expressions will be used as starting point for
filter expression on that particular property. Steps are as follows:
1. Retrieve member access expression for property
2. Try parsing/converting value to property type
3. Construct expression tree which will be used in filter
4. Construct new query with filter constructed in previous step

```csharp
public static IQueryable<T> ApplyWhere(
    this IQueryable<T> source,
    string propertyName,
    object propertyValue,
    out bool success) where T : class
{
    // 1. Retrieve member access expression
    success = false;
    var mba = PropertyAccessorCache<T>.Get(propertyName);
    if (mba == null) return source;

    // 2. Try converting value to correct type
    object value;
    try
    {
        value = Convert.ChangeType(propertyValue, mba.ReturnType);
    }
    catch (SystemException ex) when (
        ex is InvalidCastException ||
        ex is FormatException ||
        ex is OverflowException ||
        ex is ArgumentNullException)
    {
        return source;
    }

    // 3. Construct expression tree
    var eqe = Expression.Equal(
        mba.Body,
        Expression.Constant(value, mba.ReturnType));
    var expression = Expression.Lambda(eqe, mba.Parameters[0]);

    // 4. Construct new query
    success = true;
    MethodCallExpression resultExpression = Expression.Call(
        null,
        GetMethodInfo(Queryable.Where, source, (Expression<Func<T, bool>>)null),
        new Expression[] { source.Expression, Expression.Quote(expression) });
    return source.Provider.CreateQuery<T>(resultExpression);
}

private static MethodInfo GetMethodInfo<T1, T2, T3>(
    Func<T1, T2, T3> f,
    T1 unused1,
    T2 unused2)
{
    return f.Method;
}
```

There is quite a bit more code than in order by method. Let's get over some points
of interest. You will notice that I am using _GetMethodInfo_ to retrieve _MethodInfo_
for _Queryable.Where_. Since all type arguments are known at compile time we can
locate desired method the same way as it is implemented in [order by source code][1].

Second point of interest is how parameter is passed into expression tree (under
section 3.). We construct constant expression of correct type and value. While
this is simple it has undesirable side effect -- Entity Framework generates
query without using query parameter. This can cause problems in database engine
by polluting query plan cache with query plans which differ from one another only
in parameter value. We can force parameter creation by "hoisting" local variable
into instance variable and passing that into constant expression:

```csharp
// 3. Construct expression tree
var wc = Expression.Property(
    Expression.Constant(new { Value = value }),
    "Value");
var eqe = Expression.Equal(
    mba.Body,
    Expression.Convert(wc, mba.ReturnType));
var expression = Expression.Lambda(eqe, mba.Parameters[0]);
```

Since property type is not known at compile time, we have to insert convert
expression to convert parameter type from object to appropriate type.

In this rather lengthy post I have shown how to dynamically add order by and
filter statements to LINQ queries. You can construct other statements such as
selects and joins as well -- it just requires constructing appropriate expression
trees.

Thanks for reading.

[1]: https://github.com/Microsoft/referencesource/blob/master/System.Core/System/Linq/IQueryable.cs#L326
[2]: https://msdn.microsoft.com/en-us/library/mt654263.aspx
[3]: http://stackoverflow.com/questions/252785/what-is-the-difference-between-iqueryablet-and-ienumerablet
