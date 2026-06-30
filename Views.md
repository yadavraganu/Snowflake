# Materialized Views
A materialized view is a pre-computed data set derived from a query specification (the SELECT in the view definition) and stored for later use. 
Because the data is pre-computed, querying a materialized view is faster than executing a query against the base table of the view. 
This performance difference can be significant when a query is run frequently or is sufficiently complex. 
As a result, materialized views can speed up expensive aggregation, projection, and selection operations, especially those that run frequently and that run on large data sets.
### Deciding When to Create
#### Materialized View
- Query results contain a small number of rows and/or columns relative to the base table (the table on which the view is defined).
- Query results contain results that require significant processing, including - Analysis of semi-structured data or Aggregates that take a long time to calculate.
- The query is on an external table, which might have slower performance compared to querying native database tables or Apache Iceberg™ tables.
- The view’s base table does not change frequently.
#### Regular View
- The results of the view change often.
- The results are not used often (relative to the rate at which the results change).
- The query is not resource intensive so it is not costly to re-run it
