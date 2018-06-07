---
layout: post
title: "Extending LINQ Expressions"
date: 2017-07-30T19:54:09+02:00
series: "LINQ Expressions"
tags: ["Expression trees", "LINQ", "C#"]
---

In the previous post we looked into what expression trees are and how they can be used.
While this is enough most of the time, you may find yourself wondering if you can
extend expressions. This might especially be the case if you are constructing
expression trees directly via factory methods and you just need to pass some
additional data which is relevant only for your application. In this post we will
look into what is possible and how infrastructure handles extensions.

<!--more-->

## Why would you extend LINQ expression

In my particular case I was constructing expression trees from definition in our
custom DSL (domain specific language). There is one generic type which has
associated external metadata object with available members. Since type itself
did not convey enough information I wanted to store this metadata in expression
itself. This is why I started looking into how I can extend expressions.

Another case where this is used is Entity Framework translating logic. EF Core
processes expressions in two passes:
1. Process LINQ expression into normalized expression tree with objects resolved
from model.
2. Translate normalized expression tree into actual SQL query. This part is provider
specific.
Entity Framework defines some custom expressions which are used in normalized
expression tree. For example there is _ColumnExpression_ which contains data
regarding reference to column.

## Your first custom expression

First of all you cannot extend any of the built-in subclasses of _Expression_ class.
While those classes are public, all constructors are internal and there are also internal
subclasses which are optimized for different usages. What you can do however, is create
new subclass of _Expression_.

Let us implement expression which will produce answer to everything:
```csharp
public class AnswerToEverythingExpression : Expression
{
  public override ExpressionType NodeType => ExpressionType.Extension;

  public override Type Type = typeof(int);
}
```

Property _NodeType_ determines the kind of expression for specific class. For
custom expressions this should always be _ExpressionType.Extension_. While you
could put anything you want in there, there are some components (for example lambda
compiler) which use this property to dispatch calls to appropriate methods and
perform explicit casts. Those casts will fail if you use incorrect kind and you
will get exceptions.

Property _Type_ determines type of expression. This type is used when you are
composing expression to expression trees to check if specific expression can be
used as child of another expression. In our case, we declared that expression is
of type _int_ (we could use any type that can hold 42) and we can use it in all
the places that accept _int_.

Overriding those two properties is enough to create valid expression class. While
our class does not do much, you can use it in expression trees and you can visit
it by overriding _VisitExtension(Expression)_ method in [ExpressionVisitor][1]
class.

## What happens when you call compile on lambda expression

If you tried compiling lambda expression which contained _AnswerToEverythingExpression_
node or tried visiting expression tree with this node without overriding _VisitExtension(Expression)_
method, you will get _ArgumentException_: must be reducible node. This is because
visitor has automatic infrastructure in place for handling extensions:
- when extension node is encountered, check property _CanReduce_ on expression to
see if it can be reduced. If it can call _ReduceAndCheck()_ and repeat this step
on resulting expression.
- If we have reduced extension to non extension node, we can visit this node and
produce appropriate IL code in case of compiler.
- If we are still in extension node, this means that extension cannot be reduced
further and visitor or compiler does not know what to do with it. Only thing that
they can do is throw an exception.

Let's augment our _AnswerToEverythingExression_ class so that you will not get
exceptions:
```csharp
public override bool CanReduce => true;

public override Expression Reduce()
{
  return Expression.Constant(42);
}
```

Now we defined that our custom expression is actually _int_ constant with value 42.
When compiler sees our extension it will reduce it to constant expression for
which it knows how to emit appropriate IL. Similarly visitor without overriden
_VisitExtension(Expression)_ method will know how to visit this expression by reducing
it to constant and visiting constant instead. One thing to note though is that
type of reduced expression must be the same or assignable to the type of extension.
Otherwise you have invalidated previous checks which were done when constructing
expression tree and you will receive an exception.

## Visiting custom expressions

If you create more than one custom expression it can quickly become cumbersome
to check which one was visited inside _VisitExtension(Expression)_. You can avoid
that by creating infrastructure which will plug into default visitor.

First we need a new interface with visitor methods for each of our custom expressions:
```csharp
public interface ICustomExpressionVisitor
{
  Expression VisitAnswerToEverythingExpression(AnswerToEverythingExression node);
}
```

Now we can enhance our _AnswerToEverythingExpression_ class by overriding _Accept(ExpressionVisitor)_
method:
```csharp
protected override Expression Accept(ExpressionVisitor visitor)
{
  return visitor is ICustomExpressionVisitor customVisitor ?
    customVisitor.VisitAnswerToEverythingExpression(this) :
    base.Accept(visitor);
}
```

With this we can now implement this interface in our visitor class and each custom
expression node will be visited in appropriate method without doing any kind of
type checking and explicit casting. To further simplify your code you can create
base visitor class with default implementation of your extensions:

```csharp
public abstract class CustomExpressionVisitor : ExpressionVisitor, ICustomExpressionVisitor
{
  public virtual Expression VisitAnswerToEverythingExpression(AnswerToEverythingExression node)
  {
    return node;
  }
}
```

You can use this class as base class for all your visitors and override only
required methods.

## Transforming expression trees

Expression visitor can also be used to transform one expression tree to another.
All visitor methods return expression which can be the same as received in argument
or some other expression of the compatible type. Built-in expression visitor
contains logic which will create new nodes when child expressions are changed.
To support this, expression kinds with child expressions have _Update_
method which accepts new child expressions, and returns new instance with children
passed in and other parameters (such as method, or member) unchanged.

For example, let us check how _Update_ method is implemented in [MemberExpression][2]:
```csharp
public MemberExpression Update(Expression expression)
{
  if (expression == Expression)
  {
    return this;
  }
  return Expression.MakeMemberAccess(expression, Member);
}
```
Since all expressions are immutable (and yours should be too), we know just by
comparing given object expression to the current if we need to create new instance.
If they are the same, we just return current instance and avoid new allocation
and cascading effect of creating new parent expressions. If they are different,
we return new instance of _MemberExpression_ with new object expression and existing
member. This method is then called
in [ExpressionVisitor][3]:
```csharp
protected internal virtual Expression VisitMember(MemberExpression node)
{
  return node.Update(Visit(node.Expression));
}
```
Visitor will call visit on child expression and then use return value as argument
to _Update_ method. It will then return resulting expression which will be the same
if child did not change, or new instance if child has changed.

If you are creating complex expressions and you require transformation capabilities,
you should implement the same logic in your expressions as well. This way existing
and custom expressions will work as expected.

## Just another tool in the toolbox

While you probably will not go creating new custom expressions in every one of your
applications, I hope that I have shown you how you can implement them if needed.
This can be especially useful if you are working with custom DSLs which are to be
executed in .NET runtime. For me it was also an interesting topic about internals
of LINQ expressions which I have often used but never really knew how they work.

Thanks for reading.

[1]: https://msdn.microsoft.com/en-us/library/system.linq.expressions.expressionvisitor(v=vs.110).aspx
[2]: https://github.com/dotnet/corefx/blob/master/src/System.Linq.Expressions/src/System/Linq/Expressions/MemberExpression.cs#L79
[3]: https://github.com/dotnet/corefx/blob/master/src/System.Linq.Expressions/src/System/Linq/Expressions/ExpressionVisitor.cs#L371
