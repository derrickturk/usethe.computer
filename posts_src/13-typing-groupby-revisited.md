% typing group-by, revisited

In an aside to a <a href="/posts/9-why-are-data-scientists-switching-from-R-to-python.html">
previous post</a>, I mentioned that I'd rather live in the alternate universe
where data science tools were built on rich static type systems rather than 
evolved piecemeal from a variety of "dynamic" crud.
<a href="/posts/12-typing-groupby.html">Yesterday</a>,
we examined the semantics of SQL's `group by` operation, and pieced
together a minimalist version of a "data frame" in Python.
I lamented that since Python didn't provide a type system that let us check
any invariants of interest, we'd have to do all these checks at runtime.
Today, I'm going to put my money where my mouth is.
Let's peer into the alternate universe where data frames are well-typed, and
answer the question left unanswered yesterday: what *is* the type of `group by`?

This will take us "through the looking glass" to a world of purely functional
code, dependent types, type-driven programming, and "correct-by-construction"
data.
<!-- TODO: this is stupid and you know it -->
Let me show you how deep the rabbit hole goes.

### Eats, Shoots, and Leaves
*Data frames* are an oddity: a data structure ubiquitous in data science but
relative unknown to ordinary programmers.
I am not sure where they originated, but they go back at least to the early 90s
via the S statistical programming language.
I became familiar with them through S's descendent R, which makes heavy use of
them.
Python programmers in data science will likely be familiar with data frames
through the `pandas` library.

Either way, the concept is the same: a data frame is a helpful representation
for a tabular data set.
It can be represented in one of in one of two ways: as a heterogeneous
collection of homogeneous "columns", or as a homogeneous collection of
heterogeneous "rows".
The representations are *isomorphic*: we can losslessly go from one to the other
and back.

Data frames provide operations for accessing elements in various ways: by row,
by column (either by ordinal number or by name), and by *slicing* along either
of these dimensions.

Consider a typical table, perhaps from a relational database or a spreadsheet. 

patientID   age  condition   outcome
---------   ---  ---------   -------
        1    43  placebo     no change
        2    29  trial       improved
        3    35  trial       no change
        4    50  placebo     worsened

We have four columns and four rows.
The values within each column are all of the same type (that's what we mean by
homogeneous) but the types differ between columns (that's what we mean by
heterogeneous).

Most data frame implementations allow for access by column (and sometimes even
row) names rather than indices, but since that easy enough to add to an
ordinal-based implementation by using a lookup structure from names to indices,
we'll work only with ordinal indices for our proof of concept.

In `pandas`, we could create this data frame with the following:
```python
import pandas as pd

df = pd.DataFrame({
    'patientID': [1, 2, 3, 4],
    'age': [43, 29, 35, 50],
    'condition': ['placebo', 'trial', 'trial', 'placebo'],
    'outcome': ['no change', 'improved', 'no change', 'worsened'],
})
```

We can see from the way the data frame is constructed that `pandas` uses the
collection-of-columns representation.
It's a good choice, especially for performance reasons.
In the Python context, it lets us use `numpy` arrays as column representations
and take advantage of relatively fast vectorized operations on them.

The tradeoff is that traversing a data frame by rows becomes slower and more
difficult. In `pandas` we can convert a data frame into sequence-of-rows format
using `df.itertuples()`.

We can slice-and-dice a data frame in various ways:
```python
# one row
print(df.iloc[1])
```
> `patientID           2`  
> `age                29`  
> `condition       trial`  
> `outcome      improved`  
> `Name: 1, dtype: object`  

```python
# one column
print(df.iloc[:,1])
```
> `0    43`  
> `1    29`  
> `2    35`  
> `3    50`  
> `Name: age, dtype: int64`  

```python
# a row slice
print(df.iloc[1:3])
```
> `   patientID  age condition    outcome`  
> `1          2   29     trial   improved`  
> `2          3   35     trial  no change`  

```python
# a column slice
print(df.iloc[:,1:3])
```
> `   age condition`  
> `0   43   placebo`  
> `1   29     trial`  
> `2   35     trial`  
> `3   50   placebo`  

Both R and `pandas` also provide a suite of "SQL-like" operations on data
frames.
For instance, we might split a data frame into sub-frames based on unique
values of a key column---or perform a `group by` style aggregation.
`pandas` provides this with the `groupby` method:
```python
import numpy as np
df.groupby('condition').agg([len, np.mean])
```
> `          patientID      age`  
> `                len mean len  mean`  
> `condition`  
> `placebo           2  2.5   2  46.5`  
> `trial             2  2.5   2  32.0`  

That's... something, alright. The
<a href="https://pandas.pydata.org/pandas-docs/stable/user_guide/groupby.html">documentation</a>
for `groupby` is literally bewildering.
I'm sure there's a way to get something more sensible out, but it's opaque
enough to me that I'm not tempted to try.

This brings me to the reasons I am a <a href="https://github.com/derrickturk/antibiotics">notorious</a>
`pandas` <a href="https://usethe.computer/posts/12-typing-groupby.html">hater</a>.
I think `pandas` exposes way too large an API surface.
It tries to do too many things; I've seen programmers use it just to read a small
CSV file, or fetch a single column from a SQLite database.
I've also seen horrific "SQL in Python" ETL scripts which take hours to do, in
`pandas`, on the client side, what could be done in minutes on the database
server with SQL.

I don't want to complain about performance today, though.
Instead I want to complain about dynamic typing and its abuses.
The problem I find when trying to make sense of `pandas` documentation is that 
none of the functions are "typed" in a meaningful way.
They return objects of various runtime types depending upon the arguments
passed, the shape of the dataframe, and the phase of the moon.

I'm a huge fan of "type-driven development": contrary to popular belief,
most static typing aficionados don't think they're smarter than other
programmers.
To quote Dijkstra, we're "fully aware of the limited size of [our] own
skull[s]" and want the compiler to take care of as much tedious bookkeeping,
and as much error checking, as possible!
With a well-typed API, we can piece together software like a Lego set---a simple
type signature can sometimes be more useful than reams of "operational"
documentation like `pandas` provides.

Data frames are tricky beasts for most statically typed languages.
Your common-or-garden type systems don't excel at typing heterogeneous
collections, and less so at computing (say) the type of a resulting data frame
from a slicing operation.
We can do better, though, by taking a step outside the garden.
In a few hundred lines of code, we'll have a "prototype-quality"
data frame implementation which statically checks:

  * row counts
  * column types
  * indexing and slicing operations
  * conversion between column-oriented and row-oriented representations
  * split-by and group-by operations with arbitrary aggregation functions

### Into the Mirror Universe

Most readers of this blog will have learned to program in the "procedural" or
"imperative" fashion: programs are essentially laundry lists of instructions
which manipulate the state of various mutable cells containing values.
There are infinite variations on this theme (object-oriented programming,
structured programming, `On Error GoTo HELL`) but they
share a common, mechanistic philosophy of programming.
Programming, in the imperative mode, is about a machine: we understand that
the machine has a memory, and that it operates sequentially, loading
instructions and executing them.
A Turing machine, or something not unlike one, in short.

There are other ways to model computation, equivalent in "power" (this is the
crux of the <a href="https://en.wikipedia.org/wiki/Church%E2%80%93Turing_thesis">
Church-Turing thesis</a>.
One such model is drawn from Alonzo Church's work on the *lambda calculus*: a
formal system for computing only with functions, in the mathematical sense.

In imperative languages, we write "functions" but they often break the
assumptions we can make of well-behaved mathematical functions: they may
have side-effects like writing to a file, or altering mutable global state.
As a consequence, we can't indulge in equational reasoning about our programs.
In an imperative language, `x == x` does not imply `f(x) == f(x)`: perhaps `f`
uses and increments a global counter, say.

The discipline of functional programming elaborates on the lambda calculus.
We construct programs from *pure* "mathematical" functions.
By *pure*, we mean that our functions do not have side effects (yes, this raises
questions about how we ever "do" anything; we'll return to this thought), but
produce results which depend only and explicitly on their inputs.

As a consequence, we can easily apply equational reasoning to our programs:
if `x == x` then surely `f(x) == f(x)`.
This has some benefits for compiler code optimization, but the primary benefit
is to the programmer.
We reduce the number of things we have to keep in our heads.

Functional programming tends to operate on immutable structures (like
prepend-only linked lists) rather than mutable structures (like contiguous
mutable arrays).
This poses some challenges for performance, but we generally have ways of
working around them.
When absolutely needed, most functional languages provide an "escape hatch"
for executing imperative code---or better yet, for wrapping imperative code
behind a safe functional interface.

This is related to the answer to our previous question about interacting with
the real world.
One way or another, functional languages provide some loophole to "make things
happen": I/O and other side effects.
Haskell's (in)famous monads are an example of a principled solution to this
problem.

### "Well-Typed Programs Don't Go Wrong"

Functional languages have been hotbeds of type theory research and type system
development since Robin Milner's work on ML in the 1970s.
ML was, as the old saying goes, "an improvement on many of its successors": it
pioneered type inference for polymorphic functions, algebraic data types, and
the use of pattern matching as a concise notation for working with
inductively-defined data.

Milner famously coined the slogan "well-typed programs can't go wrong", which is
a bit misleading.
Of course programs of any sort can "go wrong"---but Milner means "go wrong" in
a specific technical sense.
In a *sound* type system, programs which typecheck can't exhibit the sort of
"undefined behavior" we find in languages like C: they are
<a href="https://en.wikipedia.org/wiki/Undefined_behavior">nasal-demon</a> free
even when they contain logic errors of the "do what I mean, not what I say"
variety.

On that note, how can we enhance our ability to "say what we mean" with types?
ML kicked off a miniature Cambrian explosion of functional languages and type
systems.
O'Caml (a somewhat "conservative" evolution of ML with some object-oriented
features) and Haskell (a more avant-garde "research language" which has begun
to see more serious industrial use) are the most commonly known modern
relatives.

However, a lesser-known branch of the family tree permits the use of *dependent
types*.
Dependent on what?
Henk Barendregt's <a href="https://en.wikipedia.org/wiki/Lambda_cube">"lambda cube"</a>
is my favorite way to answer this, but I'll provide a short answer.

Every language with a type system to speak of (even the ones that don't realize
it) has *types* and *terms*.
*Terms* are the pieces of code we construct programs from, and *types* classify
them.  We say that a term "has" a type (although that type may not be unique!):
the literal `3` is an integer.

It's common for terms to depend on other terms: we do this every time we write
a function!

Languages with polymorphic types (the imperative world tends to call these
"generics") make it possible for terms to depend on types
as well: for instance, we might be able to write a polymorphic "pair-with-self"
function `dup x = (x, x)` which is typed as something like
`forall t . t -> (t, t)`, where `t` is a "type variable" which stands in for
any argument type the function is called with.

These same languages also often provide polymorphic (or "generic") *type
constructors*, which let types depend on types.
For a possibly more familiar example, we might have
```c#
class List<T> {
    ...
    private T[] buffer;
}
```
Type constructors can be thought of as functions from types to types.
Some languages, like Haskell, permit the use of *higher-kinded types*
which allow polymorphism over type constructors themselves.

However, there's a combination we don't usually see: what if types could
depend on terms?
This would let us write types like "the type of arrays of exactly 5 integers",
where `5` is just an ordinary term and not some wacky type-level emulation.
This feature is exactly what we mean by *dependent types*.
In a sense, dependent type systems erase the type vs. term distinction.
We can freely intermix the two in our programs, even apply term-level functions
inside type definitions.

We're going to be using a dependently-typed functional programming language
called <a href="https://www.idris-lang.org/">Idris</a>. If you'd like to follow 
along, make sure you download and use the latest release of Idris 1 (not Idris
2, which is still an exciting work in progress).

Idris has a very similar syntax to Haskell, with a few syntactic refinements
and the addition of true dependent types.
I highly recommend it as a relatively accessible first step into the
dependently-typed world, especially if you have previous Haskell or even ML or
Lisp experience.
Edwin Brady's <a href="https://www.manning.com/books/type-driven-development-with-idris">
Type-Driven Development with Idris</a> is a great resource: not as a language
reference (that's online), but rather to see an expert use dependent types
"in anger" to solve surprising problems.

A quick primer on Idris syntax, especially for non-Haskellers:
```idris
-- this is a comment line

-- "x has type Integer" (a type signature)
x : Integer
-- define x
x = 3

-- function types are denoted by arrows
add3 : Integer -> Integer
-- this is a function definition; definitions are by pattern matching
--   and thus mirror use
add3 x = x + 3

-- function application is denoted by adjacency (not parentheses-they're just
--   for grouping), and is left-associative: f g x means (f g) x, not f (g x)
y : Integer
y = add3 x

-- multiple-argument functions are "curried": they really take one argument at
--   a time and returned the "partially applied" function
fusedMultAdd : Integer -> Integer -> Integer -> Integer
fusedMultAdd a b c = a + (b * c)

multAdd5 : Integer -> Integer -> Integer
multAdd5 = fusedMultAdd 5

-- polymorphic terms and types use type variables, which we can create
--   "implicitly" by mentioning them
-- tuple types are written (ty1, ty2, ...)
dup : a -> (a, a)
dup x = (x, x)

-- terms which denote types have type Type (this is what I meant
--   about erasing distinctions)
ty : Type
-- List is a type constructor of type Type -> Type
ty = List Integer

-- we can write algebraic data types with potentially dependent
--   constructor type signatures
data Either : Type -> Type -> Type where
  Left : a -> Either a b
  Right : b -> Either a b
```

### Apple Pie from Scratch

> If you wish to make an apple pie from scratch, you must first create
> the universe.  
> --Carl Sagan  

We're going to do this the hard way.
We'll take full advantage of Idris' type system, but make only minimal use of
the standard library: everything with an "interesting" type, we'll
(re-)implement ourselves.
We will make use of ordinary, non-dependent linked lists and a couple of
functions on them in places: these are defined the "usual" way and
re-implementation wouldn't be germane or interesting.

Many of the types we are about to implement appear in the Idris standard
library: `Nat`, `Fin`, `Vect`, and `HVect` are all available, and for the
most part should be implemented nearly identically.
This isn't because we're going to copy-and-paste code!
In fact, we're going to let the compiler write most of the code for us,
following the type signatures.
Given the types involved, there's often "one right way" to do it, which the
compiler can efficiently guide us toward.
This is a fun feature of Idris which really has to be experienced to be
believed; you can watch it in action to get a taste in Edwin Brady's
<a href="https://www.youtube.com/watch?v=gonVdPyVwQQ">conference talk</a>
on type-driven development.

Building these types from scratch will give us the opportunity to ease into
the shallow end of dependent typing before encountering the more complex
types and definitions which will be involved in defining data frames.

### Peano Recital

The few of you who have seen dependently-typed programs before probably can
guess what's coming next.
We're going to need to talk about structures of known length, and reason about
them inductively.
In order to do this, we're going to need to construct the natural numbers
(that is, the non-negative integers) first!

Following the <a href="https://en.wikipedia.org/wiki/Peano_axioms">Peano</a>
style, we'll define naturals inductively: a natural number is either zero (`Z`),
or the successor (`S n`) of ("one more than") a natural number.

Our program proper begins here---all subsequent code snippets, except REPL
interactions (you'll see a `*tafra>` prompt), are part of the same Idris module.
```idris
%hide Nat
-- we'll roll our own, this is Carl Sagan's apple pie

-- natural numbers a la Peano
data Nat : Type where
  Z : Nat
  S : Nat -> Nat

-- addition, by recursion on the left
plus : Nat -> Nat -> Nat
plus Z m = m
plus (S n) m = S (plus n m)
```

This is a verbose representation: for instance, 3 is represented as `S (S (S Z))`.
That's annoying from a human perspective when dealing with larger numbers, but
there are workarounds.
We'll use raw Peano representations, though, for clarity and consistency.
It's also problematic from a performance standpoint: 3 is represented by a
linked list of 3 elements!
One can see that this won't necessarily scale.
Performance is a theme we'll return to; for now we'll remark that in many
cases we'll be able to "erase" these numbers so that they don't need to exist
at runtime, but this is an imperfect art and performance is a valid concern.

We can now talk about lengths or counts (*cardinal* numbers), but what about
indices (*ordinal* numbers)?
Let's introduce the notion of "finite sets" of natural numbers less than a
given number.
For a given `n : Nat`, values of type `Fin n` will exactly correspond to valid
indices into a collection of length `n` (assuming zero-based indexing, as all
decent folk should---and you thought that was arbitrary convention).

```idris
-- "finite sets" - naturals less than a given number
data Fin : Nat -> Type where
  FZ : Fin (S n) -- zero is less than any successor
  FS : Fin n -> Fin (S n) -- by induction
```

We will build these indices the same way as our naturals, and the same caveats
apply: to request the third element of a collection we'll need `FS (FS FZ)`.

Now we need something to index into!
We'll start by building homogeneous vectors: these are exactly the
representation we'll use for each column of our data frames.
In dependent-typing parlance, vector types are parameterized by their
length (a `Nat`) and their element type (a `Type`).

We can use pattern matching to provide checked inductive definitions for
access, concatenation, and mapping---tracking the result types with "type-level
arithmetic".
Of course, in Idris, our type-level arithmetic is just arithmetic!
```idris
-- "vectors" - known-length homogeneous sequences
namespace Vect
  data Vect : Nat -> Type -> Type where
    Nil : Vect Z a
    (::) : a -> Vect n a -> Vect (S n) a

  -- bounds-checked element access
  index : Fin n -> Vect n a -> a
  index FZ (x :: _) = x
  index (FS n) (_ :: xs) = index n xs

  -- length-tracking concatenation
  (++) : Vect n a -> Vect m a -> Vect (plus n m) a
  [] ++ ys = ys
  (x :: xs) ++ ys = x :: xs ++ ys

  -- we can map over vectors, preserving type
  Functor (Vect n) where
    map f [] = []
    map f (x :: xs) = f x :: map f xs

  -- we'll want this later to implement "slicing"
  vslice : Vect k (Fin n) -> Vect n a -> Vect k a
  vslice [] _ = []
  vslice (i :: is) xs = index i xs :: vslice is xs

  -- we'll want this to walk two parallel vectors in "lock-step"
  zip : Vect n a -> Vect n b -> Vect n (a, b)
  zip [] [] = []
  zip (x::xs) (y::ys) = (x, y)::zip xs ys
```

We're now equipped to model known-length homogeneous collections.
Idris has a nice syntax sugar: it will translate `[x, y, z]` into
`x :: y :: z :: []`, for any type with constructors `::` ("cons") and `Nil`.

This lets us write linked lists, vectors, and even the more interesting types
we'll be constructing shortly using the same convenient notation.

Let's define an example `Vect` of 3 strings, and another of 4 strings.
```idris
exVect1 : Vect (S (S (S Z))) String
exVect1 = ["first", "second", "third"]

exVect2 : Vect (S (S (S (S Z)))) String
exVect2 = ["uno", "dos", "tres", "catorce"]
```

We can try out our functions at the Idris REPL.
For example, we can index a `Vect` with an appropriate `Fin`, but not an
invalid one!
```idris
*tafra> index (FS (FS (FS FZ))) exVect2
"catorce" : String

*tafra> index (FS (FS (FS FZ))) exVect1
(input):1:1-36:When checking an application of function Main.Vect.index:
        Type mismatch between
                Vect (S (S (S Z))) String (Type of exVect1)
        and
                Vect (S (S (S (S n)))) iType (Expected type)

        Specifically:
                Type mismatch between
                        Z
                and
                        S n
```

That error is saying we "ran out of valid indices" (in other words, hit `Z`)
before reaching our requested index.
Note that it's not happening "at runtime"---this is the compiler carrying out
static type checking.

`vslice` is a "vector slice" operation which we'll want later.
It fetches multiple elements from a `Vect`, using a collection of indices
which is itself a `Vect` of `Fin n`, where `n` is the length of the `Vect`
we're indexing in to.
This gives us safe, checked slicing:
```idris
*tafra> vslice [FZ, FZ, (FS (FS (FS FZ)))] exVect2
["uno", "uno", "catorce"] : Vect (S (S (S Z))) String

*tafra> vslice [FZ, FZ, (FS (FS (FS FZ)))] exVect1
(input):1:1-42:When checking an application of function Main.Vect.vslice:
        Type mismatch between
                Vect (S (S (S Z))) String (Type of exVect1)
        and
                Vect (S (S (S (S n)))) a (Expected type)

        Specifically:
                Type mismatch between
                        Z
                and
                        S n
```

`map`, `++` (concatenation), and `zip` work just like their equivalents
for lists, but with length checking.
```idris
*tafra> zip exVect1 exVect1
[("first", "first"), ("second", "second"), ("third", "third")]
  : Vect (S (S (S Z))) (String, String)

*tafra> exVect1 ++ exVect2
["first", "second", "third", "uno", "dos", "tres", "catorce"]
  : Vect (S (S (S (S (S (S (S Z))))))) String

-- OK, a little cheat here: the result of length is a built-in Nat, not our Nat
*tafra> map length exVect1
[5, 6, 5] : Vect (S (S (S Z))) Nat
```

We won't go into as much detail regarding all the functions which operate on
our next types, but hopefully that provides a taste of interactive development
with Idris and some intuition for how our types are working.

### Modeling Heterogeneity
Here's where things get interesting, and where we start to move beyond what can
be done straightforwardly with anything short of full dependent types.
Everything previous to this point can be written, perhaps tediously,
in not-quite-fully-dependent systems: we can construct things like `Nat` at the
type level; or, better, we can take advantage of "lightweight dependent types"
features like <a href="/posts/2-potd-gadt-oop.html">GADTs</a>.

In large part that's because the types resulting from, say, indexing a `Vect`
are easily determined.
Since a `Vect` is homogeneous, the only interesting check is for index validity;
the type of any element is fixed.

To represent "rows" of our data frames, we're going to need a heterogeneous
structure: something like a tuple.
We'll parameterize these `HVect` (heterogeneous vector) types by a
`Vect n Type`---in other words, by the vector of their element types.
```idris
-- heterogeneous vectors - we'll need these to represent "rows"
namespace HVect
  data HVect : Vect n Type -> Type where
    Nil : HVect []
    (::) : a -> HVect as -> HVect (a::as)

  -- bounds-checked element access, with appropriate return type
  index : {n : Nat} ->
          {as : Vect n Type} ->
          (i : Fin n) ->
          HVect as ->
          index i as
  index FZ [] impossible
  index (FS _) [] impossible
  index FZ (x :: _) = x
  index (FS i) (_ :: xs) = index i xs

  -- HVects can be compared for equality, when their components
  --   are comparable - we'll need this for "group by"

  Eq (HVect []) where
    [] == [] = True

  (Eq a, Eq (HVect as)) => Eq (HVect (a::as)) where
    (x :: xs) == (y :: ys) = x == y && xs == ys

  -- length- and type-tracking concatenation
  (++) : HVect as -> HVect bs -> HVect (as ++ bs)
  [] ++ ys = ys
  (x :: xs) ++ ys = x :: xs ++ ys

  -- first element only
  head : HVect (a::as) -> a
  head (x :: _) = x

  -- all but first element
  tail : HVect (a::as) -> HVect as
  tail (_ :: xs) = xs
```

Let's create an example `HVect` and interact with it at the REPL.
Remember, because our data constructors are called `::` and `Nil`, we can use
`[x, y, z]` notation.

```idris
exHVect : HVect [Integer, String, Nat, String]
exHVect = [2, "fast", (S (S Z)), "furious"]
```

Now we'll see the resulting values and types from manipulating it:
```idris
*tafra> index (FS FZ) exHVect
"fast" : String

*tafra> exHVect ++ exHVect
[2, "fast", S (S Z), "furious", 2, "fast", S (S Z), "furious"]
  : HVect [Integer,
           String,
           Nat,
           String,
           Integer,
           String,
           Nat,
           String]

*tafra> head exHVect
2 : Integer

*tafra> tail exHVect
["fast", S (S Z), "furious"] : HVect [String, Nat, String]
```

We now have everything we need to build our minimalist data frames (as before,
let's call this type a "tafra" (plural "tafrae"), because it's the "innards"
of a proper data frame).
```idris
namespace Tafra
-- a "tafra" is a list of same-length vectors of various types
-- the type is indexed by the row count and a vector of column types
  data Tafra : Nat -> Vect n Type -> Type where
    Nil : Tafra n []
    (::) : Vect n a -> Tafra n as -> Tafra n (a::as)

  -- we can concatenate tafras with the same column types and different
  --   row counts "vertically"
  (++) : Tafra n as -> Tafra m as -> Tafra (plus n m) as
  [] ++ [] = []
  (topFirst :: topRest) ++ (botFirst :: botRest) =
    Vect.(++) topFirst botFirst :: topRest ++ botRest

  -- we can concatenate tafras with different column types but the same row
  --   counts "horizontally"
  hcat : Tafra n as -> Tafra n bs -> Tafra n (as ++ bs)
  hcat [] y = y
  hcat (x :: xs) y = x :: hcat xs y

  -- indexing by columns is simple, and returns a column vector
  index : {n: Nat} ->
          {as : Vect n Type} ->
          (i : Fin n) ->
          Tafra m as ->
          Vect m (index i as)
  index FZ [] impossible
  index (FS _) [] impossible
  index FZ (x :: _) = x
  index (FS i) (_ :: xs) = index i xs

  -- indexing by rows is trickier - we have to return a heterogeneous "tuple"
  rindex : Fin n -> Tafra n as -> HVect as
  rindex _ [] = []
  rindex n (x :: xs) = index n x :: rindex n xs

  -- we can also "slice" by multiple row or column indices
  -- we'll do this a little inefficiently

  -- "column" slices
  slice : {n : Nat} ->
          {as : Vect n Type} ->
          (cols : Vect k (Fin n)) ->
          Tafra m as ->
          Tafra m (vslice cols as)
  slice [] _ = []
  slice (i :: is) xs = index i xs :: slice is xs

  -- "row" slices
  rslice : {n : Nat} ->
           (rows : Vect k (Fin n)) ->
           Tafra n as ->
           Tafra k as
  rslice _ [] = []
  rslice [] (_ :: xs) = [] :: rslice [] xs
  rslice is (x :: xs) = vslice is x :: rslice is xs

  -- just the first row
  rhead : Tafra (S n) as -> HVect as
  rhead [] = []
  rhead ((x :: _) :: ys) = x :: rhead ys

  -- all rows but the first
  rtail : Tafra (S n) as -> Tafra n as
  rtail [] = []
  rtail ((_ :: xs) :: ys) = xs :: rtail ys

  -- combine the "head" and the "tail" (we'll need this to
  --   correctly type "group by"
  rcons : HVect as -> Tafra n as -> Tafra (S n) as
  rcons [] [] = []
  rcons (x :: xs) (r :: rs) = (x :: r) :: rcons xs rs

  -- transpose to a row-oriented representation
  transpose : {n : Nat} -> Tafra n as -> Vect n (HVect as)
  transpose {n = Z} _ = []
  transpose {n = (S m)} xs = rhead xs :: transpose (rtail xs)

  -- rebuild a tafra from a transposed representation
  untranspose : {n : Nat} ->
                {k : Nat} ->
                {as : Vect k Type} ->
                Vect n (HVect as) ->
                Tafra n as
  untranspose {n = Z} {as = []} [] = []
  untranspose {n = Z} {as = (t :: ts)} [] = [] :: untranspose []
  untranspose {n = (S m)} {as = []} _ = []
  untranspose {n = (S m)} {as = (t :: ts)} (xs :: yss) =
    (head xs :: map head yss) :: untranspose (tail xs :: map tail yss)

  -- a zero-row tafra of any type
  empty : Tafra Z as
  empty {as = []} = []
  empty {as = (_ :: _)} = [] :: empty
```

There's some new syntax here.
Recall that previously we mentioned that type variables can be created
implicitly.
In general, Idris will create implicit arguments whenever a previously
unmentioned lowercase name appears in a type signature.
The `{n : Nat}` notation makes these "implicts" explicit, so that we can
name and type them ourselves, as well as pattern-match on them.
We need this in a few spots to correctly type `Tafra` functions.

Let's create a couple of example tafrae, and interact with them:
```idris
someData : Tafra (S (S (S Z))) [Double, Bool, List String]
someData =
  [ [23.4, 45.6, 78.9]
  , [True, False, True]
  , [ ["yo", "dawg"]
    , ["i heard you", "like structures"]
    , ["so i put structures in your structures"]
    ]
  ]

otherData : Tafra (S (S Z)) [Int, String]
otherData =
  [ [99, 45]
  , ["some", "text"]
  ]
```

Indexing is possible, and checked, by column or row index.
A column is a `Vect` and a row is an `HVect`.
```idris
*tafra> index FZ someData
[23.4, 45.6, 78.9] : Vect (S (S (S Z))) Double

*tafra> rindex FZ someData
[23.4, True, ["yo", "dawg"]] : HVect [Double, Bool, List String]

*tafra> rindex (FS (FS FZ)) otherData
(input):1:1-29:When checking an application of function Main.Tafra.rindex:
        Type mismatch between
                Tafra (S (S Z)) [Int, String] (Type of otherData)
        and
                Tafra (S (S (S n))) as (Expected type)

        Specifically:
                Type mismatch between
                        Z
                and
                        S n
```

Slicing can be done by rows or by columns, producing sub-tafrae:
```idris
*tafra> slice [FZ, (FS FZ)] someData
[[23.4, 45.6, 78.9], [True, False, True]] : Tafra (S (S (S Z))) [Double, Bool]

*tafra> rslice [FZ, (FS (FS FZ))] someData
[[23.4, 78.9],
 [True, True],
 [["yo", "dawg"], ["so i put structures in your structures"]]]
   : Tafra (S (S Z)) [Double, Bool, List String]
```

We have `rhead` (first row) and `rtail` (all-but-first-row) functions (this is
the "functional programming" meaning for these names, not the "pandas" meaning),
as well as `rcons` which inverts the `rhead`/`rtail` transformation.
```idris
*tafra> rhead otherData
[99, "some"] : HVect [Int, String]

*tafra> rtail otherData
[[45], ["text"]] : Tafra (S Z) [Int, String]

*tafra> rcons (rhead otherData) (rtail otherData)
[[99, 45], ["some", "text"]] : Tafra (S (S Z)) [Int, String]
```

We can convert to and from row-major (`Vect` of `HVect`) representation
with `transpose` and `untranspose`, which are probably not ideally named
(`it` is a handy REPL-only variable referring to the value of the last evaluated
expression, like `_` in the Python REPL):
```idris
*tafra> transpose otherData
[[99, "some"], [45, "text"]] : Vect (S (S Z)) (HVect [Int, String])

*tafra> untranspose it
[[99, 45], ["some", "text"]] : Tafra (S (S Z)) [Int, String]
```

Finally, just like `pandas`, we can concatenate tafrae either "vertically"
(appending rows) or "horizontally" (appending columns), but ours are checked!
For vertical concatenation we need identical column types; for horizontally
concatenation we need identical row counts.
```idris
*tafra> someData ++ someData
[[23.4, 45.6, 78.9, 23.4, 45.6, 78.9],
 [True, False, True, True, False, True],
 [["yo", "dawg"],
  ["i heard you", "like structures"],
  ["so i put structures in your structures"],
  ["yo", "dawg"],
  ["i heard you", "like structures"],
  ["so i put structures in your structures"]]] : Tafra (S (S (S (S (S (S Z))))))
                                                       [Double, Bool, List String]

*tafra> hcat otherData otherData
[[99, 45], ["some", "text"], [99, 45], ["some", "text"]]
  : Tafra (S (S Z)) [Int, String, Int, String]

*tafra> someData ++ otherData
(input):1:1-29:When checking an application of function Main.Tafra.++:
        Type mismatch between
                Tafra (S (S Z)) [Int, String] (Type of otherData)
        and
                Tafra m [Double, Bool, List String] (Expected type)

        Specifically:
                Type mismatch between
                        Z
                and
                        S Z

*tafra> hcat someData otherData
(input):1:1-23:When checking an application of function Main.Tafra.hcat:
        Type mismatch between
                Tafra (S (S Z)) [Int, String] (Type of otherData)
        and
                Tafra (S (S (S Z))) bs (Expected type)

        Specifically:
                Type mismatch between
                        Z
                and
                        S Z
```

We've already achieved much of what we set out to do.
We now have a statically checked data frame implementation which correctly
models a heterogeneous collection of same-length homogeneous columns.

### Typing Group-by
As a final goal, let's answer the question posed previously: what is the correct
(or at least *a* correct) type for a SQL-inspired `group by` operation?

Just like we did in Python, we'll execute our "group by" by accumulating rows
sharing a unique tuples of "group-by values" into an associative collection.
We don't have easy access to a hash table implementation, so we'll use a list of
`(key, value)` tuples as a purely functional implementation of associative
arrays.

On the way to "group by", we'll define helper functions to collect tuples by
unique value; `rowsByUnique` could be useful on its own but also forms a solid
building block for `groupBy`.

```idris
-- instead of maps, we'll use "association lists" for associative lookup
-- the prelude helpfully provides
-- lookup : Eq a => a -> List (a, b) -> Maybe b
-- we'll define a helpful type synonym

Map : Type -> Type -> Type
Map k v = List (k, v)

-- and some additional helpers
insert : Eq k => k -> v -> Map k v -> Map k v
insert k v [] = [(k, v)]
insert k v (x@(k', _) :: xs) =
  if k == k'
     then (k, v) :: xs
     else x :: insert k v xs

-- we're almost ready to write "group by"
-- first, let's prove we can collect rows by unique combination of group-by
--   column values

-- an "accumulating" function we'll use to collect rows by unique combination
private
accumRows : Eq k =>
            Map k (List v) ->
            Vect n (k, v) ->
            Map k (List v)
accumRows map [] = map
accumRows map ((b, g) :: rest) = case lookup b map of
  Nothing => accumRows (insert b [g] map) rest
  -- for performance, we're going to build these "backward"
  (Just xs) => accumRows (insert b (g::xs) map) rest

-- this is also the only example I have ever seen of explicitly passing
--   a "locally named" interface implementation
rowsByUnique : (is : Vect groups (Fin cols)) -> -- indices of group-by columns
               (js : Vect aggs (Fin cols)) -> -- indices of aggregated-over columns
               {auto prf: Eq (HVect (vslice is as))} ->
               Tafra rows as ->
               Map (HVect (vslice is as)) (List (HVect (vslice js as)))
rowsByUnique {prf} is js xs =
  accumRows @{prf} [] (zip (transpose (slice is xs)) (transpose (slice js xs)))

-- ok, let's type group by
groupBy : {cols : Nat} ->
          {rows : Nat} ->
          {as : Vect cols Type} ->
          {groups : Nat} ->
          {aggs : Nat} ->
          -- indices of group-by columns
          (is : Vect groups (Fin cols)) ->
          -- indices of aggregated-over columns
          (js : Vect aggs (Fin cols)) ->
          -- aggregation function
          (aggFn : List (HVect (vslice js as)) -> HVect bs) ->
          -- a proof that group-by columns can be compared for equality
          --   (automatically found by compiler)
          {auto prf: Eq (HVect (vslice is as))} ->
          -- data frame to group-by
          Tafra rows as ->
          -- result type: an existentially quantified tafra of some
          --   row count, with columns from the group-by columns and
          --   the result of the aggregation function
          (k : Nat ** Tafra k (vslice is as ++ bs))
groupBy is js fn xs = applyAggregation fn (rowsByUnique is js xs) where
  applyAggregation : (List (HVect gs) -> HVect rs) ->
                     Map (HVect as) (List (HVect gs)) ->
                     (k : Nat ** Tafra k (as ++ rs))
  applyAggregation f [] = (Z ** empty)
  applyAggregation f ((k, rs) :: rest) with (applyAggregation f rest)
    applyAggregation f ((k, rs) :: _) | (n ** restAgg) =
      (S n ** rcons (k ++ f rs) restAgg)
```

To be fair, that was nearly as exhausting to write as it probably was to read.
I take this as a sign that "group by" is a somewhat complicated operation,
semantically.

There's plenty of bookkeeping of indices and types (and of the constraint
that we must be able to do equality comparison on our group-by column values),
but there is one interesting new complication which requires us to use another
"unusual" feature of our type system.
Unlike the operations we implemented previously, there's not a fixed
relationship between the number of rows in a tafra and the number of rows
resulting from `groupBy`.

The result's row count is only known at runtime, after we consolidate unique
combinations of the grouping columns.
To handle this in our type system, we use an *existential type* in the form of a
*dependent pair*.
If polymorphic types correspond to "for all" (universal quantification),
existential types correspond to "there exists" (existential quantification).
Object-oriented programmers work with existential types every day in the form
of interfaces: they promise that "there exists" a value some concrete type
fulfilling the interface without statically committing to a given
implementation.

We use an existential to promise that "there exists" some count of rows in the
result, and that the result has that many rows, and that we can produce both
bundled together in a dependent pair (the
`(k : Nat ** Tafra k (vslice is as ++ bs))` type of our result is a dependent
pair type).

It's also noteworthy that types can contain function application!
We use this to indicate the correspondence between the columns selected for
grouping and aggregation, the aggregation function result type, and the column
types of the resulting tafra.

Let's see `groupBy` in action. We'll write the rough equivalent of this SQL:
```sql
select
  col2,
  join('\n', str(col1) & join(' ', col3))
from someData
group by col2
```

In our system:
```idris
*tafra> groupBy
  [FS FZ]
  [FZ, FS (FS FZ)]
  ((::[]) . unlines . map (\[num, strs] => unwords (show num :: strs)))
  someData

(S (S Z) **
 [[True, False],
  ["78.9 so i put structures in your structures\n23.4 yo dawg\n",
   "45.6 i heard you like structures\n"]]) : (k : Nat ** Tafra k [Bool, String])
```

There we have it! We've typed `groupBy`, and it works.

### Parting Thoughts
That was a lot of work, especially to build the more complex types involved in
dynamic operations like "group by".
However, we've gained incredible power to statically check the validity of our
data manipulation.

Why is this from "another universe"?
Why don't we see dependent types used for these kinds of systems in practice?
I think there are a few reasons: some good, some not.

Functional programming is still unpopular, although less so than it used to be.
The same goes for expressive static type systems.
We're seeing a growing trend of "mainstream" imperative languages adopting
features (typeclasses or "traits", algebraic types and pattern matching, etc.)
from this world, but languages like Haskell are still pretty far from
mainstream.

Data science had the poor luck to come of age during the long slow decline of
"dynamic typing", but before the "static typing renaissance" we're currently
living through.
As a consequence, key tools like R or `pandas` are very much products of
their times.
At the time, given the mainstream or near-mainstream alternatives, the decision
wasn't a bad one.
A REPL is very helpful for this sort of work, and they were hard to come by for
statically-typed languages.

A deeper issue is performance.
If you were playing close attention, you realized that every data structure
we built, from `Nat` to `Vect` to `Tafra`, was an immutable linked list
"under the hood".
That's far from ideal for performance under the best of circumstances, and
the kind of code we wrote to slice and manipulate frames was far from the
"best-case" scenario for these data structures.

A few years back I worked on a dependent-typing related project for a client.
I spent a lot of time researching prior work on optimizing compilers for
these languages, to improve the performance of dependently-typed programs.
Even with aggressive "erasure" (e.g. so all those `Nat` indices don't have to
exist at runtime), performance is at best comparable to a "traditional"
functional language---which is to say that performance is poor.

There are some reasons for optimism, though.
For example, <a href="https://github.com/ollef/sixten">Sixten</a> is an
experimental functional language with both "unboxed" (read: fewer
indirections---the difference between a C array and a Python list, or for that
matter a Haskell list) data and dependent function types.
There doesn't appear to be support for the kind of dependent pair types we
used in typing `groupBy`, but it's a promising start!

Comparing Idris, say, to Python, Idris probably comes out ahead in performance
when considering just the "core language".
However, Python's data science ecosystem is mostly built on Python wrappers
around fast compiled code (C, C++, or Fortran).
There's no reason we couldn't build the same sort of "layered" system to provide
a dependently-typed interface to a fast native data frame library.
True, we'd lose some of the "correct by construction" benefits we got while
writing our implementation, but with adequate testing we'd still provide the
benefits of static checking to our users: that is, assuming we could get them
to try and adopt a dependently-typed language.

Alas, that day is probably quite far off.
Or perhaps dependent types are just a stepping stone to, or a detour from,
some future approach which manages to combine good performance with static
checking and the *je ne sais quoi* which attracts "average" programmers.

For now, we'll just have to live with occasional glimpses through the
looking glass into a world that might have been.
