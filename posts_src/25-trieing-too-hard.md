% Trie-ing Too Hard With 297 and 298 Files

Today we continue rummaging through my junk drawer and find yet another exercise in parsing oil and gas data formats from the early post-COVIDian period. It's been humbling, as I write these articles, to realize just how ephemeral Internet content has become. High-quality reference material, already in limited supply with the rise of content marketing and SEO, is quickly becoming the low-background steel of the LLM era.
The reference material I used at the time no longer exists; search results point only to the artifact I myself created.
My memory is hazy, so this one will have a little less then the usual level of technical detail.

## Saturday Night's Alright (For PI/Dwighting)

Anyway, what I'm trying to say is this: once upon a time, there was a family of data formats called 297 and 298. These were "owned", plus or minus, by IHS by virtue of having acquired the old PI/Dwight's, the PI portion of which had originated these formats as the 97 and 98 formats. The 97 format was for well header data, more or less, and the 98 format was for production data. Each came in a comma-separated (297c, 298c) and fixed-width field (297f, 298f) version. It doesn't seem that there is any reference material left on the Internet for these except for the project this article describes, so I will share [what I have archived (and which I used to develop this project)](/assets/ihs29x.pdf).

These formats used to be commonly encountered when downloading public well header or production data, or transferring it to or from various oil and gas software applications. It seems that the data vendors have finally moved their users mostly over to web-based APIs, or at least to zipped Excel files, and so perhaps these formats finally are obscure and obsolete. Well, nothing is ever really gone forever in the oilfield, so I present this in the hope that someone somewhere will find it useful.

Back in 2023, according to my scant notes, I'd "been sitting on the spec for a long time now and finally decided to do something about it". (I think there may have been a client with some `.97c` and `.98c` files they wanted to automate the processing of.) There were a few code snippets around GitHub and elsewhere but nobody had a complete parser (and as far as I can tell, even those partial efforts are now completely gone from the Internet).

If you looked at the specification PDF, you probably saw that it was full of tables. Both the 297 and 298 formats consist of series of records, one per line. Each record contains fields—either separated by commas or at fixed character positions within the line—and the first of these, for every kind of record, is a "record type indicator" describing what kind of data to expect from the rest of the fields. The tables in the spec define the expected field data types and positions for each record type. For example, in the 298 format, record type '+A' is an "Entity Record" describing a producing entity (i.e. a well, usually) with the following fields:

| Field Description     | Type         | Position | Length |
|-----------------------|--------------|----------|--------|
| Record Type Indicator | alphanumeric | 1        | 3      |
| Region Code           | alphanumeric | 4        | 2      |
| State Code | alphanumeric | 6 | 2 |
| Field Code | alphanumeric | 8 | 6 |
| County/Parish Code | alphanumeric | 14 | 3 |
| County/Parish Name | alphanumeric | 17 | 8 |
| Operator Code | alphanumeric | 25 | 8 |
| Primary Product Code | alphanumeric | 33 | 1 |
| Mode | alphanumeric | 34 | 1 |
| Formation Code | alphanumeric | 35 | 8 |
| AAPG Basin Code | alphanumeric | 43 | 3 |
| Coal Bed Methane Indicator | alphanumeric | 46 | 1 |
| Enhanced Recovery Flag | alphanumeric | 47 | 1 |
| Blank | alphanumeric | 48 | 32 |

This regular structure and tabular metadata opened up the attractive possibility of building a data-driven parser or even more exotic metaprogramming applications, but first I'd need the tables in some kind of structured representation. I'm told that "AI" can do this sort of thing now, but at the time I desperately just wanted to get things out of the PDF format, so I reached for the free and open-source [Tabula](https://tabula.technology/) application. Tabula got me a reasonable first pass and a little cleanup in Vim got me reasonably clean text tables. The next question was how to best represent the specs in a machine-readable way. I decided it was a good fit for the [Dhall](https://dhall-lang.org/) configuration language. Dhall is a strange beast but one of my favorite projects in the briefly-hot sub-Turing configuration language space. ("The what?" I'll get there.)

## Turing-completeness is Overrated

So: have you ever noticed that configuration files can get a little repetitive? Maybe you've thought about templating that JSON or YAML or TOML or INI or what-have-you so that you don't have to copy and paste the same data over and over with minor edits? Maybe you've even thought about using a Python or Perl or Lua script as a configuration file, but stopped short because you thought it was probably inappropriate for a "config file" to run arbitrary code? During what I think of as the most recent programming language renaissance (peaking around 2015-ish?), several projects attempted to address this dilemma by building "configuration languages": programming languages limited in some ways to allow iteration and abstraction without the full power (or Internet access!) of a general-purpose programming language. In particular, several of these config languages were not [Turing-complete](https://en.wikipedia.org/wiki/Turing_completeness): there are programs that cannot be written in them; most importantly, infinite loops are not possible and all iteration is bounded. Removing the ability to write an infinite loop limits the possibility of a configuration file accidentally causing a denial-of-service attack on the system it configures. Removing access to things like arbitrary I/O or syscalls prevents config files from having unexpected "side effects", malicious or otherwise.

[Starlark](https://starlark-lang.org/) was a cut-down Python, [CUE](https://cuelang.org/) was... whatever the hell CUE was, but for a Haskell programmer and dependent-type dabbler, Dhall was pretty obviously where the ball was going. It's a simple functional programming language with type inference and core data types corresponding to what the JSON world thinks of as objects, arrays, strings, numbers, and booleans (plus some other helpful goodies). More importantly, it leaves out recursion! ([It's possible to translate](https://docs.dhall-lang.org/howtos/How-to-translate-recursive-code-to-Dhall.html) recursion that is equivalent to a "fold" over a finite structure into Dhall, but not unbounded recursion.) Dhall also has the nice pragmatic property that it is easy to translate to JSON, which makes Dhall "code" (or "data": same thing, really) easy to consume from most "stacks".

When I write down data in Dhall, I like to begin by defining types. Strictly speaking, this is often optional: Dhall will happily infer types for typical JSON-like structures. However, I like to catch errors early, so typically I'll end up writing things like:
```dhall
let ConfigType = ...
let Config: ConfigType = ...
in Config
```

To that end, I translated the shared structure of the 297 and 298 formats into a common specification, making use of Dhall's (tagged) union types (i.e. sum types). Disclaimer: in the real project these definitions are spread across multiple files—the code that follows is edited slightly for that reason. Here's the Dhall definition of a `FileFormat`, which specifies the header fields and record types for a given IHS file format (either 297 or 298). We use a union to describe the possible data types and formats for a column (the "Type" field in the IHS format specification tables).

```dhall
let FieldType =
  < Alphanumeric
  | Numeric
  | DateYYYYSlashMMSlashDD -- (YYYY/MM/DD)
  | DateYYYYMMDD -- (YYYYMMDD)
  | MonthYYYYMM -- (YYYYMM)
  | YearYYYY -- (YYYY)
  | G2 -- (±dd.mm.ss)
  | G3 -- (±ddd.mm.ss)
  | Literal: Text
  >

let Field = 
  { description: Text
  , type: FieldType
  , position: Natural
  , length: Natural
  }

let Record =
  { indicator: Text
  , description: Text
  , fields: List Field
  }

in
  { header: List Field
  , records: List Record
  }
```

To complete the specifications of the 297 and 298 formats, I translated my Tabula-extracted tables into Dhall `FileFormat` records using Vim macros and a little manual cleanup. These are long and make dull reading (that's the point!), but here's enough of `TwoNinetySeven.dhall` to get the idea.

```dhall
let FieldType = ./FieldType.dhall
let Field = ./Field.dhall
let Record= ./Record.dhall
let FileFormat = ./FileFormat.dhall

let FileHeader: List Field =
  [ { description = "Record Key"
    , type = FieldType.Alphanumeric
    , position = 1
    , length = 20
    }
  , { description = "Data Type"
    , type = FieldType.Literal "US WELL DATA"
    , position = 21
    , length = 20
    }
  , { description = "Download Format"
    , type = FieldType.Literal "297"
    , position = 41
    , length = 12
    }
  , { description = "Version"
    , type = FieldType.Alphanumeric
    , position = 53
    , length = 4
    }
  , { description = "Delimiter"
    , type = FieldType.Alphanumeric
    , position = 57
    , length = 7
    }
  , { description = "Write Date"
    , type = FieldType.DateYYYYSlashMMSlashDD
    , position = 64
    , length = 10
    }
  , { description = "Entity Count"
    , type = FieldType.Numeric
    , position = 74
    , length = 6
    }
  ]

let StartLabel: Record =
  { indicator = "START_US_WELL"
  , description = "Start Record Label"
  , fields =
      [ { description = "START_US_WELL"
        , type = FieldType.Literal "START_US_WELL"
        , position = 1
        , length = 30
        }
      , { description = "UWI"
        , type = FieldType.Alphanumeric
        , position = 31
        , length = 20
        }
      ]
  }

let EndLabel: Record =
  { indicator = "END_US_WELL"
  , description = "End Record Label"
  , fields =
      [ { description = "END_US_WELL"
        , type = FieldType.Literal "END_US_WELL"
        , position = 1
        , length = 30
        }
      , { description = "UWI"
        , type = FieldType.Alphanumeric
        , position = 31
        , length = 20
        }
      ]
  }

let A: Record =
  { indicator = "A"
  , description = "General Information"
  , fields =
      [ { description = "Record Type Indicator"
        , type = FieldType.Literal "A"
        , position = 1
        , length = 1
        }
      , { description = "API Number"
        , type = FieldType.Alphanumeric
        , position = 2
        , length = 14
        }
      , { description = "Latitude"
        , type = FieldType.Numeric
        , position = 16
        , length = 9
        }
      , { description = "Longitude"
        , type = FieldType.Numeric
        , position = 25
        , length = 10
        }
      , { description = "Formation at Total Depth"
        , type = FieldType.Alphanumeric
        , position = 35
        , length = 8
        }
      , { description = "Producing Formation"
        , type = FieldType.Alphanumeric
        , position = 43
        , length = 8
        }
      , { description = "Initial Well Class"
        , type = FieldType.Alphanumeric
        , position = 51
        , length = 1
        }
      , { description = "Final Well Class"
        , type = FieldType.Alphanumeric
        , position = 52
        , length = 1
        }
      , { description = "Well Status"
        , type = FieldType.Alphanumeric
        , position = 53
        , length = 6
        }
      , { description = "Elevation"
        , type = FieldType.Numeric
        , position = 59
        , length = 5
        }
      , { description = "Elevation Reference"
        , type = FieldType.Alphanumeric
        , position = 64
        , length = 2
        }
      , { description = "Total Depth"
        , type = FieldType.Numeric
        , position = 66
        , length = 5
        }
      , { description = "Completion Date"
        , type = FieldType.DateYYYYMMDD
        , position = 71
        , length = 8
        }
      , { description = "Lat/Long Source"
        , type = FieldType.Alphanumeric
        , position = 79
        , length = 1
        }
      ]
  }

...

let TwoNinetySeven: FileFormat =
  { header = FileHeader
  , records =
      [ StartLabel
      , EndLabel
      , A
      ...
      ]
  }

in TwoNinetySeven
```

Just imagine 66 more record types like `A`, and then the same thing for the 21 record types in `TwoNinetyEight.dhall`. Now that we've got structured data describing the very regular layout of these data formats, what can we do with it?

## I T→h→ink T→h→at I S→hall Never S→ee A Poem A→s Lovely A→s A T→rie

(I know, that was a [radix tree](https://en.wikipedia.org/wiki/Radix_tree), sue me.)

The main thing we can do with it is parse 297 and 298 files! With a straightforward and ("little-R"—don't go reaching for regexes!) regular grammar like this, we can at least partially derive the parser from the spec.

There are few ways to get there, but what I ended up doing is writing code to parse each data type found in the formats (dates, numbers, etc.) as well as handle fixed and comma-separated record formats (if I remember correctly, I had to guess at the quoting rules and assumed it acted roughly like CSV).

Each record can then be parsed using the correct data-type-specific value parsers and appropriate record-splitting logic (fixed-width or comma-separated); what remains is to determine the expected data types for each value in the record from the record type indicator at the beginning of the line.

For each format, using the Dhall specification (translated automatically to JSON via the Dhall tooling, so that it could be consumed from Python^[Why Python? Because that's where the oil and gas technical audience is, for better or worse.]  without additional dependencies), I generated a [trie](https://en.wikipedia.org/wiki/Trie) whose keys were record type indicators and whose values were the record specifications.

The name makes me a little [sic] but tries are a great data structure from the "B list". They're not very well known among self-taught programmers, and that's too bad. If you've ever looked at autocomplete suggestions while you write a text message, the idea is similar (and tries are a likely piece of the implementation, at least for simple autocompletion systems). We construct a tree whose nodes store a single-character label, optionally a value, and an arbitrary number of links to nodes whose labels are suffixes of the node's own label. This gives us an efficient way to find record descriptors by type indicator when the type indicator itself is of unknown length. (That's also the answer to "why not just have a lookup table by full type descriptor?"—some descriptors end at position 2 and the record immediately rolls into other data; others are as long as 13 characters.)

What that record type indicator trie looks like for the 297 format spec is this:
```
<START>
┣━S
┃ ┣━T
┃ ┃ ┣━A
┃ ┃ ┃ ┣━R
┃ ┃ ┃ ┃ ┣━T
┃ ┃ ┃ ┃ ┃ ┣━_
┃ ┃ ┃ ┃ ┃ ┃ ┣━U
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━S
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━_
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━W
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━E
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━L
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━L: Start Record Label
┃ ┣━1: Actual Bottom Hole Reference Location
┃ ┣━2: Actual Bottom Hole Reference Narrative
┣━E
┃ ┣━N
┃ ┃ ┣━D
┃ ┃ ┃ ┣━_
┃ ┃ ┃ ┃ ┣━U
┃ ┃ ┃ ┃ ┃ ┣━S
┃ ┃ ┃ ┃ ┃ ┃ ┣━_
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━W
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━E
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━L
┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┃ ┣━L: End Record Label
┃ ┣━T: Formation Tops
┃ ┣━B: Formation Bottoms
┣━A: General Information
┣━B
┃ ┣━F: Footage Location
┃ ┣━C: Congress and Carter Location
┃ ┣━T: Texas Location
┃ ┣━N: Northeast and Ohio Location
┃ ┣━O: Offshore Location
┃ ┣━M: Location from Monument
┣━C: Operator Information
┣━D
┃ ┣━A: Miscellaneous General Information
┃ ┣━B: Additional Miscellaneous General Information
┃ ┣━C: Additional Miscellaneous General Information (Permit Filer)
┣━F
┃ ┣━<END>: Initial Potential
┃ ┣━A: IP Treat
┃ ┣━D: Detailed Perforations
┃ ┣━N: IP Narrative
┣━G
┃ ┣━<END>: Production Test
┃ ┣━A: PDT Treat
┃ ┣━D: PDT Perforations
┃ ┣━N: PDT Narrative
┣━H
┃ ┣━<END>: Drill Stem Tests
┃ ┣━A: Drill Stem Tests, Pipe Recovery Detail
┃ ┣━B: Drill Stem Tests, Material to Surface Detail
┃ ┣━F: Drill Stem Tests, Flow Period
┃ ┣━N: Drill Stem Test Narrative
┣━I: Core Data
┃ ┣━D: Core Depth/Interval Data
┃ ┣━N: Core Narrative Data
┣━J: Logs Data
┣━K: Mud Data
┣━L: Casing Data
┣━M: Liner Data
┣━N: Tubing Data
┣━O
┃ ┣━N: Location Narrative
┃ ┣━A: Drilling Narrative
┣━P
┃ ┣━F: Proposed Bottom Hole Location (Footage)
┃ ┣━C: Proposed Bottom Hole Location (Congressional and Carter)
┃ ┣━T: Proposed Bottom Hole Location (Texas)
┃ ┣━N: Proposed Bottom Hole Location (Northeast and Ohio)
┃ ┣━O: Proposed Bottom Hole Location (Offshore)
┣━Q
┃ ┣━F: Actual Bottom Hole Location (Footage)
┃ ┣━C: Actual Bottom Hole Location (Congressional and Carter)
┃ ┣━T: Actual Bottom Hole Location (Texas)
┃ ┣━N: Actual Bottom Hole Location (Northeast and Ohio)
┃ ┣━O: Actual Bottom Hole Location (Offshore)
┣━R
┃ ┣━1: Proposed Bottom Hole Reference Location
┃ ┣━2: Proposed Bottom Hole Reference Narrative
┣━T: Deviation Survey
┣━U
┃ ┣━1: Deviation Survey - Run Survey/Survey Level
┃ ┣━2: Deviation Survey - Point Data
┣━V
┃ ┣━1: Horizontal General Information
┃ ┣━2: Horizontal Directional Survey Data
┃ ┣━3
┃ ┃ ┣━F: Horizontal Kickoff Point Footage Location
┃ ┃ ┣━C: Horizontal Kickoff Point Congressional and Carter Location
┃ ┃ ┣━T: Horizontal Kickoff Point Texas Location
┃ ┃ ┣━N: Horizontal Kickoff Point Northeast and Ohio Location
┃ ┃ ┣━O: Horizontal Kickoff Point Offshore Location
┃ ┣━4
┃ ┃ ┣━F: Horizontal Point of Entry Footage Location
┃ ┃ ┣━C: Horizontal Point of Entry Congressional and Carter Location
┃ ┃ ┣━T: Horizontal Point of Entry Texas Location
┃ ┃ ┣━N: Horizontal Point of Entry Northeast and Ohio Location
┃ ┃ ┣━O: Horizontal Point of Entry Offshore Location
┃ ┣━5: Horizontal Kickoff Point/Point of Entry Information Narrative
┃ ┣━6: Horizontal Spoke Length/Terminus
```

We wire up a trie lookup to the aforementioned value-parsing and record-splitting bits and we've got parser for 297c, 297f, 298c, and 298f formats all with a common codebase, driven purely by structured metadata in a type-checked configuration language.

---

You can find the Dhall specification and a simple example parser at [https://github.com/derrickturk/twoninetyex/](https://github.com/derrickturk/twoninetyex/). A "proper" Python package is [available on PyPI](https://pypi.org/project/ihs29x/) and at [https://github.com/derrickturk/ihs29x-python](https://github.com/derrickturk/ihs29x-python). Both are available under the MIT license.
