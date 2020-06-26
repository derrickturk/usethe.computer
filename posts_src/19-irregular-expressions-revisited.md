% Irregular Expressions, Revisited

<a href="https://usethe.computer/posts/18-irregular-expressions.html">Last time</a>, we used a minimalist parser combinator library to build a parser for an oddly familiar language called OBAN. 

As a refresher, the full grammar of OBAN is:
```ebnf
digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9";
number = digit, { digit };
any_character = ? any UTF-8 code point ?
string = "<<", { (any_character - ">") | "^>" }, ">>"
triboolean = "True" | "False" | "FileNotFound";
expression = number | string | triboolean | congregation | callout
congregation = "(", [ expression, { ",", expression } ], ")"
callout = "{", [ string, "!", expression, { "&", string, "!", expression } ], "}"
```

The goal of our parser is to process OBAN documents like:
```
{ <<first>> ! (1, FileNotFound)
& <<second>> ! <<some <<text^>^>>>
& <<third>> ! { <<nested>> ! True }
}
```

And produce corresponding Python structures like[^1]:
```python
{ 'first': [1, FileNotFound]
, 'second': 'some <<text>>'
, 'third': { 'nested': True }
}
```

We'll re-use the imports and OBAN data type definitions from last time:
```python
import sys
from enum import Enum, auto
from typing import Any, Callable, Dict, List, NamedTuple, Tuple, TypeVar, Union

# OBAN AST definitions

class TriBool(Enum):
    FALSE = auto()
    TRUE = auto()
    FILENOTFOUND = auto()

class Expression:
    Wrappable = Union[
        int,
        str,
        TriBool,
        List['Expression'],
        Dict[str, 'Expression']
    ]

    def __init__(self, value: Wrappable):
        self.value = value

    def __repr__(self):
        return f'Expression({repr(self.value)})'
```

### The Trouble With Maybe
The biggest problem with our previous parser is that it produces extremely unhelpful error messages, due to the use of the <a href="https://en.wikipedia.org/wiki/Option_type">option monad</a> to represent parse failure.
This is probably fine for a parser which runs as part of an automated toolchain and processes almost-always-valid input, but is completely unacceptable for a user-facing tool.
We additionally worried about the performance impact of making copies of the input (sub)string when advancing the parser.

We'll address both of these concerns, while making only minimal changes to the parser's structure, by tweaking the "base monad" on which its built.
In other words, we'll change the definition of what it means to chain parsers together.

Instead of representing parsers by functions which take input and produce either the remaining input and a parsed value, or `None`, we'll represent parsers by functions which take input and produce either the remaining input and the parsed value, or a structured error value conveying more information about what went wrong.
We'll also slightly extend the notion of "input"; instead of copying the remaining input when returning from each parser, we'll keep the original input string fixed, but additionally pass our current position in it between parsers.
This extra state can be passed around in a purely functional way, and enables us to avoid string copies and improve performance.

Let's define types for our parser state and structured errors, and update the definition of the types of parsers themselves.
```python
# this time, we'll parse into a monad with state, context,
#   and a richer notion of failure

class ParserState(NamedTuple):
    pos: int = 0

    # methods for producing updated states

    def advance(self, chars: int) -> 'ParserState':
        return self._replace(pos = self.pos + chars)

    # is the parser at the end of input?
    def is_eof(self, input: str) -> bool:
        return self.pos >= len(input)

    # examine the next n chars from the input, starting at pos
    def slice(self, input: str, chars: int) -> str:
        return input[self.pos : self.pos + chars]

    # fail at the current pos with a given 'expected' message
    def fail(self, expected: str, committed: bool = False) -> 'ParseError':
        return ParseError(self.pos, expected, committed)

class ParseError(NamedTuple):
    pos: int
    expected: str
    committed: bool = False # is this error "un-backtrackable"? see `alt`.

_T = TypeVar('_T')
ParseResult = Union[Tuple[_T, ParserState], ParseError]
Parser = Callable[[str, ParserState], ParseResult[_T]]

# a parser which can't fail
InfallibleParser = Callable[[str, ParserState], Tuple[_T, ParserState]]
```

The parse error type contains the position at which the error occurred, and a textual description of what was expected at that position; this'll be created by the individual parsers (in some cases, our combinators will be clever enough to write it themselves).
There's also a flag for whether this error is *committed*, which we'll discuss in a bit.

The methods on the `ParserState` class make it simpler to retrieve the next few characters from the input, to advance the parser's position, to check for end-of-input, and to produce an error at the current position.
`ParserState` is, by design, immutable, so functions like `advance` produce new state objects rather than mutate the current instance.

Let's see what the new parser monad looks like in action, by implementing the `exactly`, `eof`, and `char` parsers from before.
```python
def exactly(match: str) -> Parser[str]:
    def parser(input: str, state: ParserState) -> ParseResult[str]:
        if state.slice(input, len(match)) == match:
            return match, state.advance(len(match))
        return state.fail(f'"{match}"')
    return parser

eof: Parser[None]
eof = lambda input, state: (
    (None, state) if state.is_eof(input)
    else state.fail('end of input')
)

char: Parser[str]
char = lambda input, state: (
    (state.slice(input, 1), state.advance(1)) if not state.is_eof(input)
    else state.fail('any character')
)
```

Instead of "peeling off" the relevant part of the input string, we use the methods on `ParserState` to advance the current position and view slices as needed (this being Python, the slicing operation does make a copy---in other languages, we might be able to write a "zero-copy" parser using the same idea).
When we call the `fail` method, we provide a string indicating what was expected by the parser.

When we write `chars_while`, we'll need to use more explicit manipulation of the current position:
```python
# we could build this with char, but we'll write it more efficiently by
#   directly inspecting the input one character at a time
def chars_while(p: Callable[[str], bool]) -> InfallibleParser[str]:
    def parser(input: str, state: ParserState) -> Tuple[str, ParserState]:
        chars: int = 0
        while state.pos + chars < len(input) and p(input[state.pos + chars]):
            chars += 1
        return state.slice(input, chars), state.advance(chars)
    return parser

whitespace: Parser[str]
whitespace = chars_while(str.isspace)
```

Many of our basic combinators remain nearly unchanged (and our problems with giving types to the variadic combinators also remain):
```python
# combinators work like before - but there will be a little more plumbing
#   in the error case
_U = TypeVar('_U')

# map a function over a parser's result, if it succeeds
def pmap(f: Callable[[_T], _U], p: Parser[_T]) -> Parser[_U]:
    def parser(input: str, state: ParserState) -> ParseResult[_U]:
        res = p(input, state)
        if isinstance(res, ParseError):
            return res
        x, new_state = res
        return f(x), new_state
    return parser

# a parser which always succeeds by producing a constant
def pure(x: _T) -> InfallibleParser[_T]:
    return lambda _, state: (x, state)

# the opposite - a parser which always fails (with a given "expected" message)
def fail(expected: str) -> Parser[_T]:
    return lambda _, state: state.fail(expected)

# replace a successful result by a constant
def cmap(x: _T, p: Parser[_U]) -> Parser[_T]:
    return pmap(lambda _: x, p)

# these are harder to type in Python - we can do them for the fixed-arity case,
#   but it's just too convenient to let these take *args

# chain multiple parsers together and produce a tuple of their results,
#   if all succeed
def chain(*parsers: Parser[Any]) -> Parser[Tuple[Any, ...]]:
    def parser(input: str, state: ParserState) -> ParseResult[Tuple[Any, ...]]:
        results = list()
        for p in parsers:
            res = p(input, state)
            if isinstance(res, ParseError):
                return res
            val, state = res
            results.append(val)
        return tuple(results), state
    return parser
```

### Commitment Issues

Let's now consider the `alt` combinator in the context of our new parsing monad.
Remember, our goal is to provide the richest possible information on parse errors.
In the old, `None`-based system, it didn't really matter what we did in the error case: either at least one parser succeeded, or none did.
Now, though, we have a problem: when `alt` fails because no sub-parser succeeded, where should we say the error occurred, and what should we say was expected?
The straightforward answer is: the error occurred at the position where the `alt` parser was run, and the "expected" message can be built by concatenating a list of all the sub-parser expectations.
This works well---to a point.

Consider the following (malformed) OBAN expression: `(1, 2, <<half-open string)`.
Obviously, this isn't valid because there's an unclosed string (no `>>`).
However, the implementation we just described would produce an error like "at position 0, expected: digit, 'True', 'False', 'FileNotFound', '<<', '(', or '{'".  
Why? Because the string parser's reported failure, inside an `alt`-based expression parser called as part of the congregation parser, would cause the expression parser to fail, which in turn would cause the outer `alt`-based expression parser to fail.
We'd report said failure at the position it began running (that is, the first character), and the "expected" message would come from the various sub-parsers of the outermost expression parser.

This is a subtle point but an important one to consider when designing human-friendly parsers.
Naïve backtracking often results in overly general error messages.
What we've missed is this: there are certain points in a parse where we know, for certain, that we are "locked in" or *committed* to one branch of an `alt`, and don't want to backtrack out of it.

When, while parsing an expression, we see an opening '`(`' character, we know that we are now parsing a congregation.
If we subsequently fail, say, because of a missing '`,`' separator character, we want to report that error all the way back to the top-level parser.
(This is analogous to the "cut" operator in Prolog, and some combinator libraries use the name "cut" for this technique.)
In our parser, we'll implement this by providing a `commit` combinator which takes an existing parser and produces a "non-backtrackable" version of it.
Under the hood, this will work by setting the `committed` flag on parse errors, which our `alt` combinator will recognize:
```python
# run multiple parsers, recovering from errors, until one succeeds;
#   fail if all parsers fail
# this parser introduces a new issue: "unbacktrackable" errors
# we usually want `alt` to backtrack when a sub-parser fails, so that it can
#   try the next parser in the sequence without consuming any input.
# however, sometimes a sub-parser "knows" that the expected grammar is locked
#   in, and we want `alt` to fail if that parser fails.
# consider parsing Python's `x if y else z`: once we see `if`, we know that
#   we're inside a conditional expression - if we get a subsequent parse error,
#   we would not want to go back and try parsing the whole thing as, say, a list
#   comprehension!
def alt(*parsers: Parser[Any]) -> Parser[Any]:
    def parser(input: str, state: ParserState) -> ParseResult[Any]:
        expected: List[str] = list()
        for p in parsers:
            res = p(input, state)
            if not isinstance(res, ParseError):
                return res
            if res.committed:
                return res # these errors can't be backtracked
            expected.append(res.expected)
        # take a crack at an "expected" message by combining sub-parser messages
        return state.fail(', or '.join(expected))
    return parser

# run a parser, but make any errors "committed"/"unbacktrackable"
def commit(p: Parser[_T]) -> Parser[_T]:
    def parser(input: str, state: ParserState) -> ParseResult:
        res = p(input, state)
        if isinstance(res, ParseError):
            return res._replace(committed=True)
        return res
    return parser

# capture zero or more repetitions of a parser
# the end-of-loop condition is a form of backtracking, so we need to check
#   for a committed error
def many(p: Parser[_T]) -> Parser[List[_T]]:
    def parser(input: str, state: ParserState) -> ParseResult[List[_T]]:
        results: List[_T] = list()
        while True:
            res = p(input, state)
            if isinstance(res, ParseError):
                if res.committed:
                    return res
                return results, state
            val, state = res
            results.append(val)
    return parser

# we can use alt to define an "optional" parser combinator, which allows but does
#   not require its base parser to match
def optional(p: Parser[_T]) -> Parser[Union[_T, None]]:
    return alt(p, pure(None))
```

We also had to recognize "committed" errors in `many`, because it too has backtracking behavior when it encounters the first parse error.

Our additional combinators look much the same as before; in some cases the implementation is identical.
```python
# finally, a way to "decide" the next parser based on
#   the result of the previous - this is what makes Parser a monad
def bind(p: Parser[_T], f: Callable[[_T], Parser[_U]]) -> Parser[_U]:
    def parser(input: str, state: ParserState) -> ParseResult[_U]:
        res = p(input, state)
        if isinstance(res, ParseError):
            return res
        x, state = res
        return f(x)(input, state)
    return parser

# we can use bind to build some interesting extensions to previous
#   parsers

# like chars_while, but requires there to be at least one matching char
# rather than "guess" an expected message, take it as an argument
def chars_while1(p: Callable[[str], bool], expected: str) -> Parser[str]:
    return bind(chars_while(p), lambda c: (
        pure(c) if len(c) > 0 else fail(expected)
    ))

# use it to build a "run of digits" parser
digits: Parser[str]
digits = chars_while1(str.isdigit, 'digits')

# use it to build an "unsigned integer" parser
integer: Parser[int]
integer = pmap(int, digits)

# capture one or more repetitions of a parser
def some(p: Parser[_T], expected: str) -> Parser[List[_T]]:
    return bind(many(p), lambda c: (
        pure(c) if len(c) > 0 else fail(expected)
    ))

# now we can start building some more interesting combinators

# mypy hates type-checking lambdas as arguments to pmap, so we'll need some
#   tuple-extractor functions
def first(t: Tuple[Any, ...]) -> Any:
    return t[0]

def second(t: Tuple[Any, ...]) -> Any:
    return t[1]

# "only" p - match what p matches, then insist on end-of-input
def only(p: Parser[_T]) -> Parser[_T]:
    return pmap(first, chain(p, eof))

# match p, followed by optional whitespace
def lexeme(p: Parser[_T]) -> Parser[_T]:
    return pmap(first, chain(p, whitespace))

# match zero or more p, separated by sep
def sep_by(p: Parser[_T], sep: Parser[_U]) -> Parser[List[_T]]:
    # mypy hates this thing's type
    # it should be:
    # def extract(t: Union[Tuple[_T, List[Tuple[_U, _T]]], None]) -> List[_T]:
    # but that fails inference when we use it as an argument to pmap
    def extract(t):
        if t is None:
            return []
        return [t[0]] + [u[1] for u in t[1]]
    return pmap(
            extract,
            optional(chain(p, many(chain(sep, commit(p))))))
```
`sep_by` has an interesting new twist: once we've seen a separator, we `commit` to the subsequent parser: this way we get informative error messages when parsing malformed constructs like `(1,2,)`

We can now (re)write our OBAN-specific parsers.
We'll use `commit` in key places where we know that the parser should "lock in" to a given construct.
For example, once we've seen "`<<`", we know that we are parsing a string, and should insist on a closing "`>>`".
```python
tribool: Parser[TriBool]
tribool = alt(
    cmap(TriBool.FALSE, exactly('False')),
    cmap(TriBool.TRUE, exactly('True')),
    cmap(TriBool.FILENOTFOUND, exactly('FileNotFound'))
)

# again, mypy isn't smart enough to follow the type of this "lambda"
def _string_extract(t):
    _1, (parts, _2) = t
    return ''.join(p[1] if isinstance(p, tuple) else p for p in parts)

string: Parser[str]
string = pmap(
    _string_extract,
    chain(
        exactly('<<'),
        commit(chain(
            many(alt(
                chars_while1(lambda c: c != '>' and c != '^',
                    'non-escape characters'),
                chain(exactly('^'), char)
            )),
            exactly('>>')
        ))
    ))

# there's a circular dependency between congregation, callout, and expression
# we'll break it in expression, by making it a "proper" function
def expression(input: str, state: ParserState) -> ParseResult[Expression]:
    parser = pmap(
        Expression,
        lexeme(alt(integer, tribool, string, congregation, callout)))
    return parser(input, state)

# in a lazier world, this would work:
# expression: Parser[Expression]
# expression = pmap(
#     Expression,
#     lexeme(alt(integer, tribool, string, congregation, callout)))

def _congregation_extract(t):
    return t[1][0]

congregation: Parser[List[Expression]]
congregation = pmap(
    _congregation_extract,
    chain(
        lexeme(exactly('(')),
        commit(chain(
            sep_by(expression, lexeme(exactly(','))),
            lexeme(exactly(')'))))
    ))

def _callout_extract(t):
    return { kv[0]: kv[2] for kv in t[1][0] }

callout: Parser[Dict[str, Expression]]
callout = pmap(
    _callout_extract,
    chain(
        lexeme(exactly('{')),
        commit(chain(
            sep_by(chain(
                    lexeme(string),
                    lexeme(exactly('!')),
                    expression
                ),
                lexeme(exactly('&'))),
            lexeme(exactly('}'))
        ))
    ))

# since `lexeme` skips trailing whitespace, we need to allow leading whitespace
#   before a document explicitly, then ensure end-of-input
document: Parser[Expression]
document = pmap(second, chain(whitespace, only(expression)))

def main(argv: List[str]) -> int:
    for line in sys.stdin:
        res = document(line.rstrip(), ParserState())
        if isinstance(res, ParseError):
            print(f"{' ' * res.pos}^ expected: {res.expected}", file=sys.stderr)
        else:
            x, _ = res
            print(f'> {x}')
    return 0

if __name__ == '__main__':
    sys.exit(main(sys.argv))
```

Our `main` program reads one line at a time and tries to parse it as an OBAN document.
In the event of a parse error, we print a caret at the appropriate location and indicate what was expected.

You can see an example interaction below:  
<img class="centered" alt="an animated example" src="/images/parser_optim.gif" width="800px">

We're now much closer to an "industrial grade" parser, with error messages suitable for human users.
Many of these ideas are adapted from the excellent work of others on libraries for various languages; `commit` or "cut" in particular seems to be indpendently re-invented in the course of almost every parser implementation.

The OBAN grammar (`oban.ebnf`) and updated parser code (`oban_pc_improved.py`), as well as the previous material, can be viewed at <a href="https://gist.github.com/derrickturk/7ead182081f2ab1b09225fd0c11dbda9">https://gist.github.com/derrickturk/7ead182081f2ab1b09225fd0c11dbda9</a>.

[^1]: Just as before, we will need some extra object "wrappers" compared to this "ideal" parse tree, due to limitations in `mypy`'s ability to handle recursive type aliases.
