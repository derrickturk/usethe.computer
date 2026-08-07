% If You Get The Message, Hang Up The Phone; or, Whalefall

This post originally ran on Collide and LinkedIn. It's archived here with its original publication date.

Please forgive the little "sales pitch" at the end; believe me, I know there's much more to programming than a little vocabulary. That said, I can't stop clients from jumping into agentic vibe coding, and a little vocabulary will go a long way for them---think of it as "harm reduction".

---

As you probably remember from grade school or the Discovery Channel (don't make me feel old), the noble blue whale represents planet Earth's single most massive concentration of nutritious animal protein and lipids. When said blue whale goes the way of all flesh, the sudden introduction of literal tons of macronutrients and other organic matter into the benthic environment stimulates the birth, evolution, and eventual death of a dense and fast-paced hyper-localized ecosystem. Bottom-dwellers rush in to devour the easy pickings; specialized organisms undergo brief population explosions as they move in to their respective whale-salvage niches. Finally after months or even years, the organic matter is gone entirely, the music stops, and the party ends.

Say, how long do you have access to Claude Fable 5 for?

---

A while back I posted on here asking whether BI dashboard tools like PowerBI or Spotfire were "the new VBA"—that is, whether they would replace various "easy", "low-code", or "no-code" end-user programming tools and environments as rapid application development platforms for non-programmer domain experts like engineers or geologist. Gentle reader, I should have seen this coming: of course they're not. LLM coding agents are.

This isn't an article for the dedicated "vibe coder" nor the software professional. The first group has no choice, and for the second group (I include myself), it's a matter of engineering judgment whether to use these tools and to what extent. But if you're (say) a reservoir engineer with some data and automation problems floating around the back of your mind, and a coding agent subscription burning a hole in your (well, somebody's) pocket, you may be looking for some thoughts on what, and how, to build.

There's a glib rebuttal I see all the time that "these models are as bad as they'll ever be!" I think that's questionable. I remember the days when Google actually worked and StackOverflow was brand new, and I'm sure glad I didn't say anything like that about those tools then. Anyway, what I suspect is that "these models are as cheap as they're going to be for a while".

Accepting _arguendo_ that I'm right on this and the price of especially high-end coding agent access will increase over the next few years, what does a domain expert do to capitalize on their own little whalefall without drowning in slop or hitching a key workflow to a potentially unbounded future increase in LLM usage prices or loss of model quality?

---

> If you get the message, hang up the phone.  
> --Alan Watts  

As Alan Watts pointed out, if you've achieved your desired mystical insight by direct mental communion with transcendental math entities, chaos demons, non-human intelligences, and so forth, it's best to cash in your chips and exit the conversation before your sanity takes a hit. I don't exactly know how he was able to foresee the widespread use of large language models from all the way back in 1970, but I assume the drugs had something to do with it.

For my money, the highest-leverage use of frontier models at current usage prices for the typical domain expert is to generate non-LLM software. Don't build an agent skill to read, say, a PowerBI project, send it off to the LLM, and generate a wall of slop describing the data sources. Ask it to write a project parser that deterministically gives you a data source map with no LLM request. Don't have it transform your data or move it from one place to another, ask it to write a re-usable script that does the ETL. If you get the message, hang up the phone!

Where do I think a domain expert can apply this without help from a more experienced programmer or software engineer? Frankly I'm not entirely confident yet, but here are some "green flags" I'd look for:  
- the workflow is tedious but low-stakes  
- there are either many existing examples of input and output, or the result is deterministically checkable  
- the problem maps cleanly onto a common programming pattern (but there's the rub! how will you know? OK, that's as close as I'll get to a sales pitch)  
- the problem is "one-and-done" with no future maintenance required once an executable artifact is produced  
- it's OK for the artifact to be a "black box" as long as the requisite automated tests pass, etc.  

Don't blindly trust the fickle wee beastie; you'll want protective charms and a solid grounding in at least the basic principles of Fae Law. Make it write tests. Make sure the tests test what you actually care about as acceptance criteria (input/output pairs, properties of the output which must always hold). Make it present a plan before execution and make it ask clarifying questions. Don't be afraid to cancel a trip down the rabbit hole, or answer a yes-or-no question with an "it depends". I don't believe there's any magic to prompting; in fact I think the labs have "just talk to the computer" as their optimization target. I do believe that there are a few dozen computer-sciencey phrases and words that—if used in the correct context—can alleviate a lot of the LLM wiffle-waffle and hit the right target, faster.

But if I just came out and told you those, I imagine you'd hang up the phone!
