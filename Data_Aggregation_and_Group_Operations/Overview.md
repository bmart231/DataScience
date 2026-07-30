Categorizing a dataset and applying a function to each group, whether an
aggregation or transformation, can be a critical component of a data analysis
workflow. After loading, merging, and preparing a dataset, you may need to
compute group statistics or possibly _pivot tables_ for reporting or
visualization purposes. pandas provides a versatile `groupby` interface,
enabling you to slice, dice, and summarize datasets in a natural way.

One reason for the popularity of relational databases and SQL (which stands for
"Structured Query Language") is the ease, with which data can be joined,
filtered, transformed, and aggregated. However, query languages like SQL import
certain limitations on the kinds of grouping operations that can be performed.
We can perform quite complex group operations by expressing them as custom
Python functions that manipulate the data associated with each group.

In this chapter we will:

- Split a pandas object into pieces using one or more keys (in the form of
  functions, arrays, or DataFrame column names.)
- Calculate group summary statistics, like count, mean, or standard deviation,
  or a user-defined function
- Apply within-group transformations or other manipulations, like normalization,
  linear regression, rank, or subset selection.
- Compute pivot tables and cross-tabulations
- Perform quantile analysis and other statistical group analyses.
