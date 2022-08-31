# Aligning Linked Data and the CSVW annotated table

One of CSVW's [stated use
cases](https://www.w3.org/TR/2016/NOTE-csvw-ucr-20160225/#R-AnnotationAndSupplementaryInfo)
was to provide annotations on tabular data, in particular data in the
form of a CSV file.

We believe annotations on tables are an important feature of CSVW
which we want to surface to publishers and users alike. The reasoning
here is that both publishers and consumers are familiar with tables,
so tables and their components (e.g. columns, rows, cells) are a
natural and familiar place to add extra metadata annotations.

The table's structure therefore provides a familiar structure which
users can arbitrarily extend. In practice these extensions mean
annotating various locations within the table (subjects) with pairs of
predicates and objects; and in this way CSVW provides a more familiar
on ramp to the world of linked data.

Similarly we'd like these tables (datasets) to actually be linked
data; that is they should be identified by their locations on the web.
Visiting a "table" would then be dereferencing the table into an
appropriate representation; for example a HTML interface to the
annotated table model, or a `text/csv` representation of the data,
depending on [content negotiation](https://www.w3.org/TR/dwbp/#Conneg).

As we will see, there are many impedance mismatches between CSVW and
linked data, and in-spite of their shared origins and goals they are
hard to align. Consequently it is hard to integrate these two
technologies such that they are both mutually beneficial and aligned.

# What we want to happen

Before we look at the problems, we should first define what outcome we
would like from integrating CSVW with linked data.

Below is a prototype UI for a CSVW interface. It may not be apparent
from looking at it, but this UI demonstrates some interesting
properties that attempt to try and harmonise CSVW and linked data.

Firstly we anticipate that the dataset would be identified by a format
independent `@id`, that is the datasets URI would literally be the
same as the CSV's `@id`, and that visiting it in a browser like below
would return a HTML representation of the data:

![CSVW Preview](./linked-data-csvw.png)

Logically for this to occur, the URI of the resource would be
independent of the CSV file itself, and the `@id` would identify the
annotated `csvw:Table`, which would be an abstraction over the CSVW.

The URI `</data/life-expectancy>` would essentially then provide a
uniform interface to the resource and the representation(s) people
want. For example if you ask for `application/csvm+json` you would be
directed to the metadata file, `text/csv` the CSV, whilst `text/html`
or an RDF serialisation such as `application/n-triples` would combine
the two documents to yield the expected representation.

Similarly it would be extremely desirable for all of the URI's to
align appropriately, such that dereferencing an observation by its
`aboutUrl` would return an appropriate representation of it in
context of the table:

![Row dereferencing](./row-dereferencing.png)

The above feels highly intuitive, and brings the combined benefits of
linked data and CSVW to more typical data users. However it's worth
noting that to make this work, we need to unpick some subtle issues in
the specification and clarify our terms of engagement such that this
can occur.

In particular it's worth noting that there are substantial differences
in CSVW between the RDFization of data and the representation of the
source data in the CSV. In CSVW the RDF outputs of `csv2rdf` are not
typically thought to be tabular, but belong to the world of graphs,
rather than tables. However in the cases of CSVW we'd like to present
the derived graph in terms of the table.

This means that in this view `csvw:Column` definitions in the
`csvw:TableSchema` are used as a lens through which we can view the
projected RDF graph. Is a `csvw:Column` the same as an `rdf:Property`?
No, at least not always, but they are in some cases so closely linked
that for practical purposes it is worth treating them as highly
related, and in the case of dataset specific properties they could for
brevity share the same `@id` and be maintained in the same place.

The UI could for example incorporate affordances for accessing
metadata on the columns themselves:

![Column metadata](./column-metadata.png)

In particular exposing annotations on the `csvw:TableSchema` and
`csvw:Column`s gives us structural locations for publishers and users
to access knowledge in the DSD. For CSVW cubes a tableSchema could
share the same `@id` as the cubes DSD, exploiting this would benefit
maintainance and understanding, and minimise the need to develop whole
new features to handle artificially distinct structures.

I'd like to encourage the view that the same `csvw:TableSchema` is the
most useful lens through which to view the input CSV, the output RDF,
and arbitrary internal stages of processing (such as viewing
validation errors as annotations on what is substantially the same
table). Having publishers and users alike work with the same
[homoiconic representation](https://en.wikipedia.org/wiki/Homoiconicity) is highly
beneficial to understanding, and lets users leverage all
representations simultaneously as extensions within the same model.

In order to do this, and to use the `csvw:TableSchema` as a lens for
viewing the RDF output; there is one small complication, which is that
a csv row may itself yield multiple subjects (`aboutUrl`'s). This can
be solved by `mapcat`/`flatMap`ing over the outputs to remove the
layer of nesting that results. This may in some circumstances result
in one input row becoming several output rows, and may increase the
likelyhood that some columns containing null values.

It's also worth noting that typically for statistical data cubes we
would not expect Tidy data (essentially 3rd normal form) to have
multiple subjects, as that would typically imply a level of
denormalisation.

## Clarifying the Annotated Table model

Reading the specs people are often unclear about what exactly the
annotated table is. It is frequently confused or seen as equivalent to
the RDF, however in reality it is something different.

When you closely read the specifications (principally the core
[tabular data
model](https://www.w3.org/TR/2015/REC-tabular-data-model-20151217/)
specification) it is clear that the annotated table is essentially an
[in-process
representation](https://www.w3.org/TR/2015/REC-tabular-data-model-20151217/#dfn-annotated-table)
of the processing. Essentially it is a specified JSON-like in memory
data structure that results from the merger of the data with the
processing instructions in the metadata document.

As we have seen the annotated table is _not equivalent_ to the
`csv2rdf` standard mode output, as the output there is not by default
annotating much at all.

It's worth noting that the annotated table model contains metadata
such as the URI templates `aboutUrl`, `propertyUrl` and `valueUrl`
even though this core specification does not directly use them. This
was almost certainly done to ensure that all interpretations of the
annotated table expand these templates in the same defined way. This
does pose some problems as the evaluation of these [templates is
"weird"](./001-template-evaluation.md)

Another question; is what table does the annotated table actually
represent? Is it annotating the CSV input, the RDF output or the
processing of the CSV? The only clear answer to this question is that
it is all of the above because the annotated table is procedurally
defined by the algorithms in the specs and updated in place. Subsets
of the schemas and metadata apply at different points, and may take on
multiple roles or meanings. For example `csv2rdf` uses the URI
templates to build triples, but the annotated table model uses them to
help annotate cells, rows and columns depending on your perspective.

It is largely this ambiguity which we hope to exploit in our proposed
CSVW interfaces; and that we should adopt conventions that align our
representations such that they are isomorphic across different
contexts.

# The problems aligning linked data and CSVW

Our proposed interfaces aim to use the annotated table model as the
most complete representation of CSVW, to drive many features and
benefits.

However the above vision is awkward to achieve due to problems
in CSVW's construction. The main claim here is that the standardised
semantics of CSVW's annnotated table model are subtly incompatible and
create a surprising amount of friction when married with the
requirements for linked data dereferencing.

Due to the issues discussed below, I also claim that CSVW as it stands
yields and encourages unharmonised data outcomes. Outcomes which
result in a proliferation of incomplete and inadequate representations
of "the data", rather than the augmented common representation users
expect.

This issue is related also to [issue #1: CSVW's weird evaluation
semantics](./001-template-evaluation.md).

### A proliferation of representations all different and incomplete

Firstly lets look at some examples from the test suite to help explain
this point. We'll use
[test011](https://github.com/w3c/csvw/tree/gh-pages/tests/test011),
firstly we have the simple CSV file:

```csv
GID,On Street,Species,Trim Cycle,Inventory Date
1,ADDISON AV,Celtis australis,Large Tree Routine Prune,10/18/2010
2,EMERSON ST,Liquidambar styraciflua,Large Tree Routine Prune,6/2/2010
```

This CSV file is our first tabulation and representation of the data.
Users only using this CSV file as the interface get an (optional)
header row of metadata, and zero or more rows of data. In this
interpretation all cells are strings and would require out of band
knowledge on how to parse and interpret them properly.

Assuming users are using a compatible parser, the table will look
something like this:

![Input Table](./input-csv.png)

I've purposefully not distinguished between the column headings and
the data here, as that is knowledge CSVW gives us, which we'd
otherwise have to recieve out of band. Strictly speaking if all we
have is the above file, we can't even assume we know the dialect, and
can't guarantee any tabular parsing at all.

Next we have the JSON-LD metadata document:

```json
{
  "@context": ["http://www.w3.org/ns/csvw", {"@language": "en"}],
  "url": "tree-ops.csv",
  "dc:title": "Tree Operations",
  "dcat:keyword": ["tree", "street", "maintenance"],
  "dc:publisher": {
    "schema:name": "Example Municipality",
    "schema:url": {"@id": "http://example.org"}
  },
  "dc:license": {"@id": "http://opendefinition.org/licenses/cc-by/"},
  "dc:modified": {"@value": "2010-12-31", "@type": "xsd:date"},
  "tableSchema": {
    "columns": [{
      "name": "GID",
      "titles": ["GID", "Generic Identifier"],
      "dc:description": "An identifier for the operation on a tree.",
      "datatype": "string",
      "required": true
    }, {
      "name": "on_street",
      "titles": "On Street",
      "dc:description": "The street that the tree is on.",
      "datatype": "string"
    }, {
      "name": "species",
      "titles": "Species",
      "dc:description": "The species of the tree.",
      "datatype": "string"
    }, {
      "name": "trim_cycle",
      "titles": "Trim Cycle",
      "dc:description": "The operation performed on the tree.",
      "datatype": "string"
    }, {
      "name": "inventory_date",
      "titles": "Inventory Date",
      "dc:description": "The date of the operation that was performed.",
      "datatype": {"base": "date", "format": "M/d/yyyy"}
    }],
    "primaryKey": "GID",
    "aboutUrl": "#gid-{GID}"
  }
}
```

There are several interpretations of this document, which yield
slightly different but largely overlapping results:

1. csv2rdf standard-mode (and minimal-mode)
2. A pure JSON-LD interpretation
3. A bespoke (but CSVW intention honoring) interpretation; reading the
   metadata file and the CSV side by side; to construct an in process
   "annotated table". Such a process might provide a preview UI etc,
   or engage in a bespoke transformation etc.

Under csv2rdf standard-mode the `csv2rdf` algorithm
yields our "second table" of sorts, the "annotated table":

```turtle
    [
      a csvw:Table;
      # ... rows ...
      csvw:url <test010.csv>
    ]
```

So here we can see that the annotated table represented by the
outermost delimeters has no `@id`, and is instead identified by a
blank node, represented in the turtle above by the delimiters `[]`.

A blank node semantically should be interpreted as an "existential
variable", i.e. we know something in the world exists, and whilst we
don't know what thing it is we may know some facts about that thing.

This is analagous to a witness of a crime who may tell you some
descriptive characteristics about the perpetrator, but they don't know
who they were.

So without an `@id` we can't identify which table this is; and
logically the annotated table might be different from the CSV file it
is pointing at. Regardless operating under either closed or open world
assumptions, we're forced to treat this as either a different table to
that in the CSV, or as being in a heisenberg like state of
uncertainity where it's potentially the same or different.

Users can always assign an `@id` and set the annotated table's
identifer to be whatever they want. So, one obvious way to align these
representations is to set it to be the same as the CSV file's
location, it's `csvw:url`.

If we do this however we are baking the concrete representation into
our identifier and not allowing for other representations of the
table. For example it would be highly desirable for users pasting a
CSVW backed dataset's URI into a web browser to receive a HTML
interface representation of the dataset when visiting it, and this
pattern prohibits this.

### csvw:Table's aren't really tables!

One might say it is fine for the CSV and the `csvw:Table` to have
different identifiers because a `csvw:Table` *is* really just a
description of the table, whilst the csv file itself *is* the actual
table.

Whilst this distinction between description and table is logical, it
is not particularly helpful for us. The proliferation of identifiers
leads to a proliferation of locations (web resources) and an
artificial separation of concerns which does not benefit users.

Additionally it prohibits many optimisations, for example it is
significantly faster to materialise a HTML representation of a tidy
table of observations if you can take them directly from the CSV file
guided by the metadata document, than it is to atomise them into
triples indexed in a triplestore, and try to reconstitute the input
table from the triples via a SPARQL query.

So the question then becomes, can we align the `csvw:Table`
description with the actual table, such that we can treat them
logically as one and the same thing?

I believe you can do this by hand, but unfortunately not with the
`csv2rdf` specification as it stands.

Lets look at the next problem, and see what `csv2rdf` has done with
our rows:

```turtle
    [
      a csvw:Table;
      csvw:row [
        a csvw:Row;
        csvw:describes [
          :country "AD";
          :name "Andorra"
        ];
        csvw:rownum 1;
        csvw:url <test010.csv#row=2>
      ],  [
        a csvw:Row;
        csvw:describes [
          :country "AF";
          :name "Afghanistan"
        ];
        csvw:rownum 2;
        csvw:url <test010.csv#row=3>
      ],  [
        a csvw:Row;
        csvw:describes [
          :country "AI";
          :name "Anguilla"
        ];
        csvw:rownum 3;
        csvw:url <test010.csv#row=4>
      ],  [
        a csvw:Row;
        csvw:describes [
          :country "AL";
          :name "Albania"
        ];
        csvw:rownum 4;
        csvw:url <test010.csv#row=5>
      ];
      csvw:url <test010.csv>
    ]
```

The diagram below attempts to show what is occuring here; on the right
we have the concrete CSV table representation, on the left is a
representation of our annotated table. To simplify the model I've not
actually shown the annotated table as triples, but have drawn it as a
table, and have glossed over some details (for example csv2rdf
standard mode won't actually emit the column annotations, for that
you'd need to interpret the document again as JSON-LD).

The identifier of the annotated table at this stage is a blank node,
represented in the diagram as `???`.

The annotated tables row's are all connected to the `csvw:Table` via
the `csvw:row` property.

Finally at the bottom of the diagram we show the desired RDF graph
(represented as blobs and lines). Each `csvw:Row` links to the nodes
it constructed via a `csvw:describes` predicate. The 3 representations
we have are shown here:

![Three representations of the data](./three-representations.png)

### csvw:Row's aren't rows!

This is in my mind one of the biggest conceptual problems with CSVW.
If you look at the diagram above you'll realise that what we thought
were rows in the annotated table aren't really rows in our table at
all. They merely serve to connect:

1. rows in the source CSV to the RDF outputs via both the
`csv:url` with an [RFC7111](https://www.rfc-editor.org/rfc/rfc7111)
csv fragment identifier.
2. the generated RDF output to the source row.

Whilst `csv2rdf` provides ways to access the `_row` and `_source` row
in URI templates, the URI templates are only used to influence the
construction of the output; crucially they're not used to facilitate
description of the input; so there is no way within `csv2rdf` to
assign your own `@id`'s to rows in the annotated table model.

However the [csvw
vocabulary](https://www.w3.org/ns/csvw#class-definitions) does
describe them as a generalisation of what we might think of rows:

> A Row represents a horizontal arrangement of cells within a Table.

We can quibble that this definition is really defining a subset of a
Row, but it is sufficient for our purposes, as subsets contain
themselves.

So the vocabulary and its usage in csv2rdf are somewhat conflated
here. One might say it's ok, a row in `csv2rdf` is a proxy to finding
a row, and the generated resource; but structurally that is quite
different to what the `csvw:Row` represents.

So perhaps we can align the `@id` of each `csvw:row` with an RFC7111
`#row` fragment identifier instead? That way at least dereferencing a
`csvw:Row` would yield a representation of the row. The rows would
conceptually have a tautological `?row csvw:url ?row` triple, but we
could easily ignore that.

Well unfortunately, no you can't do that, `csv2rdf` insists that
`csvw:Row`'s are only blank nodes, and provides no affordances for their
identifying them in any other way.

> 4.6.1: In standard mode only, establish a new blank node R which
> represents the current row.

Yes, in the annotated table, your `csvw:Row`'s can never have global
identifiers, they are always and only ever blank nodes.

You can align `aboutUrl` to `_row` or `_sourceRow` but not the
connective `csvw:row` objects.

### Column annotations are ommitted

Oddly the `csv2rdf` output does not include a representation of the
columns though you can obtain one by additionaly interpreting the
metadata document as JSON-LD. In our diagrams above we assume we have
done this.

These columns are defined and declared within the metadata document.

The RDF output (from the URItemplates etc) is an arbitrary other
representation entirely.

It's worth noting that so far we've been working with just CSVW, but
in practice we also want to describe our statistical data as RDF Data
cubes. If we're not careful these cubes would then be a fourth
representation of the same data.

Ideally we would have one abstract "linked data dataset", which
benefitted from all of the annotations, extensions and representations
we are using. We wouldn't have a CSV file, and an annotated table, and
some RDF output with some provenance linked them, but instead would
have one dataset, which was augmented with unified metadata.

## Representation independence

This might be a curious thing to ask for, given that "CSV on the Web"
has CSV in the title, but ideally we want to be independent of
specific data formats. RDF, linked data and the web architecture
itself were all designed to support arbitrary representations, yet
many parts of CSVW's design encourage representation centricity,
making it hard to avoid.

CSVW's annotated table model and vocabulary is largely representation
independent and is intended to work with HTML tables for example.
However many parts of spec, in particular around URI generation result
in representation centricity.

Similarly the specs always assume that a CSV on the web will be
obtained by a GET request; and the specs do not assume a CSV
representation can be obtained through content negotation on an
abstract resource. For example the specs could have specified that
user agents _SHOULD_ set a header of `Accept: text/csv`.

If you're not careful however CSVW will tie you and your identifiers
to concrete representations, rather than abstract resource
identifiers. This poses a problem when we want to publish a dataset on
the web. We're already invested in linked data, so ideally we want to
publish a dataset with a single identifier, and have dereferencing
that identifier not always give you the CSV, but have it give you an
appropriate representation depending on content negotiation.

## @base URI's are weird and broken

This is discussed in the related issue of [template
evaluation](./001-template-evaluation.md). Ultimately the work around
here is to use absolute URIs everywhere, however that is highly
undesirable, as it reduces the portability of the data, and makes it
hard to untether it from a global context.

In particular it also means that data developers cannot defer
decisions about where data will eventually published to whomever does
the publishing. Instead they must coordinate around URI's and ensuring
their locations are agreed in advance.

## A proliferation of unaligned representations without a coherent core model

You can align the identifier for the abstract table resource with the
location of the csv file, so that you are treating them logically as
the same resource.

However when you do this, not only are you baking the format into your
identifiers, but you're not completely aligning the model.

For example you cannot align `csv2rdf`'s `csvw:Row` objects with the
table, because you cannot assign them `@id`'s which align with
locations in the csv, instead you have to have a level of indirection.

Many people have hoped for CSVW to be an on-ramp to linked data,
giving people the benefits of linked data with more familiar tabular
representations. However CSVW largely assumes RDF to be a product of
CSV, rather than


# Solutions

TBD