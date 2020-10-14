% Jupyter Ascending: A Retrograde Development

When was the last time you saw a Python program on "oil and gas data science LinkedIn" not published as a Jupyter notebook?
For that matter, when was the last time you saw a Python programming course---or worse, a "Python for data scientists" course---advertised which didn't heavily feature the use of Jupyter notebooks as a "beginner's programming environment"?
As the oil-to-code pipeline expands, it seems that a whole new generation of engineers-turned-programmers is being trained on "notebook-driven development".
This is not a good idea.

## Two Old Ideas
<a href="https://en.wikipedia.org/wiki/Notebook_interface">Computational notebooks</a> have a history dating back to the late 1980s, when they were introduced as a user interface for the Mathematica programming language / symbolic algebra environment / cash cow.
They combine two much older, better, but less popular ideas into a synthesis which manages to be less than the sum of its parts.
We'll talk about Jupyter today, but these comments apply to notebook interfaces more generally.

For those of you fortunate enough to have lived under a complete media blackout for the last few years, Jupyter is a "universal" notebook interface to several programming languages.
It's the evolution of a project named IPython, so no points for guessing what the "Py" in "Jupyter" stands for.
Jupyter itself is a well-designed system: the Jupyter server communicates with one of a dozen or so "kernels"; popular data science languages like Julia, Python, or R (there's your answer on the "Ju" and the "R") have compatible kernel implementations.
Jupyter can run as a sort of supercharged interactive console, or as a full-fledged notebook environment.

The first idea---the more highbrow and academic parent of notebooks---is *literate programming*. 
The term is attributed to Donald Knuth, who invented the technique during the implementation of the <a href="https://en.wikipedia.org/wiki/TeX">TeX</a> typesetting system.
Knuth's key, correct, observation was that programs should be written for human readers to understand, not just for computers to execute.

To that end, he conceived a system in which the source code of a program could be interleaved with a prose narrative, and re-arranged for readability rather than to suit the demands of a language implementation.
This combined text could be translated to both a compilable or interpretable canonical set of source code, and an elegantly-typeset human-readable document.

It's a beautiful idea, but one---as the description suggests---perhaps over-complicated by the desire to liberate a program text completely from the ordering imposed by language specifications and implementations.
A nice, accessible, example of "pure" literate programming found in the data science world is the use of `Sweave` in the R programming language.
`Sweave` is a transliteration of Knuth's "weave" program concept from the original WEB literate programming system into the R language. A `Sweave` document can interleave LaTeX markup and R code, embedding results (including graphics) into the output document.

The second idea---which evolved in the more vulgar world of practical programming---is *interactive image-based development*.
Most programming languages, even many of those with interactive interpreters like Python or Haskell, assume that program development happens in "batch mode", with program text stored in source files and a re-compilation or module re-load after each change to the program.
There is a lesser-known lineage of programming language implementations which embrace an "online mode" for program development.

In these languages (the only ones in "common" use are Lisp and Smalltalk), we don't think of a program as a set of canonical source text, but as a running *program image*.
These languages' interpreters and compilers (the lines get blurry, especially in Lisp) give the programmer the ability to inspect or alter data structures in running programs, redefine functions and classes on the fly, and many other "reflective" tricks.
The consequence is a challenge to our "traditional" notion of what a program is; in this model a program is a mutable thing, which can be captured in a "snapshot" at a point in time, but which may not even map onto a single "canonical" set of source code!
(Consider: we use the first version of function `f` to evaluate the variable `x`, then redefine `f` to use the stored value of `x` in its implementation...)

It's a unique experience, and one I recommend every programmer try at least once, but it's one I also find deeply unsettling.
How can we version-control an image?
How can multiple programmers collaborate on a large program in an image-based environment?
How can we ever take advantage of *static analysis* tools like type checking or "linting" when the program text itself is a mutable thing?
I'm sure that there are perfectly good---at least in theory---image-based answers to these questions, but for many programmers (including me), it can be an uncomfortable and alien world.

## Oh, What a Tangled WEB we `WEAVE`
Computational notebooks combine literate programming's interleaving of source text with prose descriptions and multimedia output with image-based development's interactivity and mutability.
It's the combination of the two ideas that kills notebooks as a practical tool for software engineering.

Unlike "traditional" literate programming, which used dedicated tools to re-arrange source text in a way suitable for compilation or interpretation, notebooks leverage the mutability inherent to image-based environments and leave it to the human writer (or reader!) to run "cells" in whichever order is desired.

This means that program correctness can depend on human-specified execution order, including things like "this cell must be run twice" or "this cell should not be run at all".
While a more advanced species might be able to agree on, and follow, a convention of "notebooks must always be correct when run top-to-bottom", human programmers just aren't capable of this level of discipline.
In other words, a notebook corresponds less to "a program" and more to "the potentiality to become a program".
This makes notebooks an inadequate medium for building serious software, by which I mean "software which is expected to reliably perform a task".

In addition to the impossibility of guaranteeing one canonical ordering of source code for a notebook "program", notebooks lack facilities for building larger programs---"modules" are non-existent; every notebook program is *at best* one giant source file (and at worst, a mess of an image snapshot arrived at by out-of-order execution of now-redacted program text!).
This discourages careful, re-usable design and encourages the overuse of global state and ad-hoc solutions.

As a final note, notebooks make version control a huge pain---re-running the "same" program to get the "same" output fills version control logs with spurious diffs.
Add-on solutions exist to filter and ignore this noise, but it speaks to the fundamental disconnect between the notebook paradigm and the generally accepted practices of software engineering.

## We Have Purposely Trained Him Wrong, As A Joke
When I see courses aimed at beginning programmers, or beginning data scientists (but I repeat myself), pushing the notebook-first (or notebook-only) model, I am forced to conclude that one of two possibilities holds.
Either:

  - the course designers are ignorant of the issues notebooks pose for reliable and robust software design, or  
  - the course designers are malevolent, but competent, programmers attempting to sabotage their future competition!  

---

I'm not the first to advance this, apparently, unpopular opinion.
For a more academic take on the same topic, see <a href="http://web.eecs.utk.edu/~azh/blog/notebookpainpoints.html">http://web.eecs.utk.edu/~azh/blog/notebookpainpoints.html</a>.
For a much funnier take, see <a href="https://docs.google.com/presentation/d/1n2RlMdmv1p25Xy5thJUhkKGvjtV-dkAIsUXP-AL4ffI/edit#slide=id.g362da58057_0_1">https://docs.google.com/presentation/d/1n2RlMdmv1p25Xy5thJUhkKGvjtV-dkAIsUXP-AL4ffI/edit#slide=id.g362da58057_0_1</a>.

There are literally dozens of us!
