## Dashboard Confessional

Many clients, friends, acquaintances, and (in the Before Times) strangers in bars have asked me over the years: what do I think about "business intelligence" (BI) tools?
These are applications which make it easy, without any custom programming required[^1], to connect to data sources, visualize data, and create interactive analytics and "dashboards".

Among my clients and in the oil and gas industry in general, the most popular seem to be Spotfire (from TIBCO), Tableau (from... Tableau?), and PowerBI (from Microsoft).
I've personally worked most with Spotfire, but I've watched clients use all three and have seen the feature set approximately converge over the years.

[^1]: As a particularly tedious professor might say, "this turns out not to be true in practice".

### A Spotfire Upon The Deep

I think these tools do a great job of making tabular data easy to access, filter, and visualize for end-users who are not programmers (EUNP, henceforth).
They generally adopt a relational model, even when the underlying data may be from an "unstructured" store like Excel or a "NoSQL" database.
This both helps users with an understanding of SQL easily adapt to the environment, and trains unfamiliar users in the key ideas of relational database design, which is a nice bonus. 

The real power, compared to the spreadsheets which most EUNP are more familiar with, is the combination of dynamic click-and-drag filtering of data (e.g. by "brushing" in the UI) with an implicit relational schema to result in highly dynamic visualizations.
As an example, with a correctly constructed dashboard for grocery store sales, I can click the state of Montana in a map, click "Dairy" on a per-aisle sales volume bar chart, and see month over month sales of dairy in Montana on a time series line chart.
These "dashboards" are extremely popular, especially as a way to condense high-dimensional information into a interactive narrative which can be used to communicate to peers and managers.

So far, so good!
As we should all know by now, there are "lies, damned lies, and statistics"---so it's important to beware all of the usual caveats about honest and meaningful data visualization.
I don't have anything to say about this which many others have not already said more effectively (if you want a place to start, please read---on paper, with the pictures!---Edward Tufte's <a href="https://www.edwardtufte.com/tufte/books_vdqi">The Visual Display of Quantitative Information</a>), so instead I'd like to take a programmer's perspective of where things break down.

### Low Code? No, Code
The real trouble starts when we realize that the entire idea of an EUNP is something of a fiction!
Excel itself is, strictly speaking, a programming environment, even if most of its users don't realize it and would never think of themselves as "programmers".
BI tools generally include a "formula language" or "expression language" but these fall short of even the power and expressiveness available through spreadsheet formulas, much less "real programming languages".  
In fact, these languages generally provide power at best equal to SQL queries[^2] with `group by`.

Inevitably, BI users begin to want features which can only be delivered by a bit of custom programming.
This can range from implementing prediction or scoring on-the-fly from a previously trained machine learning model, to building guided workflows within a dashboard, to implementing entire "CRUD"[^3] applications inside the confines of the BI tool.

The farther down this continuum we move toward "BI tool as application platform", the more pain we experience.
Each BI tool provides its own suite of extension points which allow it to interface to common data science programming environments: usually, R and Python.

PowerBI adopts what seems to me the best balance between utility and conservatism.
In PowerBI, it's possible to send data to an external script written in R or Python (using a locally installed interpreter).
Data is sent as an appropriate "dataframe" (either using base R `data.frame`s or `pandas` dataframes for Python) to the script, which may either return updated or new data as a dataframe or generate plots using R graphics or Python's `matplotlib`.

However, PowerBI doesn't provide much capability for custom workflow scripting---just buttons which can run canned actions (e.g. "navigate to page 3").
This limitation, while I'm certain it annoys some "power users", is really for the best.
These tools simply aren't suited for complex application logic, as I'll elaborate on when we discuss Spotfire.

Tableau's scripting abilities are significantly more limited.
Add-ons are available to allow Tableau to "shell out" to an installed Python or R interpreter (over an HTTP API) and run "scripts".
However, these scripts are written in Tableau as string arguments to a "run script" formula.
If you remember writing "one-liners" in Perl (the agony and the ecstasy), you know that this will not be a sustainable model for "software development".
It may suffice for simple applications such as "call a prediction function to apply a trained model to this row", but any complex logic will become an unmaintainable mess quickly. 

Spotfire provides the strongest suite of tools for use as an ersatz application platform.
As a consequence, I've had the most hands-on experience with it---and learned from the trauma.

"Real programmers" have it hard.
To build a typical small web application, a programmer may need to know 3 or 4 languages: HTML, CSS, and Javascript for the front-end; and, probably, another language (Python? Ruby? Rust?) for the back-end, although they may get away with Javascript here as well.
Luckily, we have wonderful "low code" tools to ease this burden---right?

Let's consider building an "application" in Spotfire which provides the typical "CRUD" operations on a database while also allowing the user to do some simple analytics on-the-fly (say, applying K-means clustering).
We'll start by connecting to our database to fetch data---that's easy and well-supported out of the box.
If we want to run custom clustering, the most straightforward way is through an R "data function"---similar to the PowerBI model, Spotfire will arrange for our code to be called with the right data stored in pre-defined variables. 
We can easily send data back, but not graphics.
Our analyst will need to understand R well enough to apply the required functions to the incoming data structures and ensure the result is structured appropriately.

We might be happy with a built-in visualization, but perhaps we find that we want something a little different.
In that case, we may take advantage of the fact that the Spotfire UI runs as a web page in an embedded browser.
We can use Javascript libraries like `d3.js` to render custom visualizations---our analyst will just need to know HTML, CSS, and Javascript to pull it off.

To stitch all this together, we may need some custom application logic.
In particular, we don't have a way "out of the box" to write results back to our database.
Luckily, Spotfire exposes a very extensive API for scripting.
This is provided in the form of an embedded IronPython interpreter.
IronPython is a "living fossil"---it's an almost-compatible implementation of Python 2.7, but running on the .NET CLR.
Our analyst will need to know Python 2 (no longer supported in the "mainstream" Python community), but also enough about the .NET runtime, type system, and libraries to successfully engage with the API (let's round that up to "knows C#", because they're effectively equivalent).

In any case, with IronPython in hand we can use the .NET libraries for database access to send data back to the database---either as raw `insert`/`update` statements or by invoking stored procedures.
Of course, this will require knowledge of SQL (and perhaps of our database's specific dialect for stored procedures).

That takes us to 6-and-a-half languages which our analyst may need to know, in some depth, to successfully build the desired "low-code" application.

This is *not* to knock Spotfire or tools like it---they provide a very impressive amount of flexibility and capability.
However, I strongly feel that trying to shoehorn the "square peg" of a logic-dense application into the "round hole" of a BI tool is a recipe for pain.
We quickly reach a point where developing the application within the constraints of the BI tool is more difficult than building it "from scratch" with more traditional software development tools.
We do save time by taking advantage of built-in data filtering and visualization, but we also expose ourselves to "un-debuggable" bugs inherited from the platform (and I assure you these exist, and they are many---I've had more than one bugfix from a vendor as a consequence of investigating after a client hit a "weird error").

[^2]: Excluding recursive CTEs---unlike "modern" SQL, these expression languages are not Turing complete. 
[^3]: Create, Read, Update, Delete---doesn't this sound like every data application ever?

### Separation of Church and State
How can we get "the best of both worlds"?
I suggest an old tongue-in-cheek programmer's maxim: we need to enforce the separation of <a href="https://en.wikipedia.org/wiki/Alonzo_Church">"Church"</a> and <a href="https://en.wikipedia.org/wiki/State_(computer_science)">state</a>.
This is commonly expressed more plainly as "imperative shell, functional core"; it's also related to the MVC design pattern.

What I propose is that we use the BI tools for what they're good at: accessing relational data and interactively filtering and visualizing it.
These tools adequately maintain their own application state, but don't make good "databases" themselves for things like model results.

Our analytics---the interesting math and logic we want to apply to our data---should be designed to be as purely functional as possible.
That is, we should strive to expose these systems as sets of functions which operate only on their inputs and produce desired outputs without any "side effects" (e.g. changing global state, writing data to file, "launching the missiles").
We can expose these to our BI tool in an appropriate way (via web API, via embedded script, etc.) so that they can be invoked on-the-fly on selected subsets of data.

Stateful data management should be outsourced to our database systems; leaning on the features of modern RBDMSs, we can constrain and validate our data at the point of insertion and provide "APIs" for CRUD operations via SQL queries and stored procedures.
The BI tool should only need some minimal "glue code" to use these interfaces.

Finally, we need to always remain aware of the hidden costs of building on BI-based platforms, and be ready to consider switching to a "proper" application development model when our project's complexity exceeds a certain threshold.
