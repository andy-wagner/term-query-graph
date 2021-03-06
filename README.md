Term-Query Graph for Search Query Recommendation
================================================

A Scala implementation of a Term-Query Graph for search query recommendations.

See [Bonchi et al. (WWW 2011)](http://dl.acm.org/citation.cfm?id=1963201), 
[Bonchi et al. (SIGIR 2012)](http://zola.di.unipi.it/rossano/wp-content/papercite-data/pdf/sigir12.pdf). An implementation similar to this was used for Feild and Allan (SIGIR 2013).

License
=======

See LICENSE in root directory.


Compiling
=========

This package uses Scala 2.11 and SBT (I'm currently using version 0.13.1). 
Compiling is simple; navigate to the main directory in a console and type:

    sbt package

This will create a jar file:

    target/scala-2.11/termquerygraph_2.11-1.1.jar



Usage
=====

There are two stages required to use this package for query recommendations.
They are: 

1. creating a term-query graph
2. running queries 

We elaborate on these below.


<a name="creating-graph"></a>
Creating a term-query graph
---------------------------

This step assumes that you have a file consisting of query pairs and counts. 
In the term-query graph, queries are nodes and the counts are placed on
directional edges going from `query1` to `query2`. Note that outgoing edges are
normalized for each query. The file should have a header and three tab-separated
columns: `query1`, `query2`, and `count`. Here's an example:

    query1  query2  count
    foo  foobar  103
    foo  bar    200
    foobar  foo bar 19
    ...

There is a fuller example in:

    samples/pairs.tsv

Assuming you have such a file, you can generate the term-query graph using
the `ProcessQueryPairs` class. Here's the usage:

    scala -cp target/scala-2.11/termquerygraph_2.11-1.1.jar \
        edu.umass.ciir.tqgraph.ProcessQueryPairs
    
    Usage: ProcessQueryPairs <query pair file> <out dir> [<query count est>]
    
     Produces three files with tab-delimited columns in <out dir>:
    
       term-postings.tsv.gz
       query-id-map.tsv.gz
       query-rewrite-matrix.tsv.gz


E.g., to create a term-query graph for our sample pairs and store it in 
`samples/graph-data`, do:

    scala -cp target/scala-2.11/termquerygraph_2.11-1.1.jar \
        edu.umass.ciir.tqgraph.ProcessQueryPairs \
        samples/pairs.tsv \
        samples/graph-data

If you have a lot of queries in the query pairs file, and you know how many, 
then supplying that number as the third argument to `ProcessQueryPairs` should
make things a little faster (it helps when building the internal hash maps).

Of the three output files, `term-postings.tsv.gz` lists the id of all queries in
which a term occurs; `query-id-map.tsv.gz` maps query ids to the query text; and
`query-rewrite-matrix.tsv.gz` contains a sparse re-write matrix.


Running Queries
----------------

This step assumes you have a file of queries to process. You can then call the
`QFGOps` class to generate recommendations. The usage is as follows:

    scala -cp target/scala-2.11/termquerygraph_2.11-1.1.jar \
        edu.umass.ciir.tqgraph.QFGOps
        
    Usage: QFGOps  <query file> <rewrite matrix dir> <term cache dir> [options]

       Divides each query into terms (after replacing punctuation with a space)
       and performs a random walk on each term. The random walk vectors for 
       each term are written to a file in <term cache dir>. The term vectors
       for the terms in a query are then combined to produce the resulting
       recommendations. Recommendations are output to stdout.

    Options:

       --alpha=<alpha>
           The restart probability for the random walk. 
           Default: 0.9

       --k=<num>
           The number of top recommendations to use per term and per query. 
           Consider setting this to 1,000 or 10,000 to improve performance.
           Set to -1 for all.
           Default: -1
           
       --parallel
           If present, then the random walk for a term will be split up into
           <split-count> parts and the parts carried out in parallel.
           Default: false
           
       --split-count=<num>
           The number of parts to split each random walk into when done in
           parallel. 
           Default: 4
           
       --convergence-distance=<num>
           The maximum distance between two consecutive iterations of the 
           random walk algorithm that should be viewed as signifying 
           convergence. 
           Default: 0.005


Of the three required parameters, `<query file>` should contain one query per 
line, `<rewrite matrix dir>` is the graph data directory that we produced in the
[Creating a term-query graph](#creating-graph) section, and the 
`<term cache dir>` is a directory in which random walks for query terms will be
stored&mdash;one file per term.

As an example, let's use the graph data we generated above 
(`samples/graph-data`), the sample query file (`samples/queries.txt`), and a 
directory for the cache (`samples/term-cache`):

    scala -cp target/scala-2.11/termquerygraph_2.11-1.1.jar \
        edu.umass.ciir.tqgraph.QFGOps \
            samples/queries.txt \
            samples/graph-data \
            samples/term-cache

That produces the output:

    Using:
        paraellel: false
        split count: 4
        alpha: 0.9
        convergence dist.: 0.005
        k: -1
    Loading transition matrix...done!
        Random walk for [::UNIFORM::], iterating: .....
        Random walk for [foobar], iterating: .......
    Processing [foobar]:
    foobar  foobar  -0.6130000861387812 foo bar -1.3029194266465904 foo -1.32...
        Random walk for [foo], iterating: ......
        Random walk for [bar], iterating: .....
    Processing [foo bar]:
    foo bar foo bar -2.063279216785201  foobar  -2.485381334434046  foo -2.56...
    Processing [bar foo]:
    bar foo foo bar -2.063279216785201  foobar  -2.485381334434046  foo -2.56...
    Processing [foo]:
    foo foo bar -1.197558910988764  foo -1.217539355435611  foobar  -1.412572...
    Processing [bar]:
    bar foo bar -0.8657203057964369 foobar  -1.0728088017382271 bar -1.151292...

Note that this include both the `stderr` and `stdout`. To store the 
recommendations, do: 


    scala -cp target/scala-2.11/termquerygraph_2.11-1.1.jar \
        edu.umass.ciir.tqgraph.QFGOps \
            samples/queries.txt \
            samples/graph-data \
            samples/term-cache > samples/recommendations.tsv

Recommendations are in the tab-delimited format:

    <query 1> <recommendation 1> <score 1> <recommendation 2> <score 2> ...
    <query 2> ...
    ...

And the scores are in log space. For the example above, the `stdout` is: 

    cat samples/recommendations.tsv 

    foobar  foobar  -0.6130000861387812 foo bar -1.3029194266465904 foo -1.32...
    foo bar foo bar -2.063279216785201  foobar  -2.485381334434046  foo -2.56...
    bar foo foo bar -2.063279216785201  foobar  -2.485381334434046  foo -2.56...
    foo foo bar -1.197558910988764  foo -1.217539355435611  foobar  -1.412572...
    bar foo bar -0.8657203057964369 foobar  -1.0728088017382271 bar -1.151292...

*Note:* all terms in the query must be in the graph somewhere in order for the 
recommendation set to be non-empty.

For large graphs, you'll need more memory. E.g.,

    JAVA_OPTS="-Xmx6g" \
    scala -cp target/scala-2.11/termquerygraph_2.11-1.1.jar \
        edu.umass.ciir.tqgraph.QFGOps \
            samples/queries.txt \
            samples/graph-data \
            samples/term-cache