# Trace vocabulary

These terms are close enough that swapping two changes what a finding says.

**Metric tree** - a hierarchical model mapping a north star metric down to the input
metrics that drive it, connected by the arithmetic relating them. `get_trees` lists them.

**Node** - one metric in a tree, either an input or the result of an arithmetic
relationship between other nodes.

**Time grain** - the period a tree's values are aggregated into. A tree lists the grains
it supports.

**Attribute** - a categorisation of the data, a dimension to group or filter by.

**Attribute value** - one specific value of an attribute.

**Segment** - an unordered set of attributes. Which categorisations are in play, not
which of their values.

**Slice** - a set of attribute and value pairs. One concrete combination, one row.

**Cube** - all the slices produced from a segment combination.

So "the largest slice" is one combination of values; "the strongest segment" is a choice
of attributes to break the metric down by.

**Mixshift** - a change in a parent metric caused by the composition of its slices moving
rather than by per-slice values moving. Never collapse a mixshift effect and a rate
effect into one number.

**Contribution** - how much a child node or slice accounts for of a change in its parent.
Signed, and can exceed the parent's net change in both directions. Do not describe one as
a share of the total unless the result says it is.
