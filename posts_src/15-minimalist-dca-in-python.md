% Minimalist DCA in Python

It's important, as a rule of thumb, when operating or investing in a firm which produces a physical commodity, to have the ability to reliably quantify the expected future production of the commodity given the firm's assets.
In the oil and gas industry, we have many different ways to forecast future production from an oil or gas well.
Some rely on <a href="https://petrowiki.org/Pressure_transient_testing">detailed measurements</a> and explicitly incorporate <a href="https://petrowiki.org/Reservoir_simulation">detailed mathematical models of flow physics</a>.
Others use whatever historical data we can scrape together and a bit of curve-fitting.

Today, we're talking about the second kind.

### Arps DCA in a Nutshell
Sophisticated approaches are great for the operator, who should have access to daily or higher-resolution rate and pressure data and a detailed understanding of the well and completion design.
For us on the outside, trying to put a value on an operator's assets, we're usually limited to "public data": for instance, the monthly volume data for Eddy County, New Mexico which we retrieved <a href="/posts/14-xmhell.html">previously</a>.

This nudges us toward applying empirical <a href="https://petrowiki.org/Production_forecasting_decline_curve_analysis">decline-curve analysis</a> (DCA).
DCA is all about curve-fitting: we know, from experience, the rough functional forms which should characterize the rate vs. time relation for oil and gas wells; we'll apply optimization algorithms to find parameters to these functions which best fit the observed data.
We can then use the best-fit parameters to forecast future production.

There is infinite nuance to be explored in this topic, very little of which we'll discuss in today's article.
The short version is this: in the early 1940s, while physicists in New Mexico toiled perfecting the most destructive weapon in human history, J. J. Arps at the British-American Oil Company (now Gulf Canada) was researching and documenting a "weapon" which would arguably---eventually---result in a similarly spectacular destruction of capital in the late 2010s.
Sarcasm aside, Arps published several papers which changed the oil and gas industry permanently.
He documented that production from flowing oil and gas wells tended to follow one of a few different functional forms
(*exponential*, *harmonic*, or *hyperbolic*) depending on the details of the fluid being produced and the reservoir physics.
These equations were straightforward to fit to observed data even using the technology of the day (that is, graph paper).

Like atomic energy, the Arps equations can be used for both beneficial and destructive purposes.
When applied with good sense by trained engineers in appropriate situations, Arps decline fits are completely adequate for forecasting production; indeed they are the most commonly used tool for estimating reserves for publicly-traded oil and gas companies.
Notice the caveats.
The advent of "unconventionals" (shale gas, shale oil, coalbed methane, etc.) throws a kink into the key assumptions.
Specifically, without going into too much detail (ask an *active* reservoir engineer!), wells in these assets generally experience multiple *flow regimes*, each observable on an economically material timescale, which alter the functional form required to correctly fit the rate vs. time data.

That is, a hydraulically-fractured horizontal shale oil well may begin production on an apparent Arps hyperbolic decline with a *b* exponent of 2.0; but it may begin to follow something closer to a hyperbolic decline with a *b* of 0.5, or even an exponential decline, within the first year.
A naïve curve fit, especially to early data, can result in a shockingly overstated *estimated ultimate recovery* (EUR).
In my opinion, this underappreciated---especially by investors---phenomenon is responsible for a large portion of the gap between producers' investor relations statements and reality, and the frequent reserves write-downs, in recent years.

Plenty of smart people have worked on this problem, and good solutions exist.
There are a number of decline curve models which account for multiple flow regimes; for example, my friend David Fulford has published extensively on the <a href="https://www.onepetro.org/conference-paper/SPE-167242-MS">transient hyperbolic relation</a> as well as <a href="https://www.onepetro.org/conference-paper/SPE-174784-MS">Bayesian techniques</a> for fitting it probabilistically (disclosure: I'm a co-author on that one).

Unfortunately, to keep things simple today, we'll just be using the original Arps relations.
As you examine the results, look for fits which illustrate these problems.
This is still a common practice in industry: you may draw your own implications.

### I Will Face Guido and Walk Backwards Into Hell
> ...in many cultures, the python is seen as a strong and powerful creature.  
> --Wikipedia

Now that I've complained about Arps decline-curve analysis but decided to use it anyway, I'm going to do the same about Python!
Engineering is, for the most part, a quantitative discipline which demands a certain degree of rigor.
Naturally, we seem to have converged on learning a programming language which is pig-slow for numeric code "out of the box" and where more-or-less anything goes, with minimal facilities for automating the task of validating the correctness of our code.

As a famous Vorlon once said, "the avalanche has already begun; it is too late for the pebbles to vote".
So, given that Python seems to be the new incumbent language of engineering automation (at least it is an improvement on Excel VBA in some ways---but not all!), we'll step through the construction of a simple Arps decline-curve forecasting system in Python.
We're going to build something at the scale that a working reservoir engineer or analyst might be able to construct as a prototype: this won't be an "industrial grade" system, but a sketch of something that'll get the job done to do quick evaluations of up to a few thousand wells.

This won't, however, be the "beginner Python" you'll frequently see in similar examples online.
We are going to take more seriously the concerns of performance and correctness.
To get there, we're going to have to address some key deficiencies of the language.

### Deficiency 0: Dependency Management
The Python ecosystem's take on package management and dependency resolution is a hot mess.
Most beginners learn how to `pip install` into the global Python's packages directory, but that's a recipe for pain as programs accumulate with mutually incompatible dependency versions.
We've got a huge variety of half-baked "solutions": `venv`, `conda`, `pipenv`, `poetry`... each with their own pain points.

We're going to take minimalism as a virtue, and use `venv` to manage a virtual environment for our project.
On Windows, we'll have to take some extra steps to acquire dependencies, but it's worth it to avoid the complexity of things like `conda`.

<a href="https://docs.python.org/3/library/venv.html">`venv`</a> is the successor to the old `virtualenv`. It's a relatively simple approach: copy the global Python interpreter to a new directory, set up `pip` and pals to install packages into that new directory, and configure the user's environment variables to point to the new system when they run `python`.
There are <a href="https://nixos.org/">cleaner</a> <a href="https://haskellstack.org">ways</a> to <a href="https://doc.rust-lang.org/cargo/">do this</a>, but at least it's not an <a href="https://anaconda.org">invasive alien ecosystem trying to take over every aspect of Python operation</a>.

We won't go into horrifying detail here, but I'll provide commands to follow along on recent versions of Windows (I know reservoir engineers).
You'll want to download the latest version of Python 3---at time of writing, Python 3.8.3---for your operating system from <a href="https://python.org">python.org</a>, if you don't already have a version installed.
Windows users should note that the default link goes to the 32-bit rather than 64-bit version of Python.
No, I don't know why the hell they do that.
You can get the 64-bit version which you almost certainly would rather have by going <a href="https://www.python.org/downloads/windows/">here</a> and choosing one of the "x86-64" options.

With Python properly installed, we can create a `venv` for our project.
I like to create these in the project's working directory with the name `.venv`:
```
python -m venv .venv
```
Now, we can "activate" the environment with (Windows):
```
.venv\Scripts\activate
```
or (Unix-likes):
```
source .venv/bin/activate
```

From this point on (in our current shell session), everything we do will touch only the `venv`: `python` will refer to the `venv`'s Python interpreter; `pip install` will install into the `venv`, and so on.

Should we choose to leave the virtual environment, we can simply run:
```
deactivate
```
to return to our original configuration.

### Deficiency 1: Static Analysis
The next problem we'll address is the lack of tools for ensuring the correctness of our programs.
There are many approaches to program correctness, all imperfect.
Historically Python's community has encouraged a reliance on automated testing, but to quote Edsger Dijkstra:

> Program testing can be used to show the presence of bugs, but never to show their absence!

Type systems are a common discipline for statically (that is, before the program is run) checking the correctness of programs with respect to desired properties.
These properties might include anything from "the program never exhibits undefined behavior (i.e. crashes)" to "the program takes a finite list of input strings, parses them to integers if possible, and produces a same-length list of integers in sorted order as output".

Fortunately, there has been significant work on "gradual typing" in recent years, resulting in the introduction of bolt-on typecheckers for "dynamic" languages like Python.
The most popular is `mypy`, which originated at Dropbox and counts Python creator Guido van Rossum among its developers.

`mypy` is conservative: it implements a relatively simple type system, and allows use of an `Any` type to hand-wave Python idioms which can't easily be statically typed.
Nonetheless, it's a powerful tool for catching common programming errors (including things like misspelled variable names, which aren't usually though of as "type errors").

We can install `mypy` into our virtual environment with:
```
pip install mypy
```

To typecheck a given program `prog.py`:
```
mypy prog.py
```

### Deficiency 2: Numeric Performance
Engineering code is often "numeric": we want to work efficiently with (approximations of) things like real numbers, vectors, matrices and so on.
Python's built-in data structures are an awkward fit for this sort of code.
As an approximation to a vector or matrix, a Python `list` is a poor substitute for a C or Fortran style array.
Python's `list`s hold their elements "boxed" behind pointers, so each element access into a `list` of `float` follows at least two levels of indirection, compared to a simple calculation of (base address + index * element size) required to index into a contiguous array.

Additionally, Python's built in numeric operations are implemented on a "scalar" basis (one value at a time) and have significant overhead due to the requirement for runtime type checking and interpretation.
We can loop over sequences and apply these operations element-wise, but we'll pay this overhead again and again for each element.
We'd prefer a representation where we store a type tag along with a contiguous, homogeneous array of elements, and provide efficient compiled-code "vectorized" operations over entire arrays. 

Fortunately, this exact representation is provided by `numpy`, the library which forms the basis for effectively all serious numerical computing in Python.
`numpy` combines fast compiled libraries for linear algebra and array manipulation with a "Pythonic" API which lets us interact easily with optimized arrays using the same interface as Python's built-in sequence types.

`numpy` is easy to install from the Python Package Index on non-Windows operating systems (with `pip install numpy`).
Windows users will want to visit Christoph Gohlke's site (<a href="https://www.lfd.uci.edu/~gohlke/pythonlibs/">https://www.lfd.uci.edu/~gohlke/pythonlibs/</a>) and download the correct "wheel" for the latest version of "numpy+mkl", depending on their Python version.

For example, at time of writing, the latest version available for 64-bit Python 3.8 is `numpy-1.18.4+mkl+cp38-cp38-win_amd64.whl`.
Download this file, then install directly from the downloaded wheel with
```
pip install numpy-1.18.4+mkl+cp38-cp38-win_amd64.whl
```
substituting the full path to the downloaded wheel.

We'll also want to make use of the `scipy` package, which provides a variety of scientific computing tools built on `numpy`.
Specifically, we're going to use the library of efficient optimization routines it provides.
Users on all platforms should be able to install this with:
```
pip install scipy
```

We will use `matplotlib` to visualize our fits.
```
pip install matplotlib
```

Finally, we'd like `mypy` to be aware of the types provided by `numpy`.
The `numpy` project doesn't officially provide support for this yet, but the <a href="https://github.com/numpy/numpy-stubs">`numpy-stubs`</a> repository contains their work-in-progress implementation, which we'll install and use.
We can install the package directly from GitHub with `pip`:
```
pip install git+https://github.com/numpy/numpy-stubs.git
```

### Deficiency 3: Simple Record Types
Python supports user-defined data types through *classes*.
Unfortunately, these often give us "too much".
Object-oriented classes combine data encapsulation and abstraction (which we want) with features like mutability, <a href="/posts/10-open-recursion-the-essence-of-oo.html">dynamic dispatch</a>, and inheritance (which we don't need).

This program will deal with simple data: well API numbers, rate and time arrays, and decline parameters.
None of these require the more complex features of classes: in fact, we'd prefer to treat them as <a href="https://en.wikipedia.org/wiki/Immutable_object">immutable</a> "bags of data"; immutability is a helpful tool for correctness as it reduces the number of possible program states and prevents important <a href="https://en.wikipedia.org/wiki/Invariant_(mathematics)#Invariants_in_computer_science">program invariants</a> from being yanked out from under us by mutation.

Python provides a few solutions to this, ranging from hand-written "record classes" to `namedtuple`.
We'll use the relatively new <a href="https://docs.python.org/3/library/dataclasses.html?highlight=dataclasses#module-dataclasses">`dataclasses`</a> package, which is included with Python since version 3.7.

`dataclasses` provides a decorator which we can use to turn a class containing member definitions with type annotations into a fully featured "record class" similar to what we might write by hand.
It will automatically write constructors (`__init__`), display functions (`__repr__`), structural equality comparisons (`__eq__`), and other common functions for us from our annotated member definitions.
We can also use the `frozen` feature to achieve (alas, dynamically-enforced) immutability.

With our tools (`venv`, `mypy`, `numpy`, `scipy`, `matplotlib`, `dataclasses`) in place, we are now ready to begin implementation.
In this article, we'll present the implementation a piece at a time, not necessarily in the order in which they occur in the final product but rather in roughly the order we might write them.
The completed program can be previewed on GitHub at <a href="https://github.com/derrickturk/pydca">https://github.com/derrickturk/pydca</a>.

### Implementation I: Forecasting
Let's begin by writing a `dataclass` which describes a single Arps hyperbolic fit.
The hyperbolic decline subsumes the harmonic (*b* = 1) and exponential (*b* = 0) declines as special cases.

We'll need to store the three decline parameters (*q<sub>i</sub>*, *D<sub>i</sub>*, and *b*), and provide functions for evaluating the rate vs. time relation.

```python
import numpy as np
import dataclasses as dc

@dc.dataclass(frozen=True)
class ArpsDecline:
    qi: float # daily rate
    Di_nom: float # nominal annual decline
    b: float # unitless exponent

    def __post_init__(self):
        if self.qi < 0.0:
            raise ValueError('Negative qi')
        if self.Di_nom < 0.0:
            raise ValueError('Negative Di_nom')
        if self.b < 0.0 or self.b > 2.0:
            raise ValueError(f'Invalid b: {self.b}')

    # time (years)
    # returns (daily rate)
    def rate(self, time: np.ndarray) -> np.ndarray:
        if self.b == 0:
            return self.qi * np.exp(-self.Di_nom * time)
        elif self.b == 1.0:
            return self.qi / (1.0 + self.Di_nom * time)
        else:
            return (
              self.qi / (1.0 + self.b * self.Di_nom * time) ** (1.0 / self.b)
            )
```

The math itself isn't very interesting.
It's a direct translation of the Arps equations for the hyperbolic, harmonic, and exponential cases, dispatched on the value of *b*.
We use ordinary Python arithmetic operators (`*`, `**`, etc.), but rely on the fact that `numpy` *overloads* these for arrays to provide efficient vectorized operations.

We indicate this in the *type signature* of the `rate` function.
These type annotations are not used by the Python interpreter during code execution, but are available to tools like `mypy`.
For functions, we write
```python
def f(x: int, y: float) -> str:
   ...
```
to mean "`f` takes an `int` named `x` and a `float` named `y` and returns a `str`".
A function with no return type annotation returns `None` (like a `void` function in C, we use this to indicate a function called only for its side effects, usually with no explicit `return None`).
The type of `self` is automatically inferred for class instance methods.

We can also annotate variables with types when needed:
```python
x: int
```
meaning `x` is an `int`.

It's common in Python to write code where variables are not *type stable*:
```python
x = None     # type(x) == type(None)
x = 3        # type(x) == int
x = 'abcdef' # type(x) == str
```

While we can type code like this for `mypy`, it makes our programs harder to reason about.
We'll keep variables type stable with the exception of one special case we'll discuss later.

We use the `dataclasses.dataclass` decorator to turn our definition of `ArpsDecline` into a "data class".
The `frozen=True` argument makes objects of our class immutable---their attributes can't be changed after they're created.
`dataclasses` writes a constructor (`__init__`) function for us, based on the declared members of the class.
The `__post_init__` method is automatically called after `__init__` and we can use it to provide some validation logic---in this case, ensuring that parameters fall within the appropriate physical bounds.

Let's use `matplotlib` to draw rate vs. time curves for some example declines:
```python
import matplotlib.pyplot as plt # type: ignore

def plot_examples():
    exp_decl = ArpsDecline(1000.0, 0.9, 0.0)
    hyp_decl = ArpsDecline(1000.0, 0.9, 1.5)
    hrm_decl = ArpsDecline(1000.0, 0.9, 1.0)

    time = np.array([m / 12.0 for m in range(0, 5 * 12)])
    exp_rate = exp_decl.rate(time)
    hyp_rate = hyp_decl.rate(time)
    hrm_rate = hrm_decl.rate(time)

    plt.semilogy(time, exp_rate)
    plt.semilogy(time, hyp_rate)
    plt.semilogy(time, hrm_rate)

    plt.savefig('examples.png')
    plt.close()

plot_examples()
```

`matplotlib` doesn't provide `mypy` type information, so we'll tell `mypy` to skip type checking for anything coming from that module with `#type: ignore`.
Assuming we've saved the code so far as `pydca.py`, we can typecheck it with `mypy pydca.py` and should see a message indicating success.

We can run the program with `python pydca.py` to generate the example plots.
If you installed Python without the Tcl/Tk libraries, you'll want to set the environment variable `MPLBACKEND` to `agg` beforehand (on Windows, do this with `MPLBACKEND=agg`).

An image should be written to `examples.png`:  
<!-- TODO /.. -->
<img alt="example plots" src="images/pydca_examples.png">

### implementation pt 2: fitting

### implementation pt 3: parsing

### results

Next week on...
