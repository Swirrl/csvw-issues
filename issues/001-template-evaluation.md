# Weird CSVW Semantics

Take the following simplified example to illustrate the issues here:

```json
{
 "@context": ["http://www.w3.org/ns/csvw", {"@language": "en",
                                            "@base": "http://example.org/expansion"
                                            }],
 "url": "https://gist.githubusercontent.com/RickMoynihan/044cf6e38da6a52acaad4d9597222f02/raw/63041dd6702252ddb7e0dab06120afbe343a262c/expansion.csv",
 "tableSchema": {"columns": [{
                              "name": "id",
                              "suppressOutput":true
                              },
                             {
                              "name": "label",
                              "datatype": "string",
                              "propertyUrl": "rdfs:label"
                              },
                             {"@id": "#parent-node",
                              "rdfs:label": "Links a node to its parent",
                              "name": "parent_id",
                              "propertyUrl": "#parent-node",
                              "valueUrl": "#{parent_id}"
                              }
                             ],
                 "aboutUrl": "#{id}"
                 }
 }
```

With the CSV file:

```csv
id,label,parent_id
root-node,Root Node,
child-node-1,Child node 1,root-node
child-node-2,Child node 2,root-node
grand-child-1,Grand child 1,child-node-2
grand-child-2,Grand child 2,child-node-2
```

This yields the following output from the specified `csv2rdf`
algorithm, I've manually prefixed and simplified the output for
greater clarity:

```turtle
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX csvdata: <https://gist.githubusercontent.com/RickMoynihan/044cf6e38da6a52acaad4d9597222f02/raw/63041dd6702252ddb7e0dab06120afbe343a262c/expansion.csv#>

csvdata:root-node
  rdfs:label "Root Node" .

csvdata:child-node-1
  csvdata:parent-node csvdata:root-node;
  rdfs:label "Child node 1" .

csvdata:child-node-2
  csvdata:parent-node csvdata:root-node;
  rdfs:label "Child node 2" .

csvdata:grand-child-1
  csvdata:parent-node csvdata:child-node-2;
  rdfs:label "Grand child 1" .

csvdata:grand-child-2
  csvdata:parent-node csvdata:child-node-2;
  rdfs:label "Grand child 2" .
```

Can you spot the issue yet? Possibly not... lets try and expose it
further by looking at a fragment of the RDF we get from the metadata
document by interpreting it as JSON-LD:

```turtle
<http://example.org/expansion#parent-node>
    rdfs:label Links a node to its parent@en ;
    csvw:name "parent_id" ;
    csvw:propertyUrl "#parent-node"^^<http://www.w3.org/ns/csvw#uriTemplate> ;
    csvw:valueUrl "#{parent_id}"^^<http://www.w3.org/ns/csvw#uriTemplate> .
```

Does that not strike you as "weird"?? No?! I assure you it is, though
I appreciate it may not appear that way to the uninitiated...

## Ok so, what's the problem? I still don't see it...

The issue is that the relative URIs we identified in our metadata
document don't align properly with those in our URI templates; they're
distinct and we intended them to be the same!

This is most apparent in our `#parent-node` column definition where we
do this:


```json
{"@id": "#parent-node",
 "rdfs:label": "Links a node to its parent",
 "name": "parent_id",
 "propertyUrl": "#parent-node",
 "valueUrl": "#{parent_id}"
}
```

Note the same lexical `#parent-node` string is used for both the `@id`
of the column and the `propertyUrl`. Given both of these are specified
in the same document with the same `@base`, you'd reasonably expect
them to yield the exact same value. However lets have a look at what
we get:

By interpreting the metadata as JSON-LD the `@id` expands as expected
to this: `<http://example.org/expansion#parent-node>`.

Lets now look at what `#parent-node` expands to when used for our
`propertyUrl`. The `csv2rdf` algorithm yields:

```turtle
PREFIX csvdata: <https://gist.githubusercontent.com/RickMoynihan/044cf6e38da6a52acaad4d9597222f02/raw/63041dd6702252ddb7e0dab06120afbe343a262c/expansion.csv#>

csvdata:child-node-1
  csvdata:parent-node csvdata:root-node;
```

Huh!?! `<http://example.org/expansion#parent-node> !=
<https://gist.githubusercontent.com/RickMoynihan/044cf6e38da6a52acaad4d9597222f02/raw/63041dd6702252ddb7e0dab06120afbe343a262c/expansion.csv#parent-node>`!!!

This isn't a bug it's specified behaviour.  The CSVW [tabular metadata spec says this](https://www.w3.org/TR/2015/REC-tabular-metadata-20151217/#uri-template-properties):

```
 The annotation value is the result of:

    1. applying the template against the cell in that column in the row that is currently being processed.
    2. expanding any prefixes as if the value were the name of a common property, as described in section 5.8 Common Properties.
    3. resolving the resulting URL against the base URL of the table url if not null.
```

So prefixes in URI templates are first expanded in the `@context` of
the metadata document; however the final identifier is resolved
relative to the CSV's `url`, not the metadata documents base!

This is to quote Dan Brickley (one of the chairs of the CSVW working
group) "weird".

In our metadata document relative URI's are all interpreted
relative to the base URI of the metadata document; except in the case
of `aboutUrl`,`propertyUrl` or `valueUrl`; where the base URI used for
URI resolution is the `url` of the CSV table!!

> So what? Setting a propertyUrl to the column's @id is weird, why do
> that?!

I appreciate some people might think doing this is distasteful. After
all a CSV column isn't the same as a RDF predicate is it?

Is it not? I think reasonable people can agree to disagree here.
However to explain the rationale I think it's worth digging into the
semantics a little...

This begins to touch a little on the infamous [HttpRange-14
issue](https://en.wikipedia.org/wiki/HTTPRange-14), but a
`csvw:Column` isn't literally a CSV column at all, it's a column
description. And a column description could be the same thing as a
predicate when you consider that a natural relational tabulation is
for properties of the data to be columns.

For dataset local (unshared) properties, I think a column is quite a
natural place to store the CSVW template for building its own
predicate; and that this can simplify dataset construction for
publishers.

Regardless this is just the most direct and easiest place to
demonstrate the problem... The same issue applies to `aboutUrl` and
`valueUrl` templates, and it is on these fields where the problem is
potentially more damaging.

## All ur @base are belong to them

Forgive the inverted [meme
reference](https://www.youtube.com/watch?v=8fvTxv46ano) but the
problem when it comes to `aboutUrl` and `valueUrl` generation is that
the linked data `@base` does not belong to us, it belongs to them!

Let me explain...

In linked data it is standard practice to coin URI's in
domains/namespaces which you control. You can say anything about
anything, but it's polite and makes sense from a data management
perspective to coin identifiers in places you can exert control. This
is fundamentally because a domain gives you not only some verifiable
authority but also because it means people looking up the meaning of a
document can find its definition there. If you don't own the location
on the internet in which you coined your identifiers you can't ensure
people can access a definition of them.

When it comes to CSVW the problem is that the relative identifiers for
`aboutUrl` and `valueUrl` are coined in terms of the CSV's location:

i.e. the triples here exist in the CSV files domain, which could be
somebody elses.  They don't exist in the domain of our annotations:

```turtle
csvdata:child-node-1 csvdata:parent-node csvdata:root-node .
```

## It violates established best practices for publishing data on the web

> Ok, I can see how that's a problem if we're not publishing the CSV
> file, but for our usecase the CSV file author and annotation
> publisher are the same so it's not a problem.

Ahh, but it still is a problem, because of linked data dereferencing!

The issue here is that it's a [best
practice](https://www.w3.org/TR/dwbp/#Conneg) and architectural
principle of not only RDF/Linked-data but also the supporting web
platform that there's a clear distinction between representation and
resource. A resource is a format independent identifier (a URI) which
is also a location on the web (a URL) which when dereferenced returns
a representation of itself via content negotiation.

This means that a user should be able to paste the identifier for a
row in the annotated table model into their web browser and for
example receive a HTMLized view of the table. Whilst a program might
ask for that same table as `text/csv`, or perhaps an RDF
serialisation.

So the problem is that the identifier `csv2rdf` generates is to the
specific CSV representation and not to the resource identifer in the
annotated table model! i.e. in our example this is the identifer for
our first resource:

```
https://gist.githubusercontent.com/RickMoynihan/044cf6e38da6a52acaad4d9597222f02/raw/63041dd6702252ddb7e0dab06120afbe343a262c/expansion.csv#root-node
```

Note, how it includes the representation in its path `expansion.csv`,
users following this link in a web browser will receive the CSV file
and not (as we would like) an annotated HTML representation of it.

> But the spec spec includes ways to [locate the metadata](https://w3c.github.io/csvw/syntax/#locating-metadata)
> document from the CSV file!

It does, but these methods are not part of standard linked data
derefencing behaviour, are not implemented in web browsers (like
content negotiation), and cannot be relied upon as they are only a
"should" in the spec. They also won't work in the case where there are
many annotations of the same CSV.

## Annotated tables but no annotated rows...

> Ok but, the CSV is the input format, the csvm metadata file contains
> transformation instructions, and the `csv2rdf` algorithm is a
> process which preserves the provenance of the original CSV
> representation through the csvw:describes predicate, which relates a
> row to the linked data resource it creates!
>
> You're just misunderstanding the relationship between the various
> pieces!

The mechanics of this is true, lets look at some csv2rdf standard mode
output from the specification to explain:

```turtle
@base <http://example.org/countries.csv> .
@prefix csvw: <http://www.w3.org/ns/csvw#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .

_:d4f8e548-9601-4e41-aadb-09a8bce32625 a csvw:TableGroup ;
  csvw:table [ a csvw:Table ;
    csvw:url <http://example.org/countries.csv> ;
    csvw:row [ a csvw:Row ;
      csvw:rownum "1"^^xsd:integer ;
      csvw:url <#row=2> ;
      csvw:describes _:8228a149-8efe-448d-b15f-8abf92e7bd17
    ], [ a csvw:Row ;
      csvw:rownum "2"^^xsd:integer ;
      csvw:url <#row=3> ;
      csvw:describes _:ec59dcfc-872a-4144-822b-9ad5e2c6149c
    ], [ a csvw:Row ;
      csvw:rownum "3"^^xsd:integer ;
      csvw:url <#row=4> ;
      csvw:describes _:e8f2e8e9-3d02-4bf5-b4f1-4794ba5b52c9
    ]
  ] .

_:8228a149-8efe-448d-b15f-8abf92e7bd17
  <#countryCode> "AD" ;
  <#latitude> "42.5" ;
  <#longitude> "1.6" ;
  <#name> "Andorra" .

_:ec59dcfc-872a-4144-822b-9ad5e2c6149c
  <#countryCode> "AE" ;
  <#latitude> "23.4" ;
  <#longitude> "53.8" ;
  <#name> "United Arab Emirates" .

_:e8f2e8e9-3d02-4bf5-b4f1-4794ba5b52c9
  <#countryCode> "AF" ;
  <#latitude> "33.9" ;
  <#longitude> "67.7" ;
  <#name> "Afghanistan" .
```

NOTE here that there is first the representation of the annotated
table itself, and then there are the triples output from the
RDFization process, by applying the URI templates. The representation
of the annotated table is



Unfortunately the specification doesn't include the actual metadata
file which yields this result, however there are some [similar
examples](https://github.com/w3c/csvw/blob/0f3a1fde0b8851692150f1862f56e0eead111560/tests/test101-metadata.json)
in the CSVW test suite which we can massage into the right shape:

```json
{
  "@context": "http://www.w3.org/ns/csvw",
  "url": "countries.csv",
  "tableSchema": {
      "columns": [{
        "name": "countryCode",
        "titles": "countryCode",
        "datatype": "string",
        "propertyUrl": "http://www.geonames.org/ontology{#_name}"
      }, {
        "name": "latitude",
        "titles": "latitude",
        "datatype": "number"
      }, {
        "name": "longitude",
        "titles": "longitude",
        "datatype": "number"
      }, {
        "name": "name",
        "titles": "name",
        "datatype": "string"
      }],
      "aboutUrl": "{#countryCode}",
      "propertyUrl": "http://schema.org/{_name}"
  }
}
```

So here we can see that the table represented by the outermost
delimeters has no `@id`; therefore under JSON-LD this is treated as a
blank node, represented in the turtle above by the delimters `[]`.

```
  csvw:table [ ... ]
```

A blank node semantically should be interpreted as an "existential
variable", i.e. we know something in the world exists, and whilst we
don't know what thing it is we may know some facts about that thing.

This is analagous to a witness of a crime who may tell you some
descriptive characteristics about the perpetrator, but they don't know
who they were. Logically this derives from the underlying logic behind
RDF's open-world-assumption.

Without an `@id` though under RDF semantics we need to assume in these
cases that there are possibly two tables; the CSV itself and the
"annotated table".

So by default CSVW's annotated table, the abstract model of the CSV,
doesn't tell us which table it is. However it affords you the
flexibility to assign your own `@id` to the logical table should you
wish, e.g. we could add the line `@id` to the above json-ld, setting
the `@id` to be the same as the `@base`:

```
{
 "@context": ["http://www.w3.org/ns/csvw", {"@base": "http://www.swirrl.com/csvw/example/countries"}],
  "@id": ""
  ...
}

```

This is the right starting point for our abstract annotated table. It
would give us something like this:

```
<http://www.swirrl.com/csvw/example/countries>
    a csvw:Table ;
    csvw:url <http://example.org/countries.csv> ;
    csvw:row [ a csvw:Row ;
      csvw:rownum "1"^^xsd:integer ;
      csvw:url <#row=2> ;
      csvw:describes _:8228a149-8efe-448d-b15f-8abf92e7bd17
    ]
```

However we now have the problem that the `csvw:Row` is also a blank
node. So how do we annotate the "row" in the annotated table if we
can't identify it? The rows themselves

Because the row itself has no identifier, there is no way to
annotate




## If you want this just use fully qualified URI's in your templates!





This is strange too and mixes up concerns.






Now lets look at the data again to see what happened to our `#parent-node`

Now lets look at the `propertyUrl`. In CSVW a propertyUrl is not
actually a URI, but a [URI template](https://www.rfc-editor.org/rfc/rfc6570)


So what is going on here? Well we're trying to align the identifier
for our `csvw:Column` definition to that of the predicate we use in
the resultant RDF. Granted some people may find the above a little
unusual or distasteful; however it is in no way as weird or unusual as
this issue. The fact is a `csvw:Column` needn't be disjoint from a
predicate.





that is besides the point; a
`csvw:Column` in CSVW's abstract table can be the same as a predicate
if I want it be.








; the algorithm for doing this is specified in
[RFC3986](https://www.rfc-editor.org/rfc/rfc3986#section-5), but
essentially it's an important way to support relative links.

Relative links in software systems are very important, they mean you
can easily and reliably reference collections of files and resources










```turtle
_:b0 csvw:url "https://gist.githubusercontent.com/RickMoynihan/044cf6e38da6a52acaad4d9597222f02/raw/63041dd6702252ddb7e0dab06120afbe343a262c/expansion.csv"^^<http://www.w3.org/2001/XMLSchema#anyURI> .
```
