## Should you go Beyond Relational Databases?
- Is your current Relational databases, such as MySQL, PostgreSQL good. 
- Do you have tables with lots of columns, 
    - only a few of which are actually used by any particular row?
- Do you have “attribute” tables where each row is a triple of 
    - (foreign key to row in another table, attribute name, attribute value) &  
    - you need ugly joins in your queries to deal with those tables?
- Have you given up on using columns for structured data, 
    - Instead just serialising it (to JSON, YAML, XML or whatever) and 
    - dumping the string into your database?
- Does your schema have a large number of many-to-many join tables or tree-like structures 
    - (a foreign key that refers to a different row in the same table)?
- Do you find yourself frequently needing to make schema changes so that you can properly represent incoming data?
- Other symptoms relate to the scalability of your system:
    - Are you reaching the limit of the write capacity of a single database server? 
        - (If read capacity is your problem, you should set up master-slave replication. 
        - Also make sure that you have first given your database the fattest hardware you can afford, 
        - you have optimised your queries, and your schema cannot easily be split into shards.)
    - Is your amount of data greater than a single server can sensibly hold?
    - Are your page loads being slowed down unacceptably by background batch processes overwhelming the database?
    - Focus first on the Customer functional requirements, only after the product is hugely successful would you get Scaling challenges. 

### Document databases and BigTable
- The BigTable paper describes how Google developed their own massively scalable database for internal use, 
    - as basis for several of their services. 
    - The data model is quite different from relational databases: 
        - columns don’t need to be pre-defined, and 
        - rows can be added with any set of columns. 
        - Empty columns are not stored at all.
- BigTable inspired many developers to write their own implementations of this data model; 
    - amongst the most popular are HBase, Hypertable and Cassandra. 
    - The lack of a pre-defined schema can make these databases attractive in applications 
    - where the attributes of objects are not known in advance, or change frequently.

Document databases have a related data model (although the way they handle concurrency and distributed servers can be quite different): a BigTable row with its arbitrary number of columns/attributes corresponds to a document in a document database, which is typically a tree of objects containing attribute values and lists, often with a mapping to JSON or XML. Open source document databases include Project Voldemort, CouchDB, MongoDB, ThruDB and Jackrabbit.

How is this different from just dumping JSON strings into MySQL? Document databases can actually work with the structure of the documents, for example extracting, indexing, aggregating and filtering based on attribute values within the documents. Alternatively you could of course build the attribute indexing yourself, but I wouldn’t recommend that unless it makes working with your legacy code easier.

The big limitation of BigTables and document databases is that most implementations cannot perform joins or transactions spanning several rows or documents. This restriction is deliberate, because it allows the database to do automatic partitioning, which can be important for scaling — see the section on distributed key-value stores below. If the structure of your data is lots of independent documents, this is not a problem — but if your data fits nicely into a relational model and you need joins, please don’t try to force it into a document model.

Are you ready to start learning?
Learning with Treehouse for only 30 minutes a day can teach you the skills needed to land the job that you’ve been dreaming about.

Start a Free Trial

Two people working on a computer
Graph databases
Graph databases live at the opposite end of the spectrum. While document databases are good for storing data which is structured in the form of lots of independent documents, graph databases focus on the relationships between items — a better fit for highly interconnected data models.

Standard SQL cannot query transitive relationships, i.e. variable-length chains of joins which continue until some condition is reached. Graph databases, on the other hand, are optimised precisely for this kind of data. Look out for these symptoms indicating that your data would better fit into a graph model:

you find yourself writing long chains of joins (join table A to B, B to C, C to D) in your queries;
you are writing loops of queries in your application in order to follow a chain of relationships (particularly when you don’t know in advance how long that chain is going to be);
you have lots of many-to-many joins or tree-like data structures;
your data is already in a graph form (e.g. information about who is friends with whom in a social network).
There is less choice in graph databases than there is in document databases: Neo4j, AllegroGraph and Sesame (which typically uses MySQL or PostgreSQL as storage back-end) are ones to look at. FreeBase and DirectedEdge have developed graph databases for their internal use.

Graph databases are often associated with the semantic web and RDF datastores, which is one of the applications they are used for. I actually believe that many other applications’ data would also be well represented in graphs. However, as before, don’t try to force data into a graph if it fits better into tables or documents.

MapReduce
Going on a slight tangent: if background batch processing is your problem and you are not aware of the MapReduce model, you should be. Popularised by another Google paper, MapReduce is a way of writing batch processing jobs without having to worry about infrastructure. Different databases lend themselves more or less well to MapReduce — something to keep in mind when choosing a database to fit your needs.

Hadoop is the big one amongst the open MapReduce implementations, and Skynet and Disco are also worth looking at. CouchDB also includes some MapReduce ideas on a smaller scale.

Distributed key-value stores
A key-value store is a very simple concept, much like a hash table: you can retrieve an item based on its key, you can insert a key/value pair, and you can delete a key/value pair. The value can just be an opaque list of bytes, or might be a structured document (most of the document databases and BigTable implementations above can also be considered to be key-value stores).

Document databases, graph databases and MapReduce introduce new data models and new ways of thinking which can be useful even in a small-scale application; you don’t need to be Google or Facebook to benefit from them. Distributed key-value stores, on the other hand, are really just about scalability. They can scale to truly vast amounts of data — much more than a single server could hold.

Distributed databases can transparently partition and replicate your data across many machines in a cluster. You don’t need to figure out a sharding scheme to decide on which server you can find a particular piece of data; the database can locate it for you. If one server dies, no problem — others can immediately take over. If you need more resources, just add servers to the cluster, and the database will automatically give them a share of the load and the data.

When choosing a key-value store you need to decide whether it should be opimised for low latency (for lightning-fast data access during your request-response cycle) or for high throughput (which is what you need for batch processing jobs).

Other than the BigTables and document databases above, Scalaris, Dynomite and Ringo provide certain data consistency guarantees while taking care of partitioning and distributing the dataset. MemcacheDB and Tokyo Cabinet (with Tokyo Tyrant for network service and LightCloud to make it distributed) focus on latency.

The caveat about limited transactions and joins applies even more strongly for distributed databases. Different implementations take different approaches, but in general, if you need to read several items, manipulate them in some way and then write them back, there is no guarantee that you will end up in a consistent state immediately (although many implementations try to become eventually consistent by resolving write conflicts or using distributed transaction protocols; see the algorithm of Amazon’s Dynamo for an example). You should therefore only use these databases if your data items are independent, and if availability and performance are more important than ACID properties. For more information, read about Brewer’s CAP Theorem, which states that amongst Consistency, Availability and Partition tolerance, you can only choose two, and no database will ever be able to get around that fact.

Richard Jones, co-founder of Last.fm, has written up an excellent overview of distributed key-value stores. Also Tony Bain gives an introduction to the conceptual differences between relational databases and key-value stores, and recently there was a NOSQL event in San Francisco at which a number of different non-relational databases were presented.

Distributed systems are hard… really hard. I suggest that you use them only if you really need the scaling aspects they offer (or just for fun outside of a production environment).