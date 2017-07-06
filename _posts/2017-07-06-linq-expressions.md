---
layout: post
title: "LINQ Expressions"
date: 2017-07-06T21:04:19+02:00
series: "LINQ Expressions"
---

It has been quite some time from my last blog post. With getting used to new job
and new found love for mountain hiking I didn't have as much time as I would have
liked for writing. However, now I have a topic that I really like since I worked
with LINQ expressions for some time and I find them really powerful. In this post
I will explain what LINQ expressions are and where they are used.


Chance is you may have used LINQ expressions already, you were just not aware of it.
If you ever used MVC Razor engine with view model binding or Entity Framework, you
were passing LINQ lambda expressions as arguments to various methods. However from
programmer's point of view those arguments were indistinguishable from normal lambdas
and this is the reason why you may have never noticed the difference. So let us look
into what the difference is and where expression trees are used.

## What exactly are expression trees

Expression tree is nothing more than tree structure where each node represents some
kind of expression. Expression can represent many .NET constructs, for example:
- _ConstantExpression_ which represents some constant value, for example value '1'
of type _int_
- _BinaryExpression_ which  represents infix operation over two other expressions,
for example (+) operation in expression '1 + 1'
- _InvocationExpression_ which represents function or method call, for example call
to _ToString_ method in expression 'a.ToString()'
There are currently 25 different built-in types of expressions. But what is common to
all of them is that each expression declares what is resulting type of expression.
This information is then used when constructing expressions which require other
expressions as input. For example, above mentioned _BinaryExpression_ requires two
other expressions for which binary operation is defined. '1 + 1' would be a valid
expression while '1 + "a"' would fail since there is no (+) operation defined
between _int_ and _string_.

Root class for every expression is [System.Linq.Expressions.Expression][1]. This class
also contains factory methods for all built-in expressions. While you can use
those methods to create expression tress, you usually don't have to because compiler
can do that automatically for us. For example consider the following code:
```csharp
Func<int, int> f = x => x + 1;
f(1); // will return 2
```
This is definition of lambda function which accepts _int_ and returns increment.
Compiler will produce function which can then be executed by supplying parameters.

Consider now this code:
```csharp
Expression<Func<int, int>> fexp = x => x + 1;
fexp(1); // ERROR: will not compile
```
The only thing that is different is type declaration. But since we specified that
variable _fexp_ is expression, compiler will transparently emit code for creation of
corresponding expression tree. Above declaration would be equivalent to the following
code:
```csharp
var p = Expression.Parameter(typeof(int), "x");
var fexp = Expression.Lambda(
  Expression.AddChecked(
    p
    Expression.Constant(1)
  ),
  p
);
```
This is quite a mouthful for just a simple lambda expression which increments parameter.
Now imagine what you would have to write if you have a more complex expression.

## What are expression trees used for

What can we do with lambda expressions that we cannot do with lambdas? If you take
lambda _f_ once this is compiled it is essentially a black box. You can supply
lambda with arguments to get result, but there is no way to know at runtime what
this lambda does. In contrast if you take lambda expression _fexp_ which is tree
structure representing expression 'x => x + 1', you can visit nodes at runtime and
see that it will return increment of argument. In fact visiting logic is already
provided for you by [System.Linq.Expressions.ExpressionVisitor][2]. You just need
to extend this abstract class and override methods for those expression types you
are interested in.

Lambda expression has another feature: after inspection you can dynamically compile
it to normal lambda which can be executed. For example:
```csharp
Expression<Func<int, int>> fexp = x => x + 1;
var fcompiled = (Func<int, int>)fexp.Compile();
fcompiled(1); // will return 2
```
Keep in mind though that compilation is not cheap and you should cache result.

You could also transform given expression tree to new expression tree, which you
then wrap in lambda and dynamically compile to executable code. There really are
a lot of possibilities what you can do. But let us look how Razor engine and Entity
Framework utilizes expression trees.

### How expression trees are used in Razor engine

If you ever used MVC and Razor engine you are probably familiar with the following
construct:
```html
@Html.TextBoxFor(m => m.Field1)
```

While first argument looks like normal lambda, if you look at the definition of
method _TextBoxFor_ you would see that it is generic method which accepts lambda
expression which accepts model as input and returns something. With expression
trees, Razor engine can achieve following goals:
- When generating various inputs which are bound to model it can deterministically
generate '_name_' attribute. It does that by walking expression tree of lambda body
and constructing string value which represents given expression. This value then
becomes value of the attribute.
- Since name is a string representation of expression, model binder can use this
value to populate model with data received from client. It does that by walking
model hierarchy as is defined in string representation of expression in '_name_'
attribute.

This way the only thing that developer has to specify is property accessor, everything
else will be done for him automatically. Razor engine will of course also compile
and cache compiled expressions for retrieving model value, which is used when
rendering page, so performance should not suffer.

Note about new tag helpers feature: while you don't see explicit lambdas anymore,
engine still uses expressions in the background. This means that this code:
```html
<input asp-for="Field1" />
```
is equivalent to above code, and underlying lambda expression will be the same.

### What about Entity Framework

Entity Framework translates expression trees to equivalent SQL query which will
return expected data from database. There are two different mechanisms at work:
- Every method in [System.Linq.Queryable][2] class which you use when you are
writing LINQ query is extension method on [System.Linq.IQueryable&lt;T&gt;][4].
Concrete class that implement this interface contains expression tree which is
built by _System.Linq.Queryable_ methods. When it is time to execute query,
expression tree is walked and SQL query is produced.
- Most methods in _System.Linq.Queryable_ require passing lambda expressions.
For example _Where_ method accepts lambda expression where input is element
and output is _true_ if we wish to retain this element. This expressions are also
embedded in expression tree but with special semantic: because they might access
variables outside of expression they are quoted. What this means will be explored
more in depth in future post.

## Conclusion

Expression trees are really a powerful and useful feature which can be used in
many different situations. I hope that expressions are no longer a mystery to
you and you are already thinking where you could use them. In the next post I will
explore how to create custom expressions and how infrastructure handles them.

Thanks for reading.

[1]: https://msdn.microsoft.com/en-us/library/system.linq.expressions.expression(v=vs.110).aspx
[2]: https://msdn.microsoft.com/en-us/library/system.linq.expressions.expressionvisitor(v=vs.110).aspx
[3]: https://msdn.microsoft.com/en-us/library/system.linq.queryable(v=vs.110).aspx
[4]: https://msdn.microsoft.com/en-us/library/bb351562(v=vs.110).aspx
