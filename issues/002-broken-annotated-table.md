# Shattered dreams: CSVW's annotated table

One of CSVW's [stated use
cases](https://www.w3.org/TR/2016/NOTE-csvw-ucr-20160225/#R-AnnotationAndSupplementaryInfo)
was to provide annotations on tabular data, in particular data in the
form of a CSV file.

CSVW aims to do this by adding a metadata file, which describes a CSV
file, and grants it a dialect, an optional schema. The metadata
document is a JSON-LD derived format for annotating the CSV file,
within what CSVW calls the [annotated table
model](https://www.w3.org/TR/2015/REC-tabular-data-model-20151217/#dfn-annotated-table)

My proposition is that the standardised semantics of CSVW's annnotated
table model are largely incompatible with linked data dereferencing.
CSVW sadly yields and encourages unharmonised outcomes, leading to a
proliferation of incomplete and inadequate representations, rather
than supporting a common unique representation.

It is easy to read the standards, or to look at some metadata
document examples and believe you understand them and know how they
work. Unfortunately this is not the case, the CSVW standards like many
standards are complex, and hard to read, and do little to present the
big picture. There are many subtleties with wide ranging implications.

I have been working with CSVW for years, and am coming at this from
the perspective of trying to find solutions. Whether that is through
work arounds, extensions or updates through the standardisation
process. We want to work with the CSVW community to solve these
issues, though I believe there are some fundamental problems with the
standards, it is also my belief that they can be resolved.





> Ok but, the CSV is the input format, the csvm metadata file contains
> transformation instructions, and the `csv2rdf` algorithm is a
> process which preserves the provenance of the original CSV
> representation through the csvw:describes predicate, which relates a
> row to the linked data resource it creates!
>
> You're just misunderstanding the relationship between the various
> pieces!

The mechanics of this is true, lets look at some csv2rdf standard mode
output from the specification to explain. Unfortunately the
specification doesn't include the actual metadata file which yields
this result, however there are some [similar
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

When `csv2rdf` is run on a document like the above something like this
will be the output of the transformation:

```turtle
@base <http://example.org/countries.csv> .

<#AD>
  <#countryCode> "AD" ;
  <#latitude> "42.5" ;
  <#longitude> "1.6" ;
  <#name> "Andorra" .

<#AE>
  <#countryCode> "AE" ;
  <#latitude> "23.4" ;
  <#longitude> "53.8" ;
  <#name> "United Arab Emirates" .

<#AF>
  <#countryCode> "AF" ;
  <#latitude> "33.9" ;
  <#longitude> "67.7" ;
  <#name> "Afghanistan" .
```

NOTE that the output of this process is to create new identifers
logically in the CSV document, but that these identifiers are not
themselves the same as the identifiers for the CSV rows. For those we
need to look at the representation of the annotated table model where
we see this emitted (under standard mode):


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
      csvw:describes <#AD>
    ], [ a csvw:Row ;
      csvw:rownum "2"^^xsd:integer ;
      csvw:url <#row=3> ;
      csvw:describes <#AE>
    ], [ a csvw:Row ;
      csvw:rownum "3"^^xsd:integer ;
      csvw:url <#row=4> ;
      csvw:describes <#AF>
    ]
  ] .
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

Without an `@id` we can't identify which table this is; and logically
the annotated table is still distinct from the CSV file it is backed
by.

CSVW does however allow you to assign your own `@id` to the logical
table, e.g. we could add the line `@id` to the above json-ld, setting
the `@id` to be the same as the `@base`:

```
{
 "@context": ["http://www.w3.org/ns/csvw", {"@base": "http://example.org/countries"}],
  "@id": ""
  ...
}

```

This is the right starting point for our abstract annotated table. It
would give us something like this:

```
<http://example.org/countries>
    a csvw:Table ;
    csvw:url <http://example.org/countries.csv> ;
    csvw:row [ a csvw:Row ;
      csvw:rownum "1"^^xsd:integer ;
      csvw:url <#row=2> ;
      csvw:describes <#AD>
    ]
```

This literally says there is an "annotated table" at
`<http://www.swirrl.com/csvw/example/countries>` which has a csv `url`
at `countries.csv` and in that csv there exists a `csvw:Row`, though we
don't know which row, which describes the resource found at
`</countries.csv#AD>`.

The data in the `csvw:Row` itself does then give us metadata which
lets us (as humans) work out through deduction that it comes from
`csvw:rownum 1` which can be found at `<#row=2>` in the csv file,
however this information would in many cases naturally want to be part
of the `csvw:Row` identifier and not expressed in such an unhelpful
manner, which necessitates further post processing.

So surely CSVW lets us not only assign an `aboutUrl`, but a `rowUrl`
or an identifier for the row in the annotated table?  Well no, it turns
out if you read the part of the standard it's not possible:

> 4.6.1: In standard mode only, establish a new blank node R which
> represents the current row.

Yes, in the annotated table, your `csvw:Row`'s can never have
identifiers, they are always and only ever blank nodes. The specs do
leverage [RFC7111](https://www.rfc-editor.org/rfc/rfc7111) which was
largely developed to support CSVW.
