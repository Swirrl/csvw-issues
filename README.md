# csvw-issues

This repository contains a collection of notes and problems I have
found with the CSVW specifications.

The background for many of these issues is that we would like very
much to support a [vision for CSVW as outlined here](./csvw-vision.md).

I hope many of these issues can be further clarified, and I can be
corrected on a misinterpretation of the specifications. However I
believe these issues are fundamental issues with the specifications
deserving discussion, and hopefully resolutions or appropriate work
arounds.

It is easy to read the standards, or to look at some metadata document
examples and believe you understand them and know how they work.
Unfortunately this is not the case, the CSVW standards are complex,
hard to read, and surprising. They do little to present the big
picture. There are many subtleties with wide ranging implications.

I have been working with CSVW for years, and am coming at this from
the perspective of trying to find solutions. Whether that is through
work arounds, extensions or updates through the standardisation
process.

We want to work with the CSVW community to solve these issues, though
I believe there are some fundamental problems with the standards, it
is also my belief that they can be resolved.

## Issues

As we will see, there are many impedance mismatches between CSVW and
linked data, and in-spite of their shared origins and goals they are
hard to align. Consequently it is hard to integrate these two
technologies such that they are both mutually beneficial and aligned.

First read:

- [A vision for CSVW](./csvw-vision.md) for context and background on
  what we hope to achieve.
- [What is the annotated table model?](./issues/003-what-is-the-annotated-table-model.md) to clear up some misconceptions on this topic.

Then consider the issues:

1. [Alignment Problems](./issues/001-aligning-linked-data-and-annotated-table.md)
2. [Template Evaluation is "weird"](./issues/002-template-evaluation.md)
