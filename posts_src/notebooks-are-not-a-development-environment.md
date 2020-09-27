% Jupyter Ascending: A Retrograde Development

When was the last time you saw a Python program on "oil and gas data science LinkedIn" not published as a Jupyter notebook?
For that matter, when was the last time you saw a Python programming course advertised which didn't heavily feature the use of Jupyter notebooks as a "beginner's programming environment"?
As the oil-to-code pipeline expands, it seems that a whole new generation of engineers-turned-programmers is being trained on "notebook-driven development".
This is not a good idea.

## Two Old Ideas
<a href="https://en.wikipedia.org/wiki/Notebook_interface">Computational notebooks</a> have a history dating back to the late 1980s, when they were introduced as a user interface for the Mathematica programming language / symbolic algebra environment / cash cow.
They combine two much older, better, but unfortunately less popular ideas into a synthesis which manages to be less than the sum of its parts.
We'll talk about Jupyter today, but these comments apply to notebook interfaces more generally.

For those of you fortunate enough to have lived under a complete media blackout for the last few years (how 'bout them 'Stros?), Jupyter is a "universal" notebook interface to several programming languages.
It's the evolution of a project named IPython, so no points for guessing what the "Py" in "Jupyter" stands for.
Jupyter itself is a well-designed system: the Jupyter server communicates with one of a dozen or so "kernels"; popular data science languages like Julia, Python, or R (there's your answer on the "Ju" and the "R") have compatible kernel implementations.
Jupyter can run as a sort of supercharged interactive console, or as a full-fledged notebook environment.

...

Oh yeah, and it's damn near impossible to version control the bastards while providing any kind of meaningful "diffs".

...

For a more academic take on the same topic, see http://web.eecs.utk.edu/~azh/blog/notebookpainpoints.html
For a much funnier take, see https://docs.google.com/presentation/d/1n2RlMdmv1p25Xy5thJUhkKGvjtV-dkAIsUXP-AL4ffI/edit#slide=id.g362da58057_0_1

There are dozens of us!
