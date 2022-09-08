# Life Expectancy Rejigged

This CSVW cube is a rejigged variant of a CSVW cube Ross provided.

The complete example metadata file can be found [here](./life-expectancy-by-region-sex-and-time.csv-metadata.json)

It aims to be substantially the same in terms of the "data" but
importantly it tries to align the various models of table in CSVW and
a cube, such that all the representations extend and enhance each
other, rather than having multiple representations, each with their
own unique pros and cons.

We also try to exploit natural correspondences in the data model to
simplify both the consumption and production of this data. For example
the CSVW tableSchema is the same `@id` as the cubes DSD, and annotated
accordingly; whilst `csvw:Columns` are also
`qb:ComponentSpecification`s.

## Explanation of Ross's original example

Ross's example sets the `@id` of the metadata document to be on the
CSV file itself:

```
"@id": "http://data.gov.uk/dataset/life-expectancy-by-region-sex-and-time.csv",

```

This is then declared to be a `dcat:distributionOf` of the abstract
dataset:

`http://data.gov.uk/dataset/life-expectancy-by-region-sex-and-time`

This "abstract dataset", then has another distribution which is the
cube which is given a different identifier altogether:

`http://data.gov.uk/dataset/life-expectancy-by-region-sex-and-time/datacube`

The `tableSchema` for the CSV then builds the observations in this cube.

This makes sense in terms of the RDF and is perfectly logical, however
it has a number of issues:

1. The CSVW annotated table does not benefit the linked data cube
2. The Cube does not benefit from the CSVW annotated table

Whilst the cube and the CSV remain distinct `@id`'s users have to pick
either a CSV, a CSVW (annotated table) or a cube. Each option however
comes with some down sides:

### Raw CSV

(included for completeness)

Pros:

- Easy and familiar

Cons:

- No datatypes - everything's a string
- No annotations
- No links
- No schema

### Annotated Table

Pros:

- datatypes
- links
- schema
- Potential for enhanced performance (vs pure RDF implementations) by
  not having to reconstitue the tables from triples.
- Potentially TableGroups and foriegn keys (strong links)
- Linked data (codes link to their codelists etc)

Cons:

- Increase in complexity

### Cube (triples only)

Cons:

- Slow to reconstitute tables from triples
- Requires knowledge of RDF/SPARQL/Linked-Data to use

Pros:

- Consistent schema for stats and their reference data
  - Reliably identify measure, attributes and dimension columns
  - Consistently access code lists not just CSVW but ones defined also
    in skos or other vocabs
  - Potential to assist in automating roll ups, pivots etc without
    error because we know dimensions distinct from other columns
    e.g Doesn't make sense to roll up by attribute
  - Potential for meceness

## Differences

This model attempts to align the cube, and leverage it with the
proposed solutions outlined in the
[two](../issues/001-aligning-linked-data-and-annotated-table.md)
[issues](../issues/002-template-evaluation.md).

For example we assume templates are resolved in terms of the metadata
documents `base` URI and not in terms of the `table:url`.

Points to note:


1. We set the `base`
   [explicitly](https://github.com/Swirrl/csvw-issues/blob/e7351b7bfe17b3f38f14dd0f75cb00e2d9de5ccf/aligned-cube/life-expectancy-by-region-sex-and-time.csv-metadata.json#L3)
   so the relative URIs are the same whatever context you evaluate this within.

2. [We set](https://github.com/Swirrl/csvw-issues/blob/main/aligned-cube/life-expectancy-by-region-sex-and-time.csv-metadata.json#L5) `@id` to `""` so it is the same as the `base`. Setting an
   `@id` and pointing it to the right location is important as it
   ensures dereferencing works properly. We use the empty string only
   to be DRY as it means the URI is the same as the `base`; it could
   also be a `#fragment` id if for example there were multiple tables
   in the metadata document. It could equally be hardcoded to the
   explicit absolute value too.

3. We align the cube URIs within the metadata document, and make the
   `tableSchema` [do double duty as our DSD too](https://github.com/Swirrl/csvw-issues/blob/main/aligned-cube/life-expectancy-by-region-sex-and-time.csv-metadata.json#L15-L17)

4. Columns in the CSV tableSchema are given explicit `@id`'s and [do
   double
   duty](https://github.com/Swirrl/csvw-issues/blob/main/aligned-cube/life-expectancy-by-region-sex-and-time.csv-metadata.json#L20)
   as `qb:ComponentSpecification`'s, and support arbitrary
   annotations.

5. Components [link to their
   dimensions](https://github.com/Swirrl/csvw-issues/blob/main/aligned-cube/life-expectancy-by-region-sex-and-time.csv-metadata.json#L29-L32),
   so columns in the CSVW UI can connect users to dimensions and
   codelists.

6. In the case of locally defined dimensions and components; where
   they are essentially managed with or belong to the dataset, we can
   define them inline. [For
   example](https://github.com/Swirrl/csvw-issues/blob/main/aligned-cube/life-expectancy-by-region-sex-and-time.csv-metadata.json#L65-L70)
   `#measure/life-expectancy` is defined with a relative URI that
   derefences to within metadata document.

Note that in this example we also provide some of the plumbing, for
example [attaching components to the
DSD](https://github.com/Swirrl/csvw-issues/blob/main/aligned-cube/life-expectancy-by-region-sex-and-time.csv-metadata.json#L89-L94).
This is done here primarily for illustrative purposes; tooling such as
csvcubed could do this; or ETL processes if they knew the provided
tableSchema was of an appropriate style.

There are also questions over how much information we should expect to
provide in the CSVW document, and how much we can either infer (or
expect others to infer). I leave all of these good questions open for
now; they're decisions and trade offs for another day.

# Prototype CSVWUI with Rejigged data

NOTE: This prototype is relatively realistic in how it works; but does
have some small bugs and there is a small amount of smoke and mirrors
going on.

So don't worry too much about all the details in the prototype being
correct.

The main point is that it could all work properly! :-)

In particular here is a URL to the aligned dataset dereferencing:

https://deref.netlify.app/data/life-expectancy

and here is an observation within that dataset dereferencing too:

http://deref.netlify.app/data/life-expectancy/observations/W06000015/2005-01-01T00%3A00%3A00%2FP3Y/Female

The observant will note that the metadata in the prototype sets a base
elsewhere and the URI's are slightly different.
