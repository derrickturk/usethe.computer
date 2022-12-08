% Zipping Python Trees

The following appeared, as it were, by chance on a private Slack channel as a tangent to a discussion involving this year's Advent of Code. I have not cleaned it up for the insufferable and joyless pedants of various news aggregation sites, orange or otherwise; nor do I plan to. Life is simply too short. Therefore I apologize in advance for neglecting:

* the rich history of the zipper concept  
* that one cringy thing about Theseus  
* the obvious superiority of lenses  
* the obvious inferiority of lenses  
* the (don't giggle) phrase "one-hole context"  
* the joys of performing "type calculus" to derive zippers as per the above  
* continuation-based "zippers"  
* the utility (or not) of pure functional programming  
* the futility of doing this in Python in the first place  
* the pain of dealing with chained operations on `T` | `None` in Python
* why I didn't "just" do it in XSLT, or on a SPARCstation, or...  

I present it here with minor alterations only, in the interest of introducing an elegant and useful trick from the functional programming world to a generation of programmers raised on Python.

### Transmission Follows
Language path: Haskell→Ocaml→Python, mypy typesystem [semantic loss ~31%]  
Subject: Zippers as the key insight  

What great fun! Since mypy 0.991 (or so?) we can finally, finally work with recursive types.
This, combined with the pattern-matching features of Python 3.10, opens up "typed" Python to some traditional functional programming practices with a minimum of BS.

Let's walk amongst the trees for a bit.
We can represent binary trees as nested tuples.
A tree is either:  

* the empty tree (represented by `None`), or
* a branch with a data element (here, an `int`) and left and right children (which are trees)

```python
from typing import NamedTuple, TypeAlias

class Branch(NamedTuple):
    value: int
    left: 'Tree'
    right: 'Tree'

Tree: TypeAlias = Branch | None
```

For example:

```python
example_tree: Tree = Branch(3,
  Branch(2, Branch(1, None, None), None),
  Branch(5, Branch(4, None, None), Branch(6, None, None))
)
```

At the REPL:
```
>>> example_tree
Branch(value=3, left=Branch(value=2, left=Branch(value=1, left=None, right=None), right=None), right=Branch(value=5, left=Branch(value=4, left=None, right=None), right=Branch(value=6, left=None, right=None)))
```

We can make this easier to understand with some ASCII art:
```python
def pprint(tree: Tree, indent: int = 0) -> None:
    if tree is None:
        print(indent * ' ' + '·')
    else:
        print(indent * ' ' + str(tree.value))
        pprint(tree.left, indent = indent + 4)
        pprint(tree.right, indent = indent + 4)
```

At the REPL:
```
>>> pprint(example_tree)
3
    2
        1
            ·
            ·
        ·
    5
        4
            ·
            ·
        6
            ·
            ·
```

Now, suppose that for some contrived reason (maybe we're solving a programming puzzle <a href="https://adventofcode.com/2022/day/7">programming puzzle</a>), we have a sequence of commands we want to execute against this tree, where each command may either navigate the tree ("go left", "go right", "go back to the parent") or edit the current position ("set value to 3", "replace left child with None").
In a purely functional program, all data is immutable: we will not be editing the tree in place.
Rather, each command must produce a new copy of the data with the necessary edits applied.

One more thing: for similarly contrived reasons, we care about the final state of the _entire_ tree (in other words, the "root") after all commands are applied.
This makes life harder!
Let's try to write a function for "go left, then go left, then set value to 77."

```python
def left_left_77_first_try(tree: Tree) -> Tree:
    # navigation is pretty straight-forward:
    if tree is None:
        raise ValueError("can't go left!")
    tree = tree.left
    if tree is None:
        raise ValueError("can't go left!")
    tree = tree.left
    if tree is None:
        raise ValueError("can't go left!")

    # hmm - this doesn't feel quite right!
    return tree._replace(value = 77)
```

Let's try it out:
```
>>> pprint(left_left_77_first_try(example_tree))
77
    ·
    ·
```

That's not right at all---we've edited the left, left subtree, but we've "forgotten how we got there".
Remember, we're interested in making edits at the leaves but retaining the whole tree (with those edits).
To make that work, we need to take apart the tree on our way "down" to the destination, and rebuild it with edits on our way back "up".

```python
def left_left_77_second_try(tree: Tree) -> Tree:
    if tree is None:
        raise ValueError("can't go left!")
    first_left = tree.left
    if first_left is None:
        raise ValueError("can't go left!")
    second_left = first_left.left
    if second_left is None:
        raise ValueError("can't go left!")

    # rebuild the tree, making replacements...
    return tree._replace(
      left = first_left._replace(
        left = second_left._replace(value = 77)))
```

And now:
```
>>> pprint(left_left_77_second_try(example_tree))
3
    2
        77
            ·
            ·
        ·
    5
        4
            ·
            ·
        6
            ·
            ·
```

That worked!
Great, that was just a one-off transformation, right? ...Right?


Oh dear.
It seems the actual data is a sequence of these relative commands.
```python
commands = [
    'left', 'left', 'set 77',
    'up', 'set 99',
    'up', 'right', 'right', 'set 33',
]
```

Well, we _could_ translate each sequence of moves followed by a "set" to the appropriate traversal and edit, like `left_left_77_second_try`---but we'll be walking and rebuilding the _whole_ tree, from the root, every time we make an edit.
That clearly won't scale to the thousands of nodes that this toy problem _obviously_ will one day reach.

Let's slow down and think a little.
We've already noticed that when we walk a tree to make edits, we have to remember "the rest of the tree" in order to reconstruct it.
We also have to know "how we got there" to know which parts of the original tree to replace as we work our way back up.

The mechanism is kind of like a zipper opening and closing: we "unzip" the tree apart as we move "down" to the focus of the edit, and "zip" it back together when we're done.
We are not the first to notice this! <a href="https://en.wikipedia.org/wiki/G%C3%A9rard_Huet">Gérard Huet</a>, known also for his work on the Caml programming language (ancestor of OCaml) and the Coq proof assistant, <a href="https://www.cambridge.org/core/journals/journal-of-functional-programming/article/zipper/0C058890B8A9B588F26E6D68CF0CE204">stumbled across an idea he called the "zipper"</a> while working on a structured editor (perhaps for Coq) in 1996.

The zipper concept works for many inductively-defined data types.
The core of the idea is this: represent a "zipper" for a given datatype by the contents of the "focused" node plus enough "context" to recover the rest of the structure given alterations to the focus.
With the right representation for the context, we can make navigation (that is, moving the focus) efficient.
Remember---this is pure functional programming, so the zipper itself is also immutable data; navigation operations will take zippers and return new ones, not mutate the input.
Let's make that all concrete by building a zipper for our binary tree type!

A zipper for a binary tree represents a focused node in the tree, with the additional information required to navigate around and rebuild the whole thing.
The "context" piece of our zipper will consist of a stack of steps we took to reach the focus from the root.
For each step, we need to store enough data to recreate the whole tree around the focus; this means we'll need to store both the direction we took and the "remainder" of the node we navigated from.

First, let's define an enumerated type to describe the possible directions we can take navigating "downward" in a tree:
```python
from enum import Enum, auto

class Direction(Enum):
    Left = auto()
    Right = auto()
```

For each step we take, we'll want to store the direction we went, the value stored at the branch, and the "other" subtree (that is, if we go `Direction.Left`, we need to store the `right` subtree as `other`, and _vice versa_).
```python
class PathStep(NamedTuple):
    dir: Direction
    value: int
    other: Tree
```

Finally, the zipper proper combines a stack of the steps we took to get to the focused subtree, and the focused subtree itself.
I'm going to use a list as a purely-functional stack by copying at each step; this is a little ugly but the goal here is not to recapitulate the definition of immutable lists in Python.
Remember, we don't need `deepcopy` because all the data is immutable!
```python
class Zipper(NamedTuple):
    path: list[PathStep]
    focus: Tree
```

Functions to convert between a root tree and the corresponding root zipper with that tree as the focus are straightforward:
```python
def to_zipper(tree: Tree) -> Zipper:
    return Zipper([], tree)

def from_zipper(zipper: Zipper) -> Tree:
    return zipper.focus
```

We can now build operations on zippers, for navigating the tree.
Going left or right requires remembering the value at the branch, and the other subtree, pushing these onto the path.
These operations can fail if the current focus is a dead-end (`None`), so we'll return `None` to indicate failure to navigate in that case.
```python
def go_left(zipper: Zipper) -> Zipper | None:
    if zipper.focus is None:
        return None
    step = PathStep(Direction.Left, zipper.focus.value, zipper.focus.right)
    return Zipper([step] + zipper.path, zipper.focus.left)

def go_right(zipper: Zipper) -> Zipper | None:
    if zipper.focus is None:
        return None
    step = PathStep(Direction.Right, zipper.focus.value, zipper.focus.left)
    return Zipper([step] + zipper.path, zipper.focus.right)
```

Remember, we store the "road not taken" in the path stack; if we go left, we store the right subtree for later "re-zipping", and if we go right, we store the left subtree for later.
This lets us rebuild the parent branch when we go "up", using the stored subtree for the other fork.
If we're at the root already, we'll just return the current zipper unchanged:
```python
def go_up(zipper: Zipper) -> Zipper:
    match zipper.path:
        case []:
            return zipper
        case [PathStep(Direction.Left, value, right), *rest]:
            return Zipper(rest, Branch(value, zipper.focus, right))
        case [PathStep(Direction.Right, value, left), *rest]:
            return Zipper(rest, Branch(value, left, zipper.focus))
    raise RuntimeError("this can't happen but mypy doesn't know it!")
```

Navigating all the way back to the root (so that we can recover the final tree) is just a matter of recursing "up" until the path stack is empty:
```python
def go_root(zipper: Zipper) -> Zipper:
    if not zipper.path:
        return zipper
    return go_root(go_up(zipper))
```

For convenience, and functional "flavor", we'll provide a higher-order function for "editing" the focus of a zipper as well:
```python
from typing import Callable

def update_focus(zipper: Zipper, f: Callable[[Tree], Tree]) -> Zipper:
    return zipper._replace(focus = f(zipper.focus))
```

Now let's try applying our new zipper to navigate and edit the example tree.
```
>>> left = go_left(to_zipper(example_tree))
>>> print(left)
Zipper(path=[PathStep(dir=<Direction.Left: 1>, value=3, other=Branch(value=5, left=Branch(value=4, left=None, right=None), right=Branch(value=6, left=None, right=None)))], focus=Branch(value=2, left=Branch(value=1, left=None, right=None), right=None))
>>>
>>> left2 = left and go_left(left)
>>> print(left2)
Zipper(path=[PathStep(dir=<Direction.Left: 1>, value=2, other=None), PathStep(dir=<Direction.Left: 1>, value=3, other=Branch(value=5, left=Branch(value=4, left=None, right=None), right=Branch(value=6, left=None, right=None)))], focus=Branch(value=1, left=None, right=None))
>>> print(go_root(left2))
Zipper(path=[], focus=Branch(value=3, left=Branch(value=2, left=Branch(value=1, left=None, right=None), right=None), right=Branch(value=5, left=Branch(value=4, left=None, right=None), right=Branch(value=6, left=None, right=None))))
>>>
>>> # dumb Python trick: if `x: None | T` and `f: Callable[[T], T]`
>>> #   then `x and f(x): None | T`
>>> root_zipper = go_root(
...   update_focus(left2, lambda t: t and t._replace(value = 77)))
>>> new_tree = from_zipper(root_zipper)
>>> pprint(new_tree)
3
    2
        77
            ·
            ·
        ·
    5
        4
            ·
            ·
        6
            ·
            ·
```

Success! We've moved our zipper left twice, performed a functional update of the focused node, moved the zipper back to the root, and extracted the resulting tree.

Let's revisit our command sequence from before.
We'll write a function translating each command to an action updating a zipper; it's more awkward to do this in a purely functional way in Python than by rebinding a variable in a loop, but as a show of "nothing up my sleeve", let's use the `reduce` higher-order function to apply the sequence of commands to a tree.
We'll need to convert the tree to a zipper before we start applying commands; we'll take the final zipper, navigate to the root, and recover the full tree as our output.
```python
def apply_commands(commands: list[str], tree: Tree) -> Tree:
    def apply_command(z: Zipper, c: str) -> Zipper:
        match c.split():
            # we'll be "permissive" of bad commands, and just stay where we
            #   are if navigation fails
            case ['left']:
                return go_left(z) or z
            case ['right']:
                return go_right(z) or z
            case ['up']:
                return go_up(z)
            case ['set', n]:
                return update_focus(z,
                  lambda t: t and t._replace(value = int(n)))
            case _:
                raise ValueError(f'bad command: {c}')

    from functools import reduce
    return from_zipper(go_root(
      reduce(apply_command, commands, to_zipper(tree))))
```

We have now solved the original challenge in a purely-functional way with the aid of our shiny new zipper!
```
>>> pprint(apply_commands(commands, example_tree))
3
    99
        77
            ·
            ·
        ·
    5
        4
            ·
            ·
        33
            ·
            ·
```
