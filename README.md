## Machinist

[![Build Status](https://travis-ci.org/typelevel/machinist.svg?branch=master)](https://travis-ci.org/typelevel/machinist)

> "Generic types and overloaded operators would let a user code up all
> of these, and in such a way that they would look in all ways just
> like types that are built in. They would let users grow the Java
> programming language in a smooth and clean way."
>
> -- Guy Steele, "Growing a Language"

### Overview

One of the places where type classes incur some unnecessary overhead
is implicit enrichment. Generic types have very few methods that can
be called directly, so Scala uses implicit conversions to enrich these
types with useful operators and methods.

However, these conversions have a cost: they instantiate an implicit
object which is only needed to immediately call another method. This
indirection is usually not a big deal, but is prohibitive in the case
of simple methods that may be called millions of times.

Machinist's ops macros provide a solution. These macros allow the same
enrichment to occur without any allocations or additional
indirection. These macros can work with most common type class
encodings, and are easily extensible. They can also remap symbolic
operators (e.g. `**`) to text names (e.g. `pow`).

Machinist started out as part of the
[Spire](http://github.com/non/spire) project.

For a more detailed description, you can read this
[article](http://typelevel.org/blog/2013/10/13/spires-ops-macros.html)
at [typelevel.org](http://typelevel.org).

### Examples

Here's an example which defines a very minimal typeclass named `Eq[A]`
with a single method called `eqv`. It is designed to support type-safe
equals, similar to `scalaz.Equal` or `spire.algebra.Eq`, and it is
specialized to avoid boxing primtive values like `Int` or `Double`.

```scala
import scala.{specialized => sp}

import machinist.DefaultOps

trait Eq[@sp A] {
  def eqv(lhs: A, rhs: A): Boolean
}

object Eq {
  implicit val intEq = new Eq[Int] {
    def eqv(lhs: Int, rhs: Int): Boolean = lhs == rhs
  }

  implicit class EqOps[A](x: A)(implicit ev: Eq[A]) {
    def ===(rhs: A): Boolean = macro DefaultOps.binop[A, Boolean]
  }
}

object Test {
  import Eq.EqOps

  def test(a: Int, b: Int)(implicit ev: Eq[Int]): Int =
    if (a === b) 999 else 0
}
```

Here are some intermediate representations for how the body of the
`test` method will be compiled:

```scala
// our scala code
if (a === b) 999 else 0

// after implicit resolution
if (Eq.EqOps(a)(Eq.intEq).===(b)) 999 else 0

// after macro application
if (Eq.intEq.eqv(a, b)) 999 else 0

// after specialization
if (Eq.intEq.eqv$mcI$sp(a, b)) 999 else 0
```

There are a few things to notice:

1. `EqOps[A]` does not need to be specialized. Since we will have
removed any constructor calls by the time the `typer` phase is over,
it will not introduce any boxing or interfere with specialization.

2. We did not have to write very much boilerplate in `EqOps` beyond
specifying which methods we want to provide implicit operators for. We
did have to specify some type information though (in this case, the
type of `rhs` (the "right-hand side" parameter) and the result type.

3. `machinist.DefaultOps` automatically knew to connect the `===`
operator with the `eqv` method, since it has a built-in mapping of
symbolic operators to names. You can use your own mapping by extending
`machinist.Ops` and implementing `operatorNames`.

### Including Machinist in your project

Machinist supports Scala 2.10, 2.11, 2.12, and 2.13.0-M3. If you have
an SBT project, add the following snippet to your `build.sbt` file:

```scala
libraryDependencies += "org.typelevel" %% "machinist" % "0.6.4"
```

Machinist also supports Scala.js. To use Machinist in your Scala.js
projects, include the following `build.sbt` snippet:

```scala
libraryDependencies += "org.typelevel" %%% "machinist" % "0.6.4"
```

### Shapes supported by Machinist

Machinist has macros for recognizing and rewriting the following
shapes:

```scala
// unop
conversion(lhs)(ev).method()
  -> ev.method(lhs)

// unop0
conversion(lhs)(ev).method
  -> ev.method(lhs)

// unopWithEv
conversion(lhs).method(ev)
  -> ev.method(lhs)

// binop
conversion(lhs)(ev).method(rhs)
  -> ev.method(lhs, rhs)

// rbinop, for right-associative methods
conversion(rhs)(ev).method(lhs)
  -> ev.method(lhs, rhs)

// binopWithEv
conversion(lhs).method(rhs)(ev)
  -> ev.method(lhs, rhs)

// rbinopWithEv
conversion(rhs).method(lhs)(ev)
  -> ev.method(lhs, rsh)
```

Machinist also supports the following oddball cases (which may only be
useful for Spire):

```scala
// binopWithLift
conversion(lhs)(ev0).method(rhs: Bar)(ev1)
  -> ev.method(lhs, ev1.fromBar(rhs))

// binopWithSelfLift
conversion(lhs)(ev).method(rhs: Bar)
  -> ev.method(lhs, ev.fromBar(rhs))
```

In both cases, if "method" is a symbolic operator, it may be rewritten
to a new name if a match is found in `operatorNames`.

### Details & Fiddliness

To see the names Machinist provides for symbolic operators, see the
`DefaultOperatorNames` trait.

One caveat is that if you want to extend `machinist.Ops` yourself to
create your own name mapping, you must do so in a separate project or
sub-project from the one where you will be using the macros. Scala
macros must be defined in a separate compilation run from where they
are applied.

It's also possible that despite the wide variety of shapes provided by
`machinist.Ops` your shape is not supported. Machinist only provides
unary and binary operators, meaning that if your method takes 3+
parameters you will need to write your own macro.

It should be relatively easy to extend `Ops` to support these cases,
but that work hasn't been done yet. Pull requests will be gladly
accepted.

### Copyright and License

All code is available to you under the MIT license, available at
http://opensource.org/licenses/mit-license.php as well as in the
COPYING file.

### Code of Conduct

See the [Code of Conduct](CODE_OF_CONDUCT.md)

Copyright Erik Osheim and Tom Switzer 2011-2018.
