# Open Recursion: the Essence of Object Oriented Programming?

What is object-oriented programming really about? What's so special about "late binding"? And why do I have to pass `self` around everywhere in Python?
We'll take a meandering path in today's post which will try to answer each of these questions, and build our own miniature object system along the way.

We'll be using C++ and Python to explore these ideas, but I hope you'll make the attempt to follow along even if you've never used these languages.
C++ was chosen because it makes some of the distinctions we need syntactically explicit; Python because it's more familiar to the average modern programmer, engineer, or data scientist - and because it'll give us a chance to build our own implementations of these concepts "from scratch" using just a simple subset of the language.

## a bad idea which could only have originated in California

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
    // this is "C++ese" for an interface method with no default implementation
    virtual void go_camping() = 0;

    virtual void haul() = 0;
}

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

We can do this because the call to each truck's relevant functions is *late bound*, *dynamically dispatched*, or *virtually dispatched* - these terms are (mostly) equivalent. The meaning is by contrast to *early dispatch*, where the compiler would choose the function to be called based on the statically-known type of the variable `t` (`Truck` - note this would end poorly with our example code, in a pure virtual call exception) rather than generate code to choose an implementation at runtime based on the "dynamic type" (`GasPoweredTruck` or `Cybertruck`) of each object.

There's another, even more interesting implication of the combination of late binding with inclusion polymorphism provides to us.

## call me, maybe

It's commonplace for new code to call old code. We've taken it for granted for many years, since the first "subroutine library", that we can build up programs one piece at a time, re-using the work we've already done when possible.

A more novel trick is for old code to call new code!

## when I think about you, I touch my `self`