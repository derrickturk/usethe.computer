% Inadvisable Unit Shenanigans

Today's post is less of a "post" and more of an embedded (forgive me) notebook. This'll be the last entry from the Python junk drawer.

This was done on a lark back in 2019 as part of a private discussion group. I wanted to show off a few different ways to embed unit (as in "units of measure": furlongs per fortnight and such) checking into a program. The particular way shown here was my favorite because a) it is one of the few useful uses of metaclasses in Python I've ever made, and b) it embeds dimensional analysis into Python's dynamic type system, which is great fun for any mechanical engineer.

It's overwrought, hard to type check, confusing to read, and probably doesn't meaningfully catch errors a human would actually make. Nonetheless, I like it.

For further reading, look into the [Frink](https://frinklang.org/) programming language, and play around in type systems more expressive than Python's. (In particular, this would all look a lot cleaner with type-level functions or some equivalent.)

By the way, with regard to that "key insight" buried in the notebook, I wrote this long before that bastard Claude was a twinkle in Dario's eye! If your head doesn't instantly go "hexapodia", we have lived very different lives.

<script src="https://gist.github.com/derrickturk/50ec3ebeec6993cc980c8aa613b3fc5b.js?file=annotated_dimensional_system.ipynb"></script>
