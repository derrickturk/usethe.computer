## Open Recursion: the Essence of Object Oriented Programming?

What is object-oriented programming really about? What's so special about "late binding"? And why do I have to pass `self` around everywhere in Python?
We'll take a meandering path in today's post which will try to answer each of these questions, and build our own miniature object system along the way.

We'll be using C++ and Python to explore these ideas, but I hope you'll make the attempt to follow along even if you've never used these languages.
C++ was chosen because it makes some of the distinctions we need syntactically explicit; Python because it's more familiar to the average modern programmer, engineer, or data scientist - and because it'll give us a chance to build our own implementations of these concepts "from scratch" using just a simple subset of the language.

### A Bad Idea Which Could Only Have Originated in California

Many computer scientists and programmers have proposed definitions of object-oriented programming (henceforth "OO" or "OOP").
It's a bit of a "blind men and the elephant" situation, but the majority of definitions seem to agree on a few crucial ideas.
Let's consider a few.

> Object-oriented programming is an exceptionally bad idea which could only have originated in California.  
> -- Edsger W. Dijkstra, attributed

Ok, maybe not that one! As much as I admire Dijkstra's cutting wit, there's actually no written citation for the quote. And, for that matter, Simula 67 was created by two of Dijkstra's fellow Europeans before Alan Kay and his team in Palo Alto even began work on what would become Smalltalk.

I'm not a "fan" of object-oriented programming myself. In some ways, I think the technique we're about to discuss is "too powerful" - it complicates reasoning about code when used in places it's not absolutely necessary. I also find that "object-oriented languages" in practice carry around a whole host of cultural baggage: dynamic typing, implementation inheritance, performance-sapping heavyweight runtimes, and a focus on imperative code which manipulates shared mutable state.

Nonetheless, there's a specific scenario where the "essence" of OO really shines - and we can recreate this key insight no matter what language we're working in.

> The three pillars of object-oriented programming are *encapsulation*, *inheritance*, and *polymorphism*.  
> -- Unknown, now omnipresent in computing folklore and CS101 textbooks

I hate this definition, not because it's wrong, but because it's too broad! I tried to find out who originated the "three pillars" framing (which was burned into my mind permanently by my high school computer science classes), but to no avail. It seems to have simply permeated into the groundwater of "intro to CS" resources.

The reason I find this definition too broad is that we find two of these "pillars" in decidedly non-OOP contexts, and the third is controversial even within the OO design space, with many considering it to be a mistake.

Encapsulation is the idea of designing software systems in a modular way, binding together data structures with the operations available on them, and constraining their use via a formally specified public interface. This is a great idea - perhaps even the key idea for designing large software systems.
However, it's neither new with, nor unique to, object-oriented systems.
D. L. Parnas's seminal paper "On the Criteria To Be Used in Decomposing Systems into Modules", from 1972, makes a case for "information hiding" - in other words, encapsulation - as a key principle for program design.
We find it expressed in various programming languages through "abstract data types", "modules", "namespaces", "access specifiers", and a host of other mechanisms - not just classes with "public" and "private" members.
And, for that matter, many "OOP" languages (for instance, Python) provide only minimal support for encapsulation by information hiding.

Inheritance is controversial in the OOP community, if such a thing can still be said to exist (perhaps it's been swallowed up by the Agile cult).
Some draw a distinction between "implementation inheritance" (naughty) and "interface inheritance" (pure and virtuous).
Like a thinly-veiled leonine allegory, we shall come to know "implementation inheritance" by another name.
For now I'll merely remark that the fundamental issue with "implementation inheritance" is that "an A is-a B" and "an A is-implemented-in-terms-of-a B" are very different statements, with very different implications for program design.
Languages with implementation inheritance conflate the two.

Let's take a look at a short "object-oriented" program snippet in C++ to understand the distinction.
```c++
class Truck {
  public:
    ...

    virtual void go_camping()
    {
        engine_.start();
        ...
    }

    virtual void haul(const Cargo& c)
    {
        engine_.start();
        ...
    }

    virtual double fuel_remaining()
    {
        ...
    }

    ...

  private:
    InternalCombustionEngine engine_;
    ...
};

class TeslaCybertruck : public Truck {
  public:
    ...

    virtual void go_camping()
    {
        battery_.check_voltage();
        ...
    }

    virtual void haul()
    {
        battery_.check_voltage();
        ...
    }

    /* note we've inherited both the existence of
     *   and definition of a fuel_remaining() function
     *   from Truck */

    ...

  private:
    ElectricMotor motor_;
    Battery battery_;
    ...
};
```

We have a few problems here. It's correct to say that a Cybertruck "is a" Truck - it can do all the things any Truck can do. That's interface inheritance - a Cybertruck can `go_camping` and so forth.

But it's not correct to say that a Cybertruck "is implemented as" a Truck. It doesn't need an `InternalCombustionEngine`, but in the C++ type system, it gets one anyway (and a `fuel_remaining` function) by virtue of being defined as a subclass of `Truck`. That's implementation inheritance; the idea is to allow "code re-use", but sometimes it isn't appropriate.

A better design, using only interface inheritance, might look like this - we'll need some boilerplate to pull this off in C++: 
```c++
// an "interface" for trucks - this is what C++ calls an "abstract class"
class Truck {
  public:
    // this is "C++ese" for an interface method with no default implementation
    virtual void go_camping() = 0;

    virtual void haul() = 0;
};

class GasPoweredTruck: public Truck {
  public:
    ...

    virtual void go_camping()
    {
        engine_.start();
        ...
    }

    virtual void haul(const Cargo& c)
    {
        engine_.start();
        ...
    }

    double fuel_remaining()
    {
        ...
    }

    ...

  private:
    InternalCombustionEngine engine_;
    ...
};

class TeslaCybertruck : public Truck {
  public:
    ...

    virtual void go_camping()
    {
        battery_.check_voltage();
        ...
    }

    virtual void haul()
    {
        battery_.check_voltage();
        ...
    }

    ...

  private:
    ElectricMotor motor_;
    Battery battery_;
    ...
};
```

Here we only make the claim we can defend: that a Cybertruck can do everything a gasoline-powered truck can do. As an aside, those of you who know enough C++ to understand why `go_camping` and `haul` are `virtual` functions already understand open recursion, if not in those terms!

In any case, we'd be hard-pressed to argue that implementation inheritance is "fundamental" in any way to object-oriented programming.
As a counter-example, I'd propose that the Microsoft COM model might be one of the most purely object-oriented systems ever put into practice, and it operates purely via interfaces like our `Truck` above - no implementation inheritance.
Code reuse is achieved through composition, directly modeling the "is-implemented-in-terms-of" relationship. 

We find ourselves at last at, in my opinion, the true crux of object-oriented programming: polymorphism. But we are going to need to get more specific to make any further progress.
Let's consider a much better definition, or at least a much more qualified one, from the person who coined the term "object-oriented programming".

> OOP to me means only messaging, local retention and protection and hiding of state-process, and extreme late-binding of all things.  
> -- Alan Kay

Kay's conception of object-oriented programming starts with a philosophical vision of a more "organic" world of computing, constructed from "objects" which themselves act as tiny computers, communicating only via "messages" passed between them.
Frankly, I've never seen "message passing" implemented in a way meaningfully distinct from "function calls" - it seems rather more a distinction of viewpoint than of semantics.

The ideas of limiting the scope of mutable state and hiding complex logic behind simple interfaces are certainly good ideas for program structure, but don't seem to me unique to OOP.
Let's focus on that last bit.

"Late-binding"? What does Kay mean by that?
Consider the truck example from before.
We'd like to be able to write code which is agnostic to the specific sorts of trucks it operates on, using only the known abilities of all trucks.

The general idea of code which is valid for many types of inputs, but which operates in potentially different ways on each combination of input types, is called *polymorphism*, from Greek roots meaning "many forms".
Programming languages implement polymorphism in many different ways - often one language will contain several approaches to polymorphism.

There's a great <a href="http://lucacardelli.name/Papers/OnUnderstanding.A4.pdf">paper</a> from 1985 by Luca Cardelli and Peter Wegner called "On Understanding Types, Data Abstraction, and Polymorphism" which lays out a useful framework for understanding the varieties of polymorphism commonly found in programming languages.

In their framework, the polymorphism we're interested in is *inclusion polymorphism* - the ability to operate on elements of various types according to those types' membership in defined classes.
This is sometimes called *subtype polymorphism*, because it's typically implemented by allowing operations to be defined for one class and refined or overridden by *subclasses* of that class, which are treated as subtypes of the superclass in the language's type discipline.

This is the facility which lets us write code like:
```c++
void beach_party(std::vector<Truck&> trucks)
{
    for (Truck &t : trucks) {
        t.haul(Cargo::BEER);
        t.go_camping();
    }
}
```

Each truck in the vector which we operate on may be of a different (sub)type - we can happily have a beach party with ordinary trucks and Cybertrucks, with each executing the appropriate definitions of `haul` and `go_camping`.

We can do this because the call to each truck's relevant functions is *late bound*, *dynamically dispatched*, or *virtually dispatched* - these terms are (mostly) equivalent. The meaning is by contrast to *early binding*, where the compiler would choose the function to be called based on the statically-known type of the variable `t` (`Truck` - note this would end poorly with our example code, in a pure virtual call exception) rather than generate code to choose an implementation at runtime based on the "dynamic type" (`GasPoweredTruck` or `Cybertruck`) of each object.

There's another, even more interesting implication of the combination of late binding with inclusion polymorphism.

### Call `Me`, Maybe

It's commonplace for new code to call old code. We've taken it for granted for many years, since the first "subroutine library", that we can build up programs one piece at a time, re-using the work we've already done when possible.

A more novel trick is for old code to call new code!
In our `beach_party` example, for instance, we could later define a new subclass of Truck with different definitions of our required functions.
Assuming our compiler hasn't changed the <a href="https://en.wikipedia.org/wiki/Application_binary_interface">ABI</a> out from under us (abstractions always leak, eventually), we can add objects of this new class (say, `SolarTruck`) to the vector we pass to `beach_party` and it'll do the right thing - even if we don't recompile `beach_party` (again, assuming the ABI has not changed).

This is enabled by late binding - we don't need the compiler to be aware of every possible callee, just to know the "protocol" to call them dynamically.
Of course, we can pull this off without inclusion/subtype polymorphism - we just need to ability to call a function via indirection; for example, by passing function pointers in C:
```c
#include <stdio.h>

typedef void (*action_t)(int);

void do_it(action_t action)
{
    action(3);
}

void print_it(int x)
{
    printf("x is %d\n", x);
}

void print_twice_it(int x)
{
    printf("twice x is %d\n", x * 2);
}

int main(void)
{
    do_it(print_it);
    do_it(print_twice_it);
    return 0;
}
```
> `x is 3`  
> `twice x is 6`

However, subtyping grants us a surprising ability.
Here's a somewhat contrived example in C++:
```c++
#include <iostream>

struct Doubler {
    void do_it(int x)
    {
        hook(x * 2);
    }

    virtual void hook(int x)
    {
        std::cout << x << '\n';
    }
};

struct FancyDoubler : public Doubler {
    // we "inherit" the definition of do_it from Doubler

    virtual void hook(int x)
    {
        std::cout << ">>>>> " << x << " <<<<<\n";
    }
};

struct Caller {
    int val;

    void call(Doubler &callee)
    {
        callee.do_it(val);
    }
};

int main()
{
    Caller clr = Caller { 5 };
    Doubler d;
    FancyDoubler fd;

    clr.call(d);
    clr.call(fd);
}
```
> `10`  
> `>>>>> 10 <<<<<`

We can write code in a superclass which calls code in a subclass - even if the subclass is written (and compiled) much later.
Even though the `do_it` function is not itself dynamically dispatched, it provides a "template" which can be filled in by each subclass's definition of the `hook` function, which is late-bound.
The C++ community uses this structure frequently and refers to it as the "non-virtual interface" pattern.
For our purposes, though, we're less interested in the benefits C++ programmers derive from hiding late binding behind an early-bound interface, and more interested in the odd interplay of inclusion/subtype polymorphism and late binding. 

To me, this is perhaps the single most interesting feature of "object-oriented programming".
We can use this feature to build frameworks (what's the difference between a library and framework? you call a library, a framework calls you) which can be extended by code written later, in a very controlled manner.

Type theorists and programming language researchers refer to this feature of the semantics of object-oriented languages as *open recursion*.
"Recursion" makes sense - we have functions on an object calling other functions on the same object.
But what do we mean by "open"?
For a rigorous explanation, Benjamin Pierce's <a href="https://www.cis.upenn.edu/~bcpierce/tapl/">Types and Programming Languages</a> is probably the best source (it's where I picked up the terminology, although I've not been able to track down a reference for the original coinage).
Ralf Hinze's <a href="http://www.cs.ox.ac.uk/people/ralf.hinze/talks/Open.pdf">slides</a> provide a more concise sketch of the semantics.
We'll approach the topic from a more practical direction - by building up our own toy "object system".

### When I Think About You, I Touch My `self`

We can easily port our C++ program to Python:
```python
class Doubler:
    def do_it(self, x):
        self.hook(x * 2)

    def hook(self, x):
        print(x)

class FancyDoubler(Doubler):
    def hook(self, x):
        print(f'>>>>> {x} <<<<<')

class Caller:
    def __init__(self, val):
        self.val = val

    def call(self, callee):
        callee.do_it(self.val)

clr = Caller(5)
d = Doubler()
fd = FancyDoubler()

clr.call(d)
clr.call(fd)
```
> `10`  
> `>>>>> 10 <<<<<`

Modulo differences in the underlying implementation (dispatch through dictionaries vs. indirect calls through "vtables"), this program has the same semantics as our earlier C++ program. In C++ terms, all class methods in Python are `virtual` and thus permit open recursion.

What are we really doing, here, though? And why do all of our class methods need that `self` argument?
Let's pretend for a time that we are programmers from an alternate timeline where Python does not have classes.
Instead, we'll use only the imperative and functional programming features of Python.
How could we achieve the same semantics?

Consider that an object can be viewed from two perspectives: as a bundle of data with some functions that operate on it, or as a bundle of functions which share common data.
In Python and many other languages, we can achieve the latter by constructing *closures*.
The concept of *closure* in programming language semantics refers to the handling of free variables which appear in function definitions in most languages which provide functions as first-class values.

In Python, for instance, we can write:
```python
def make_adder(addend):
    def adder(augend):
        return addend + augend

    return adder

a5 = make_adder(5)
print(a5(3))
print(a5(6))
```
> `8`  
> `11`

Inside the definition of `adder` (a function which is returned as a value from `make_adder`), we say that the variable `augend` is *bound*, because it appears as a function argument.
We say that the variable `addend` is *free*, because it does not.
In Python and most functional languages, the semantics of `adder` are that the function value "closes over" the value of `addend` captured from `make_adder`.
We can think of the Python interpreter as packaging up the code which defines `adder` with the value of `addend` which `make_adder` was called with, and returning that bundle from `make_adder`.
This bundle of a function definition plus the captured values of its free variables is often itself referred to as a "closure".

There's a bit of programming folklore to the effect that "objects are a poor man's closures" - and vice versa.
Here's why: we can build "objects" out of functions which close over common state.
In our toy system, we'll model objects as dictionaries of these closures. We're going to have to use a funny keyword along the way, because Python is only imperfectly a lexically-scoped functional language.

Here's a simple "counter" object which tracks a count and provides the ability to increase or decrease it by one, as well as to retrieve the current value.
```python
def counter():
    x = 0

    def increment():
        nonlocal x
        x += 1

    def decrement():
        nonlocal x
        x -= 1

    def value():
        nonlocal x
        return x

    return {
        'increment': increment,
        'decrement': decrement,
        'value': value
    }

# we can use it like so:
c = counter()
c['increment']()
c['increment']()
c['decrement']()
print(c['value']())
```
> `1`

Let's try to recreate our "open recursion" example using this new approach.
Here's a first attempt.
We'll start by defining the `caller` and `doubler` classes via writing their closure-constructor functions.
```python
def caller(val):
    def call(callee):
        callee['do_it'](val)

    return { 'call': call }

def doubler():
    def hook(x):
        print(x)

    def do_it(x):
        hook(x * 2)

    return { 'hook': hook, 'do_it': do_it }

clr = caller(5)
d = doubler()
clr['call'](d)
```
> `10`  

So far, so good.

Now, let's try to write the new version of `FancyDoubler`.
We can model (implementation) inheritance by creating an "object" of the superclass and updating its dictionary of closures.
```python
def fancy_doubler():
    def hook(x):
        print('>>>>> ' + str(x) + ' <<<<<')

    d = doubler()
    d['hook'] = hook
    return d

fd = fancy_doubler()
clr['call'](fd)
```
> `10`  

That's not right at all!
What went wrong?

In the definition of `doubler`, the call to `hook` inside `do_it` was bound statically (that is, early) to the `hook` defined in the body of the `doubler` constructor function.
In other words, we "closed over" the early-bound definition.

To get the behavior we want back, we need to "open" the closure so that the implicit self-reference can be evaluated dynamically (late bound) rather than statically.
Hence the name "open recursion"!

We'll take the same approach which Python adopted, by establishing a protocol where each "class method" (closure, in our model) takes an explicit self-reference as a first argument.
The closures can then *indirectly* call other "class methods" via this argument, resulting in open recursion and proper late binding.

By convention, as you've probably guessed, we'll name this argument `self`.
```python
def caller(val):
    def call(self, callee):
        callee['do_it'](callee, val)

    return { 'call': call }

def doubler():
    def hook(self, x):
        print(x)

    def do_it(self, x):
        # here's the explicit "open" recursion via self
        self['hook'](self, x * 2)

    return { 'hook': hook, 'do_it': do_it }

def fancy_doubler():
    def hook(self, x):
        print('>>>>> ' + str(x) + ' <<<<<')

    d = doubler()
    d['hook'] = hook
    return d

clr = caller(5)
d = doubler()
fd = fancy_doubler()
clr['call'](clr, d)
clr['call'](clr, fd)
```
> `10`  
> `>>>>> 10 <<<<<`

Now we see the desired late binding in action!
Python doesn't provide "implicit" open recursion, like C++ does.
Instead, just like in our toy model, recursion between class methods occurs through the `self` argument.

I hope this meandering path through object orientation and open recursion has helped illustrate one of the truly unique powers of the object-oriented style of programming.
As you've seen, it complicates the semantics of OO languages, and it's easy to imagine how it can be abused to create unmaintainable tangles of subclass methods and "ping-pong" dynamic dispatch.
However, as a library or framework author, when you need to provide a "fill-in-the-blank" framework for carrying out a task, and especially when you want to provide sensible defaults for many of the blanks to be filled in, letting the client customize specific aspects as needed, there's really nothing else like it.

The technique isn't limited to "OO" languages, either.
As an example, I once implemented a system very similar to our toy system above as part of a parsing library written in Haskell.
I wanted to provide client code the ability to quickly customize a "tree walker" for abstract syntax trees from the parser, filling in only the actions they cared about (e.g. run this code when a "integer expression" node is encountered).
The solution was to provide a "base class" (in the form of a record of closures, just as we did in Python) whose elements were functions taking an explicit `self` argument, and which could be overridden in "subclass" records to provide customization points.

Open recursion is a powerful and subtle tool, worth understanding well.
I hope you'll consider reaching for it the next time you need it.
