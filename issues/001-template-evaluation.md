# CSVW's weird evaluation semantics

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


## If you want this just use fully qualified URI's in your templates!

This is extremely awkward to do in practice. Hardcoding URI's
prohibits portability, and forces people to have prior knowledge of
what specific URL will be used at publication time.

The final URL's are frequently managed by content management and
publishing systems, and may not be known in advance. Where data is
prepared in advance by a different team, that team may need to defer
the decision of the final URL to the publishing team who may lack the
expertise to correctly edit and test the changes are correct.

Similarly this approach forces the simple task of coining identifiers
to become an extremely error prone and unneccessarily beuraucratic and
political process.

If the identifiers are allowed to be relative, then data engineers can
effectively defer these decisions to the publisher; whilst still
allowing the data to work properly in a local context.

# Solutions

Unfortunately there are no good obvious solutions here.

## Solution 1: Change the base for resolution of templates

This amounts to a small but important deviation from the CSVW
standards. It essentially amounts to replacing a single noun in
the specification.

Where [tabular metadata spec says this](https://www.w3.org/TR/2015/REC-tabular-metadata-20151217/#uri-template-properties)

> 3. resolving the resulting URL against the base URL of the table url if not null.

If we replaced the phrase _table url_ with the word _metadata-document
url_ this whole problem would disappear.

In my opinion this would have been by far been the best solution. I
should also add that this point was raised numerous times by at least
four of the six named authors and chairs of the working group. All of
whom seemed to think this was a bad idea, yet there is no documented
decision on why the reverse decision (to use the _table url_) was
made. For a comprehensive exposé of the working groups decision making
here please see
[w3c/csvw#888](https://github.com/w3c/csvw/issues/888).

Unfortunately changing this would represent a breaking or divergent
change in a documented standard. Changing the resolution under the
guise of a new `csv2rdf` is unfortunately not sufficient, because the
templates need to resolve to the same URI's in both the annotated
table and in the RDFization, and the resolution is unfortunately
specified in the core tabular metadata standard; not the csv2rdf
standard.

## Solution 2: Introduce new variables to the URItemplate context

Another solution would be to introduce new variables into the context
of the URITemplates which would give you the ability to derive
identifiers on a different base URI.  For example we could define:

- `_base` to be the base URI of the metadata document (whether that is
  set explicitly with `@base` or derived explicitly from the metadata
  documents location).
- `_tableid` to be the `@id` of the currently scoped `csvw:Table`, as
  this may be a blank node, in those cases we would need to set it to
  a `null` value.

NOTE: there are different semantics between resolution in terms of a
base and template expansion.

For example URI resolution of `/blah` against the base
`http://example.org/data/foo.json` would yield something like
`http://example.org/blah` whilst the expansion of the template
`{+_base}/blah` would yield something like
`http://example.org/data/base`. This is consistent with template
expansion, and we can get the desired effect by ensuring `_base` is
calculated as essentially the parent path of the metadata document, or
whatever was explicitly set as `@base`.

One small thing to note is that documents using these extra variables
are subtly incompatible with compliant processors. This is because
syntactically valid URItemplates are not permitted to fail because of
unbound variables in their context.

For example a standards compliant template will interpret the template
`{+ _base}#foo` as equivalent to `#foo` which will then be expanded in
terms of the CSV file, e.g. `http://example.org/table.csv#foo` where
as a processor with `_base` mapped to `http://swirrl.com/dataset`
could yield something radically different like `http://swirrl.com/dataset#foo`

For this reason this is strictly speaking an incompatible change; but it
is explicit and requires opt in.
