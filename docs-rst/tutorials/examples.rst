.. _ksql_examples:

KSQL Examples
=============

These examples use a ``pageviews`` stream and a ``users`` table.

.. contents:: Contents
    :local:
    :depth: 2


Creating streams
----------------

Prerequisite:
    The corresponding Kafka topics must already exist in your Kafka cluster.

Create a stream with three columns on the Kafka topic that is named ``pageviews``. It is important to instruct KSQL the format
of the values that are stored in the topic. In this example, the values format is ``DELIMITED``.

.. code:: sql

    CREATE STREAM pageviews \
      (viewtime BIGINT, \
       userid VARCHAR, \
       pageid VARCHAR) \
      WITH (KAFKA_TOPIC='pageviews-topic', \
            VALUE_FORMAT='DELIMITED');

**Associating Kafka message keys:** The above statement does not make
any assumptions about the Kafka message key in the underlying Kafka
topic. However, if the value of the message key in Kafka is the same as
one of the columns defined in the stream in KSQL, you can provide such
information in the WITH clause. For instance, if the Kafka message key
has the same value as the ``pageid`` column, you can write the CREATE
STREAM statement as follows:

.. code:: sql

    CREATE STREAM pageviews \
      (viewtime BIGINT, \
       userid VARCHAR, \
       pageid VARCHAR) \
     WITH (KAFKA_TOPIC='pageviews-topic', \
           VALUE_FORMAT='DELIMITED', \
           KEY='pageid');

**Associating Kafka message timestamps:** If you want to use the value
of one of the columns as the Kafka message timestamp, you can provide
such information to KSQL in the WITH clause. The message timestamp is
used in window-based operations in KSQL (such as windowed aggregations)
and to support event-time based processing in KSQL. For instance, if you
want to use the value of the ``viewtime`` column as the message
timestamp, you can rewrite the above statement as follows:

.. code:: sql

    CREATE STREAM pageviews \
      (viewtime BIGINT, \
       userid VARCHAR, \
       pageid VARCHAR) \
      WITH (KAFKA_TOPIC='pageviews-topic', \
            VALUE_FORMAT='DELIMITED', \
            KEY='pageid', \
            TIMESTAMP='viewtime');

Creating tables
---------------

Prerequisite:
    The corresponding Kafka topics must already exist in your Kafka cluster.

Create a table with several columns. In this example, the table has columns with primitive data
types, a column of ``array`` type, and a column of ``map`` type:

.. code:: sql

    CREATE TABLE users \
      (registertime BIGINT, \
       gender VARCHAR, \
       regionid VARCHAR, \
       userid VARCHAR, \
       interests array<VARCHAR>, \
       contact_info map<VARCHAR, VARCHAR>) \
      WITH (KAFKA_TOPIC='users-topic', \
            VALUE_FORMAT='JSON',
            KEY = 'userid');



Working with streams and tables
-------------------------------

Now that you have the ``pageviews`` stream and ``users`` table, take a
look at some example queries that you can write in KSQL. The focus is on
two types of KSQL statements: CREATE STREAM AS SELECT and CREATE TABLE
AS SELECT. For these statements KSQL persists the results of the query
in a new stream or table, which is backed by a Kafka topic.

Transforming
~~~~~~~~~~~~

For this example, imagine you want to create a new stream by
transforming ``pageviews`` in the following way:

-  The ``viewtime`` column value is used as the Kafka message timestamp
   in the new stream’s underlying Kafka topic.
-  The new stream’s Kafka topic has 5 partitions.
-  The data in the new stream is in JSON format.
-  Add a new column that shows the message timestamp in human-readable
   string format.
-  The ``userid`` column is the key for the new stream.

The following statement will generate a new stream,
``pageviews_transformed`` with the above properties:

.. code:: sql

    CREATE STREAM pageviews_transformed \
      WITH (TIMESTAMP='viewtime', \
            PARTITIONS=5, \
            VALUE_FORMAT='JSON') AS \
      SELECT viewtime, \
             userid, \
             pageid, \
             TIMESTAMPTOSTRING(viewtime, 'yyyy-MM-dd HH:mm:ss.SSS') AS timestring \
      FROM pageviews \
      PARTITION BY userid;

Use a ``[ WHERE condition ]`` clause to select a subset of data. If you
want to route streams with different criteria to different streams
backed by different underlying Kafka topics, e.g. content-based routing,
write multiple KSQL statements as follows:

.. code:: sql

    CREATE STREAM pageviews_transformed_priority_1 \
      WITH (TIMESTAMP='viewtime', \
            PARTITIONS=5, \
            VALUE_FORMAT='JSON') AS \
      SELECT viewtime, \
             userid, \
             pageid, \
             TIMESTAMPTOSTRING(viewtime, 'yyyy-MM-dd HH:mm:ss.SSS') AS timestring \
      FROM pageviews \
      WHERE userid='User_1' OR userid='User_2' \
      PARTITION BY userid;

.. code:: sql

    CREATE STREAM pageviews_transformed_priority_2 \
          WITH (TIMESTAMP='viewtime', \
                PARTITIONS=5, \
                VALUE_FORMAT='JSON') AS \
      SELECT viewtime, \
             userid, \
             pageid, \
             TIMESTAMPTOSTRING(viewtime, 'yyyy-MM-dd HH:mm:ss.SSS') AS timestring \
      FROM pageviews \
      WHERE userid<>'User_1' AND userid<>'User_2' \
      PARTITION BY userid;

Joining
~~~~~~~

The following query creates a new stream by joining the
``pageviews_transformed`` stream with the ``users`` table:

.. code:: sql

    CREATE STREAM pageviews_enriched AS \
      SELECT pv.viewtime, \
             pv.userid AS userid, \
             pv.pageid, \
             pv.timestring, \
             u.gender, \
             u.regionid, \
             u.interests, \
             u.contact_info \
      FROM pageviews_transformed pv \
      LEFT JOIN users u ON pv.userid = users.userid;

Note that by default all the Kafka topics will be read from the current
offset (aka the latest available data); however, in a stream-table join,
the table topic will be read from the beginning.

Aggregating, windowing, and sessionization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now assume that you want to count the number of pageviews per region.
Here is the query that would perform this count:

.. code:: sql

    CREATE TABLE pageviews_per_region AS \
      SELECT regionid, \
             count(*) \
      FROM pageviews_enriched \
      GROUP BY regionid;

The above query counts the pageviews from the time you start the query
until you terminate the query. Note that we used CREATE TABLE AS SELECT
statement here since the result of the query is a KSQL table. The
results of aggregate queries in KSQL are always a table because it
computes the aggregate for each key (and possibly for each window per
key) and *updates* these results as it processes new input data.

KSQL supports aggregation over WINDOW too. Let’s rewrite the above query
so that we compute the pageview count per region every 1 minute:

.. code:: sql

    CREATE TABLE pageviews_per_region_per_minute AS \
      SELECT regionid, \
             count(*) \
      FROM pageviews_enriched \
      WINDOW TUMBLING (SIZE 1 MINUTE) \
      GROUP BY regionid;

If you want to count the pageviews for only “Region_6” by female users
for every 30 seconds, you can change the above query as the following:

.. code:: sql

    CREATE TABLE pageviews_per_region_per_30secs AS \
      SELECT regionid, \
             count(*) \
      FROM pageviews_enriched \
      WINDOW TUMBLING (SIZE 30 SECONDS) \
      WHERE UCASE(gender)='FEMALE' AND LCASE(regionid)='region_6' \
      GROUP BY regionid;

UCASE and LCASE functions in KSQL are used to convert the values of
gender and regionid columns to upper and lower case, so that you can
match them correctly. KSQL also supports LIKE operator for prefix,
suffix and substring matching.

KSQL supports HOPPING windows and SESSION windows too. The following
query is the same query as above that computes the count for hopping
window of 30 seconds that advances by 10 seconds:

.. code:: sql

    CREATE TABLE pageviews_per_region_per_30secs10secs AS \
      SELECT regionid, \
             count(*) \
      FROM pageviews_enriched \
      WINDOW HOPPING (SIZE 30 SECONDS, ADVANCE BY 10 SECONDS) \
      WHERE UCASE(gender)='FEMALE' AND LCASE (regionid) LIKE '%_6' \
      GROUP BY regionid;

The next statement counts the number of pageviews per region for session
windows with a session inactivity gap of 60 seconds. In other words, you
are *sessionizing* the input data and then perform the
counting/aggregation step per region.

.. code:: sql

    CREATE TABLE pageviews_per_region_per_session AS \
      SELECT regionid, \
             count(*) \
      FROM pageviews_enriched \
      WINDOW SESSION (60 SECONDS) \
      GROUP BY regionid;

Working with arrays and maps
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``interests`` column in the ``users`` table is an ``array`` of
strings that represents the interest of each user. The ``contact_info``
column is a string-to-string ``map`` that represents the following
contact information for each user: phone, city, state, and zipcode.

The following query will create a new stream from ``pageviews_enriched``
that includes the first interest of each user along with the city and
zipcode for each user:

.. code:: sql

    CREATE STREAM pageviews_interest_contact AS \
      SELECT interests[0] AS first_interest, \
             contact_info['zipcode'] AS zipcode, \
             contact_info['city'] AS city, \
             viewtime, \
             userid, \
             pageid, \
             timestring, \
             gender, \
             regionid \
      FROM pageviews_enriched;

Avro format and integration with Confluent Schema Registry
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. contents::
    :local:

Supported functionality
^^^^^^^^^^^^^^^^^^^^^^^

KSQL can read and write messages in Avro format by integrating with
:ref:`Confluent Schema
Registry <schemaregistry_intro>`.
KSQL will automatically retrieve (read) and register (write) Avro
schemas as needed and thus save you from both having to manually define
columns and data types in KSQL as well as from manual interaction with
the schema registry.

Currently KSQL supports Avro data in the values of Kafka messages:

+-------------+-------------------+----------------------------+
|             | Message Key       | Message Value              |
+=============+===================+============================+
| Avro format | Not supported yet | Supported (read and write) |
+-------------+-------------------+----------------------------+

What is not supported yet:

-  Message keys in Avro format. Message keys in KSQL are always
   interpreted as STRING format, which means KSQL will ignore any Avro
   schemas that have been registered for message keys.
-  Avro schemas with nested fields because KSQL does not yet supported
   nested columns.

Configuring KSQL for Avro
^^^^^^^^^^^^^^^^^^^^^^^^^

You must configure the API endpoint of Confluent Schema Registry by
setting ``ksql.schema.registry.url`` (default:
``http://localhost:8081``) in the KSQL configuration file that you use
to start KSQL. You *should not* use ``SET`` to configure the registry
endpoint.

Using Avro in KSQL
^^^^^^^^^^^^^^^^^^

First you must ensure that:

1. Confluent Schema Registry is up and running.
2. ``ksql.schema.registry.url`` is set correctly in KSQL (see previous
   section).

Then you can use ``CREATE STREAM`` and ``CREATE TABLE`` statements to
read from Kafka topics with Avro-formatted data and ``CREATE STREAM AS``
and ``CREATE TABLE AS`` statements to write Avro-formatted data into
Kafka topics.

Example: Create a new stream ``pageviews`` by reading from a Kafka topic
with Avro-formatted messages.

.. code:: sql

    CREATE STREAM pageviews
      WITH (KAFKA_TOPIC='pageviews-avro-topic',
            VALUE_FORMAT='AVRO');

Example: Create a new table ``users`` by reading from a Kafka topic with
Avro-formatted messages.

.. code:: sql

    CREATE TABLE users
      WITH (KAFKA_TOPIC='users-avro-topic',
            VALUE_FORMAT='AVRO',
            KEY='userid');

Note how in the above example you don’t need to define any columns or
data types in the CREATE statement because KSQL will automatically infer
this information from the latest registered Avro schema for topic
``pageviews-avro-topic`` (i.e., the latest schema at the time the
statement is first executed).

If you want to create a STREAM or TABLE with only a subset of all the
available fields in the Avro schema, then you must explicitly define the
columns and data types.

Example: Create a new stream ``pageviews_reduced``, similar to the
previous example, but with only a few of all the available fields in the
Avro data (here, only the two columns ``viewtime`` and ``pageid`` are
picked).

.. code:: sql

    CREATE STREAM pageviews_reduced (viewtime BIGINT, pageid VARCHAR)
      WITH (KAFKA_TOPIC='pageviews-avro-topic',
            VALUE_FORMAT='AVRO');

KSQL allows you to work with streams and tables regardless of their
underlying data format. This means that you can easily mix and match
streams and tables with different data formats (e.g. join a stream
backed by Avro data with a table backed by JSON data) and also convert
easily between data formats.

Example: Convert a JSON stream into an Avro stream.

.. code:: sql

    CREATE STREAM pageviews_json (viewtime BIGINT, userid VARCHAR, pageid VARCHAR)
      WITH (KAFKA_TOPIC='pageviews-json-topic', VALUE_FORMAT='JSON');

    CREATE STREAM pageviews_avro
      WITH (VALUE_FORMAT = 'AVRO') AS
      SELECT * FROM pageviews_json;

Note how you only need to set ``VALUE_FORMAT`` to Avro to achieve the
data conversion. Also, KSQL will automatically generate an appropriate
Avro schema for the new ``pageviews_avro`` stream, and it will also
register the schema with Confluent Schema Registry.

Running KSQL
------------

KSQL supports various :ref:`modes of operation <modes-of-operation>`, including a standalone mode and a client-server mode.

Additionally, you can also instruct KSQL to execute a single statement
from the command line. The following example command runs the given
``SELECT`` statement and show the results in the terminal. In this
particular case, the query will run until 5 records have been found, and
then terminate.

.. code:: shell

    $ ksql http://localhost:8088 --exec "SELECT * FROM pageviews LIMIT 5;"