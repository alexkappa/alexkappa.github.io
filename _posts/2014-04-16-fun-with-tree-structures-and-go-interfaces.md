---
layout: post
title: Fun with binary expression trees and Go
category: go
author: alex
---

Recently, I found myself trying to implement a mechanism, able to evaluate expressions. Essentially what I needed was a [binary expression tree](http://en.wikipedia.org/wiki/Binary_expression_tree). I used Go for this project, and quickly came to a delightful surprise that using Go's extremely powerful interface mechanisms, it became extremely easy to implement and extend.
{:.lead}

## Before we dive in

From Wikipedia:

> A binary expression tree is a specific application of a binary tree to evaluate certain expressions. Two common types of expressions that a binary expression tree can represent are algebraic and boolean. These trees can represent expressions that contain both unary and binary operators.

> The leaves of a binary expression tree are operands, such as constants or variable names, and the other nodes contain operators.

> These particular trees happen to be binary, because all of the operations are binary, and although this is the simplest case, it is possible for nodes to have more than two children. It is also possible for a node to have only one child, as is the case with the unary minus operator. An expression tree, T, can be evaluated by applying the operator at the root to the values obtained by recursively evaluating the left and right subtrees.

So a binary expression tree

- can be algebraic (addition, multiplication, etc) or boolean (and, or, not).
- has operands as leaves and operators as the other nodes.
- node usually has two children, but it may have more (or one).

## So how are we going to use them?

Our need for binary expression trees arose by wanting to evaluate whether a set of url query parameters matches a particular pattern. So the pattern `date >= 2014-01-01` should evaluate for `/path?date=2014-10-10` but should not evaluate for `/path?date=2013-12-12`. Alright that was too easy. How about combining conditions together with logical operators?

    (date >= 2014-01-01) AND (lang == "EN")

And a little more complex expressions by combining logical operators:
(date >= 2014-01-01) AND ((lang == "EN") OR NOT (foo = bar))

How do we go on from here? Well lets construct a boolean expression tree that represents this arbitrary handwritten expressions.

![alt](/assets/img/content/fun-with-tree-structures-and-go-interfaces/binary_expression_tree_example-2.png)

## Okay, lets code

To get things started, it makes sense to see every node in the tree as an object, whose children are themselves objects. Starting from the top, the `AND` node could look like this:

```go
type And struct {
	Left, Right interface{}
}
```

Now, although this looks simple enough, we haven't yet addressed the trees functionality, which is why we need such a tree in the first place. Remeber we need to evaluate a set of query parameters against this tree and the response would be a simple true or false.

So lets add this to the topmost node we just created.

```go
func (a And) Eval(p map[string]string) bool {
	// ...
}
```

The `And` struct has a method called `Eval` which we can supply with parameters and it should tell us if the parameters match the conditions or not.

Now we need to write the implementation of the Eval method which should evaluate it's children.

```go
func (a And) Eval(p map[string]string) bool {
	return a.Left.Eval(p) && a.Right.Eval(p)
}
```

Can you already see where I'm going with this? Struct `And` is a node in the tree, which has two children, themselves nodes in the tree. All of them have an `Eval` that take parameters as arguments and return a boolean.

If `And`, `Or`, `Not` and all the operators need to have an `Eval` function, we can group them by a common interface? Let's call that interface `Node`.

```go
type Node interface{
	Eval(p map[string]string) bool
}
```

Let's change the `And` struct to use Nodes as it's children instead of arbitrary interfaces.

```go
type And struct {
	Left, Right Node
}
```

Now the `Eval` method we defined on `And` is correct and would compile and run correctly. However our tree is incomplete as we haven't defined `Or`, `Not` or any operators yet.

```go
type Or struct {
	Left, Right Node
}

func (o Or) Eval(p map[string]string) bool {
	return o.Left.Eval(p) || o.Right.Eval(p)
}

type Not struct {
	Elem Node
}

func (n Not) Eval(p map[string]string) bool {
	return !n.Elem.Eval(p)
}
```

With the above code we are able to construct and evaluate boolean expressions. To showcase an example, we'll need a couple of helper structs, `True` and `False` which always evaluate to `true` and `false` respectively.

```go
type True struct{}

func (t True) Eval(p map[string]string) bool {
	return true
}

type False struct{}

func (f False) Eval(p map[string]string) bool {
	return false
}
```

Let's try it out:

```go
t1 := And{True{}, True{}}
t1.Eval(nil) // returns true

t2 := Or{False{}, True{}}
t2.Eval(nil) // returns true
```

Next we can define algebraic expressions like `equals`, `greater than` and `less than`.

```go
type Eq struct{ Key, Value string }

func (eq Eq) Eval(p map[string]string) bool {
	if val, found := p[eq.Key]; found {
		return val == eq.Value
	}
	return false
}

type Gt struct{ Key, Value string }

func (gt Gt) Eval(p map[string]string) bool {
	if val, found := p[gt.Key]; found {
		return val > gt.Value
	}
	return false
}

type Lt struct{ Key, Value string }

func (lt Lt) Eval(p map[string]string) bool {
	if val, found := p[lt.Key]; found {
		return val < lt.Value
	}
	return false
}
```

Now lets construct the tree we drew out initially using the structures we created, and evaluate some parameters against it.

```go
tree := And{
	Eq{"date","2014-01-01"},
	Or{
		Eq{"lang","EN"},
		Not{Eq{"foo","bar"}},
	},
}

p := map[string]string{
	"date": "2014-01-01",
	"lang": "EN",
}

tree.Eval(p) // evaluates to true
```

I've published the examples on [GitHub](http://github.com/alexkappa/tree), so feel free to fork the repo and adapt it to your needs.

## Final thoughts

Go's interfaces made tree nodes easliy interchangeable and extremely easy to construct. Additionally, tree nodes can be added without any additions to the package. If you need to create your own operators (for example `>=`, `<=`, `before`, `after`) all you have to do is define an `Eval` method for it and it implicitly becomes a `Node`.

I've tried to keep the examples easy to follow, so I haven't talked at all about operator values and type casting. For example you might want to implement a `before` or `after` operator that acts on dates, or a `contains` operator that acts on strings. The implementation of an operator is completely up to you, so if you need to type cast, or parse a date you are free to do so. A quick example:

```go
type Before struct{ Key, Value string }

func (b Before) Eval(p map[string]string) bool {
	if val, found := p[b.Key]; found {
		ta, ea := time.Parse("2006-01-02", val)
		tb, eb := time.Parse("2006-01-02", b.Value)
		if ea != nil || eb != nil {
			return false
		}
		return ta.Before(tb)
	}
	return false
}

```

Another thing I didn't mention is that parameters in my use case came from the requests URL query parameters, meaning their type is [`url.Values`](http://golang.org/pkg/net/url/#Values). `url.Values` is a map itself with additional getter and setter methods. So I adapted the `Node` interface's arguments from `map[string]string` to `Parameters`. And `Parameters` is an interface defined like so:

```go
type Parameters interface {
	Get(key string) string
}
```

So instead of transforming a requests query parameters into a `map` before evaluating, I can simply pass a `url.Values` to `Eval` instead.

**Update**: The source code is available on [GitHub](https://github.com/alexkappa/go-exp) although it's fairly modified and the API is changed quite heavily.

Thanks for reading!
