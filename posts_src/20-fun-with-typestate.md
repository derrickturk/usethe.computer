% Fun With Simulated Typestate in Python 3.8

> "There's an old saying in Tennessee—I know it's in Texas, probably in Tennessee—that says, 'Fool me once, shame on...shame on you. Fool me—you can't get fooled again.'"  
> --George W. Bush  

It's true, you can't get fooled again! Not any more than you can open an already-open door. But does your type system know that?

Today, in honor of the recent release of Python 3.8[^1], we'll introduce a fun type-level programming trick well-known already in other language communities, which will let us automatically check these and other invariants.

Technically, what we're trying to do is approximate a discipline of <a href="https://en.wikipedia.org/wiki/Typestate_analysis">*typestate*</a> in our available type system.
There are several popular tricks for doing this; we'll cover two of them in this article.
The first is most commonly found in languages with at least some semblance of *dependent typing*---that is, types which are parameterized by values (for example, integers, as in C++'s `std::array<double, 5>`).
With the new <a href="https://docs.python.org/3/library/typing.html#typing.Literal">`typing.Literal`</a> class introduced in Python 3.8, Python takes a big step in this direction.
The second is awkward and verbose, but available in most languages with subtyping and generic types (for example, almost all "object-oriented" languages).

Typestate is an augmentation to a type system which lets us model objects which have defined states, with certain operations only available in given states, and where operations may alter the state of the object in a defined way.
"True" typestate support is quite uncommon in programming languages, but we can use the techniques we'll illustrate shortly to simulate typestate by turning each state into a distinct type, avoiding duplication by making use of polymorphic ("generic") types.

## After The Horse Has Left The Barn
Let's start with a simple system, like a door (if you think this is silly, feel free to imagine a filesystem, or a database connection, with similar states and constraints).
Doors have the following rules:

* A door can be open or closed.
* If closed, it can be locked or unlocked (we'll take closed-but-unlocked to be the "default" state of a door).
* We can only open a closed, unlocked door.
* We can only close an open door.
* We can only lock a closed, unlocked door.

We'd like to write a `Door` class which provides open, close, lock, and unlock operations honoring these invariants.
The approach we'll take is this: first, define a type whose values are the possible states of the door, then create a `Door` class parametric over `Literal`s of that type.

The `Literal` type is parameterized by the values objects of the type are permitted to contain. It's probably easiest to understand by example: `3` is a valid value of type `Literal[3]`, but `4` is not.
`Literal` can be given multiple parameters, indicating objects which may be take any of several literal values.
The original intent of the type was to capture the typing of functions like Python's `open` (simplified, something like  `def open(path: str, mode: Literal['r', 'rb', 'w', 'wb']) ...`).
However, in conjunction with `Enum` types, it'll be quite useful as something like "singleton types" in Haskell: a way to lift values denoting possible "typestates" to the type level.

Let's begin with some imports, and a definition of our possible door states:
```python
from enum import Enum, auto
from typing import cast, Generic, Literal, TypeVar

class DoorState(Enum):
    CLOSED = auto()
    LOCKED = auto()
    OPEN = auto()
```

That covers the possible states as previously discussed (`CLOSED` denotes a closed, unlocked door).
We'll need to write a generic `Door` class now; in Python's type system this requires a `TypeVar` and the use of `Generic` as a superclass:
```python
_S = TypeVar('_S', bound = DoorState)

class Door(Generic[_S]):
    # these methods are "privileged"; they can cast because the invariants
    #   are guaranteed as long as calling code typechecks
    
    def open(self: 'Door[Literal[DoorState.CLOSED]]'
            ) -> 'Door[Literal[DoorState.OPEN]]':
        return cast(Door[Literal[DoorState.OPEN]], self)

    def close(self: 'Door[Literal[DoorState.OPEN]]'
            ) -> 'Door[Literal[DoorState.CLOSED]]':
        return cast(Door[Literal[DoorState.CLOSED]], self)

    def lock(self: 'Door[Literal[DoorState.CLOSED]]'
            ) -> 'Door[Literal[DoorState.LOCKED]]':
        return cast(Door[Literal[DoorState.LOCKED]], self)

    def unlock(self: 'Door[Literal[DoorState.LOCKED]]'
            ) -> 'Door[Literal[DoorState.CLOSED]]':
        return cast(Door[Literal[DoorState.CLOSED]], self)

def new_door() -> Door[Literal[DoorState.CLOSED]]:
    return Door()
```

The methods on `Door` each have a different type annotation on `self` and a different return type, using the appropriate `Literal` types lifted from values of `DoorState` to implement the rules we laid out earlier. Unfortunately, we still have to do the stupid type-as-string dance because Python still insists on eagerly evaluating type annotations.

The `Door` constructor (the default `__init__` method) is generic over door states: we could create a door in a given state by annotating a variable; for instance:
```python
open_door: Door[Literal[DoorState.OPEN]] = Door()
```
However, we've provided the `new_door` function to concisely initialize a door in the closed, unlocked state. 

"Under the hood", all door states are represented the same way, so it's safe to use `cast`.
Python is all about convention over rigor, so we don't have a way to prevent programmers from misusing our class by `cast`ing in their calling code, but we should document that it's used internally and safe due to the use of simulated typestate to ensure our invariants hold.

Let's see it in action:
```python
if __name__ == '__main__':
    # a valid sequence
    door = new_door() # a closed door
    door = door.open() # open it
    door = door.close() # close it again
    door = door.lock() # lock it
    door = door.unlock() # unlock it
    door = door.open() # open it again

    # various forbidden operations
    # door = door.lock() # can't lock an open door
    # door = door.unlock() # or unlock it
    door = door.close()
    # door = door.close() # or close a closed door
```

Try uncommenting the "forbidden" lines and see the type errors produced by `mypy`!
They're not pretty, but they convey the problem, more-or-less:
```
typestate.py:47: error: Invalid self argument "Door[Literal[DoorState.OPEN]]" to attribute function "lock" with type "Callable[[Door[Literal[DoorState.CLOSED]]], Door[Literal[DoorState.LOCKED]]]"
```
In other words, we can't lock an open door.

Note that you'll need to invoke `mypy` with the `--allow-redefinition` argument in order for the code to typecheck; that's so that we can redefine `door` to take new types on each line.
This practice of "variable shadowing", uncontroversial in other languages, is essential for writing typestate-heavy code in a concise way.

## Fool Me Twice, Shame On My Type System

We can now implement the rules of "fooling", as elaborated by President Bush, in a very similar fashion:
```python
from enum import Enum, auto
from typing import cast, Generic, Literal, TypeVar

class MeState(Enum):
    UNFOOLED = auto()
    FOOLED = auto()

_S = TypeVar('_S', bound = MeState)

class Me(Generic[_S]):
    def __init__(self, name: str):
        self.name = name

    def fool(self: 'Me[Literal[MeState.UNFOOLED]]'
            ) -> 'Me[Literal[MeState.FOOLED]]':
        print(f'shame on {self.name}')
        return cast(Me[Literal[MeState.FOOLED]], self)

if __name__ == '__main__':
    me: Me[Literal[MeState.UNFOOLED]] = Me('George')
    me = me.fool()
    # me = me.fool() # you don't get fooled again!
```
Once again, you'll need to run `mypy --allow-redefinition` to typecheck this code.

## Into the Chasm of Madness

What if the rules of fooling had been a hair different?

> "Fool me once, shame on me. Fool me twice, shame on me. Fool me three times, shame on me. Fool me four times, shame on me. Fool me five times, shame on me. Fool me more than five times---you don't get fooled again!"  
> --Georg M. Shrub, from an adjacent parallel universe

This is a much harder problem.
To quote the <a href="https://www.python.org/dev/peps/pep-0586/#true-dependent-types-integer-generics">PEP introducing `Literal`</a>, "[a]t the very least, it would be useful to add some form of integer generics"---but that's not possible with the current version of `Literal`.
Luckily, we don't really need integer generics (or true dependent types, which subsume them) for this challenge: we'll get by with an encoding of natural numbers as types.

This is the older, nastier, but more reliable trick for "typestate" in object-oriented languages with generic types (say, Python): we'll create a class hierarchy describing the possible states (these classes will generally not be used at runtime at all; they're purely "compile-time" artifacts), then make our type-with-typestate generic over these state types (usually using a "superclass bound").

Oh, right: we'll also need a type-level encoding of proofs that one given natural number is less than or equal to another!
We're going to lift the necessary ideas right out of the dependently-typed world, and find that they work just fine (if a bit awkwardly) in Python's type system.

Natural numbers are easy, following the typical Peano-style unary encoding, with a zero and a successor function. The only odd thing is that these are types, not values!
```python
from enum import Enum, auto
from typing import cast, Generic, Literal, TypeVar

# natural numbers, as types

class Nat:
    pass

class Z(Nat):
    pass

_N = TypeVar('_N', bound = Nat)
class S(Nat, Generic[_N]):
    pass
```

I don't recall the traditional names of the less-than-or-equal proof constructors for the trivial and inductive cases, so we'll call them `duh` and `well` for the humorous effect:
```python
# proofs of less-or-equal between type-level nats, as types

_M = TypeVar('_M', bound = Nat)
class _LTE(Generic[_N, _M]):
    pass

# lock LTE construction behind functions which ensure well-typedness

def duh() -> _LTE[Z, _N]:
    return _LTE()

def well(prf: _LTE[_N, _M]) -> _LTE[S[_N], S[_M]]:
    return _LTE()
```

Of course, a user could be very naughty and directly construct an (invalid) `_LTE` proof object, but we've used the underscore prefix to indicate our disapproval of such intrigues.
The idea is that `duh()` constructs a proof that zero is less than or equal to any natural number (the trivial case), and `well()` transforms a proof that `n <= m` to a proof that `n + 1 <= m + 1` (the inductive case).

The `fool` function we're about to write will require one of these `_LTE` objects as an argument, to ensure that we've been fooled no more than four times prior to being fooled again.
```python
# we'll parameterize Me by a Nat indicating the number of times fooled thus far

# since we can be fooled at most five times, we'll want a handy type-level four
FOUR = S[S[S[S[Z]]]]

class Me(Generic[_N]):
    def __init__(self, name: str):
        self.name = name

    def fool(self: 'Me[_N]', prf: _LTE[_N, FOUR]) -> 'Me[S[_N]]':
        print(f'shame on {self.name}')
        return cast(Me[S[_N]], self)

def naif(name: str) -> Me[Z]:
    return Me(name)
```

The `fool` function requires that `self` has been fooled no more than four times (as witnessed by the proof `prf`), and produces an updated `Me` which has been fooled one more time than previously.
This isn't Idris (or another dependently-typed language with support for proof search), so we'll need to construct our proofs by hand!
Luckily, they're more tedious than they are difficult:
```python
if __name__ == '__main__':
    me = naif('George')
    me = me.fool(duh())
    # unfortunately, this isn't Idris, so we have to bring our own proofs
    #   each time we get fooled, attesting that we've been fooled no more
    #   than four times already
    me = me.fool(well(duh()))
    me = me.fool(well(well(duh())))
    me = me.fool(well(well(well(duh()))))
    me = me.fool(well(well(well(well(duh())))))
    # me = me.fool(no_possible_proof) # you don't get fooled again!
```

There's no possible value which can be provided for `no_possible_proof`, unless you bypass our "security" and directly invoke `_LTE()`.
Thus, the type system prevents us from getting fooled, well, again again again again again again.

---

All code for this article is available at <a href="https://gist.github.com/derrickturk/f77d467481fae1b8c2c188e884713dfa">https://gist.github.com/derrickturk/f77d467481fae1b8c2c188e884713dfa</a>.

[^1]: This is (still) not a Python blog! Don't worry, at some point, we will return to more "exotic" languages. My goal is to illustrate interesting concepts in a familiar language to help introduce data scientists and new programmers to a wider world.
