pgMemento
=====

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/pgmemento_logo.png "pgMemento Logo")

pgMemento is a versioning approach for PostgreSQL using triggers and server-side
functions in PL/pgSQL and PL/V8.


0. Index
--------

1. License
2. About
3. System requirements
4. Background & References
5. How To
6. Future Plans
7. Media
8. Developers
9. Contact
10. Special thanks
11. Disclaimer


1. License
----------

The scripts for pgMemento are open source under GNU Lesser General 
Public License Version 3.0. See the file LICENSE for more details. 


2. About
--------

Memento. Isn't there a movie called like this? About losing memories?
And the plot is presented in reverse order, right? With pgMemento it is similar.
Databases have no memories of what happened in the past unless it is written
down somewhere. From these notes it can figure out, how a past state might have
looked like.

pgMemento is a bunch of mostly PL/pgSQL scripts that enable auditing of a 
PostgreSQL database. I take extensive use of JSON/JSONB functions to log my data. 
Thus version 9.4 or higher is needed. I also use PL/V8 to work with the JSON-logs 
on the server side. This extension has to be installed on the server. Downloads
can be found [here](http://pgxn.org/dist/plv8/1.4.3/) or [here](http://www.postgresonline.com/journal/archives/341-PLV8-binaries-for-PostgreSQL-9.4-windows-both-32-bit-and-64-bit.html).

As I'm on schemaless side with JSONB one audit table is enough to save 
all changes of all my tables. I do not need to care a lot about the 
structure of my tables.

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/generic_logging.png "Generic logging")

For me, the main advantage using JSONB is the ability to convert table rows
to one column and populate them back to sets of records without losing 
information on the data types. This is very handy when creating a past
table state. I think the same applies to the PostgreSQL extension hstore
so pgMemento could also be realized using hstore except JSONB.

But taking a look into the past of the database is not the main motivation.
In the future everybody will use logical decoding for that, I guess. With 
pgMemento the user shall be able to roll back certain transactions happened in 
the past. I want to design a logic that checks if a transactions can be reverted 
in order that it fits to following transactions, e.g. a DELETE is simple, but 
what about reverting an UPDATE on data that has been deleted later anyway? 

For further reading please consider that pgMemento has neither been tested nor
benchmarked a lot by myself. It is still work-in-progress. 

I tagged a first version of pgMemento (v0.1) that uses the JSON data type 
and can be used along with PostgreSQL 9.3, but it is slower and can not 
handle very big data as JSON strings. I recommend to always use the newest
version of pgMemento.


3. System requirements
----------------------

* PostgreSQL 9.4
* PL/V8


4. Background & References
--------------------------

The auditing approach of pgMemento is nothing new. Define triggers to log
changes in your database is a well known practice. There are other tools
out there which can also be used I guess. When I started the development
for pgMemento I wasn't aware of that there are so many solutions out there
(and new ones popping up every once in while). I haven't tested any of 
them and can not tell you exactly how they differ from pgMemento.

I might have directly copied some elements of these tools, therefore I'm
now referencing tools where I looked up details:

* [audit trigger 91plus](http://wiki.postgresql.org/wiki/audit_trigger_91plus) by ringerc
* [tablelog](http://pgfoundry.org/projects/tablelog/) by Andreas Scherbaum
* [Cyan Audit](http://pgxn.org/dist/cyanaudit/) by Moshe Jacobsen
* [wingspan-auditing] (https://github.com/wingspan/wingspan-auditing) by Gary Sieling
* [pgaudit](https://github.com/2ndQuadrant/pgaudit) by 2ndQuadrant


5. How To
-----------------

### 5.1. Add pgMemento to a database

Run the `SETUP.sql` script to create the schema `pgmemento` with tables
and functions. `VERSIONING.sql` is necessary to restore past table
states and `INDEX_SCHEMA.sql` includes functions to define constraints
in the schema where the tables state has been restored. `REVERT.sql`
contains a procedure to rollback changes of a certain transaction and
some more procedures to move a recreated database state into the
production schema that is truncated in advance.


### 5.2. Start pgMemento

The functions can be used to initialize auditing for single tables or a 
complete schema e.g.

<pre>
SELECT pgmemento.create_schema_audit(
  'public', 
  ARRAY['not_this_table'], ['not_that_table']
);
</pre>

This function creates triggers and an additional column named audit_id
for all tables in the 'public' schema except for tables 'not_this_table' 
and 'not_that_table'. Now changes on the audited tables write information
to the tables 'pgmemento.transaction_log', 'pgmemento.table_event_log' 
and 'pgmemento.row_log'.

When setting up a new database I would recommend to start pgMemento after
bulk imports. Otherwise the import will be slower and several different 
timestamps might appear in the transaction_log table.

ATTENTION: It is important to generate a proper baseline on which a
table/database versioning can reflect on. Before you begin or continue
to work with the database and change contents, define the present state
as the initial versioning state by executing the procedure
`pgmemento.log_table_state` (or `pgmemento.log_schema_state`). 
For each row in the audited tables another row will be written to the 
'row_log' table telling the system that it has been 'inserted' at the 
timestamp the procedure has been executed.


### 5.3. Have a look at the logged information

For example, an UPDATE command on 'table_A' changing the value of some 
rows of 'column_B' to 'new_value' will appear in the log tables like this:

TRANSACTION_LOG

| txid_id  | stmt_date                | user_name  | client address  |
| -------- |:------------------------:|:----------:|:---------------:|
| 1        | 2015-02-22 15:00:00.100  | felix      | ::1/128         |

TABLE_EVENT_LOG

| ID  | transaction_id | op_id | table_operation | schema_name | table_name  | table_relid |
| --- |:--------------:|:-----:|:---------------:|:-----------:|:-----------:|:-----------:|
| 1   | 1              | 2     | UPDATE          | public      | table_A     | 44444444    |


ROW_LOG

| ID  | event_id  | audit_id | changes                  |
| --- |:---------:|:--------:|:------------------------:|
| 1   | 1         | 555      | {"column_B":"old_value"} |
| 2   | 1         | 556      | {"column_B":"old_value"} |
| 3   | 1         | 557      | {"column_B":"old_value"} |

As you can see only the changes are logged. DELETE and TRUNCATE commands
would cause logging of the complete rows while INSERTs would leave a 
blank field for the 'changes' column. 

The logged information can already be of use, e.g. list all transactions 
that had an effect on a certain column by using the ? operator:

<pre>
SELECT t.txid 
  FROM pgmemento.transaction_log t
  JOIN pgmemento.table_event_log e ON t.txid = e.transaction_id
  JOIN pgmemento.row_log r ON r.event_id = e.id
  WHERE 
    r.audit_id = 4 
  AND 
    (r.changes ? 'column_B');
</pre>

List all rows that once had a certain value by using the @> operator:

<pre>
SELECT DISTINCT audit_id 
  FROM pgmemento.row_log
  WHERE 
    changes @> '{"column_B": "old_value"}'::jsonb;
</pre>


### 5.4. Revert certain transactions

The logged information can be used to revert certain transactions that
happened in the past. The procedure is simply called `revert_transaction`.

The procedure loops over each row that was affected by the transaction.
Deleted rows are processed first as they are going to be inserted. The rows
are sorted by their audit_id in ascending order because the oldest row
(which have a lower audit_id) need to be inserted first. Next come updated
rows. They are again sorted by their audit_id but in descending order.
Finally, inserted rows are looked up (`ORDER BY audit_id DESC`) and deleted.

Regarding referential integrity it is important that INSERTs are done before
UPDATEs and UPDATEs are done before DELETEs. The order of the audit_ids is
not only important for references between tables but also for self-references
when dealing with tree-structures in one table.

Note that newly inserted buildings will always get a new audit_id and not
keep their old one (even though it would still be a unique ID). Otherwise
the restoring part would get too complicated.

The query looks like this (in extracts):

<pre>
SELECT * FROM (
  (SELECT r.audit_id, r.changes, e.schema_name, e.table_name, e.op_id
     FROM pgmemento.row_log r
     JOIN pgmemento.table_event_log e ON r.event_id = e.id
     JOIN pgmemento.transaction_log t ON t.txid = e.transaction_id
     WHERE t.txid = $1 AND e.op_id > 2 -- DELETE oder TRUNCATE
       ORDER BY r.audit_id ASC) -- oldest tuples are inserted first
  UNION ALL
  (SELECT ...
     WHERE t.txid = $1 AND e.op_id = 2 -- UPDATE
       ORDER BY r.audit_id DESC) -- youngest tuples are updated first
  UNION ALL
  (SELECT ...
     WHERE t.txid = $1 AND e.op_id = 1 -- INSERT
       ORDER BY r.audit_id DESC) -- youngest tuples are deleted first
) txid_content
ORDER BY op_id DESC -- first process the DELETEs, then the UPDATEs and finally the INSERTs
</pre>


### 5.5. Restore a past state of your database

A table state is restored with the procedure `pgmemento.restore_table_state
(transaction_id, 'name_of_audited_table', 'name_of_audited_schema', 'name_for_target_schema', 'VIEW', 0)`: 
* With a given transaction id the user requests the state of a given table before that transaction.
* The result is written to another schema specified by the user. 
* Tables can be restored as VIEWs (default) or TABLEs. 
* If chosen VIEW the procedure can be executed again (e.g. by using another transaction id)
  and replaces the old view(s) if the last parameter is specified as 1.
* A whole database state might be restored with `pgmemento.restore_schema_state`.

How does the restoring work? Well, imagine a time line like this:

1_2_3_4_5_6_7_8_9_10 [Transactions] <br/>
I_U_D_I_U_U_U_I_D_now [Operations] <br/>
I = Insert, U = Update, D = Delete

Let me tell you how a record looked liked at date x of one sample row 
I will use in the following:

TABLE_A

| ID  | column_B  | column_C | audit_id |
| --- |:---------:|:--------:|:--------:|
| 1   | new_value | abc      | 555      |

Imagine that this row is updated again in transactions 6 and 7 and 
deleted at last in transaction 9. In the 'row_log' table this would
be logged as follows:

| ID  | event_id  | audit_id | changes                                                           |
| --- |:---------:|:--------:|:-----------------------------------------------------------------:|
| ... | ...       | ...      | ...                                                               |
| 66  | 15        | 555      | {"column_B":"new_value"}                                          |
| ... | ...       | ...      | ...                                                               |
| 77  | 21        | 555      | {"column_C":"abc"}                                                |
| ... | ...       | ...      | ...                                                               |
| ... | ...       | ...      | ...                                                               |
| 99  | 21        | 555      | {"ID":1,"column_B":"final_value","column_C":"def","audit_id":555} |
| ... | ...       | ...      | ...                                                               |

#### 5.5.1. The next transaction after date x

If the user just wants to restore a past table/database state by using
a timestamp he will need to find out which is the next transaction 
happened to be after date x:

<pre>
WITH get_next_txid AS (
  SELECT txid FROM pgmemento.transaction_log
  WHERE stmt_date >= '2015-02-22 16:00:00' LIMIT 1
)
SELECT pgmemento.restore_schema_state(
  txid,
  'public',
  'test',
  'VIEW',
  ARRAY['not_this_table'], ['not_that_table'],
  1
) FROM get_next_txid;
</pre>

The resulting transaction has the ID 6.


#### 5.5.2. Fetching audit_ids (done internally)
 
I need to know which entries were valid before transaction 6 started.
This can be done by simple JOIN of the log tables querying for audit_ids.
But still, two steps are necessary:
* find out which audit_ids belong to DELETE and TRUNCATE operations 
  (op_id > 2) before transaction 6 => excluded_ids
* find out which audit_ids appear before transaction 6 and not belong
  to the excluded ids of step 1 => valid_ids

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/fetch_auditids_en.png "Fetching Audit_IDs")

<pre>
WITH
  excluded_ids AS (
    SELECT DISTINCT r.audit_id
    FROM pgmemento.row_log r
    JOIN pgmemento.table_event_log e ON r.event_id = e.id
    JOIN pgmemento.transaction_log t ON t.txid = e.transaction_id
    WHERE t.txid &lt; 6
      AND e.table_relid = 'public.table_A'::regclass::oid
	  AND e.op_id &gt; 2
  ),
  valid_ids AS (  
    SELECT DISTINCT y.audit_id
    FROM pgmemento.row_log y
    JOIN pgmemento.table_event_log e ON y.event_id = e.id
    JOIN pgmemento.transaction_log t ON t.txid = e.transaction_id
    LEFT OUTER JOIN excluded_ids n ON n.audit_id = y.audit_id
    WHERE t.txid &lt; 6
      AND e.table_relid = 'public.table_A'::regclass::oid
      AND (
        n.audit_id IS NULL
        OR
        y.audit_id != n.audit_id
      )
  )
SELECT audit_id FROM valid_ids ORDER BY audit_id;
</pre>


#### 5.5.3. Generate entries from JSONB logs (done internally)

For each fetched audit_id a row has to be reconstructed. This is done by
searching the values of each column of the given table. If the key is not 
found in the row_log table, the recent state of the table is queried.

![alt text](https://github.com/pgMemento/pgMemento/blob/master/material/fetch_values_en.png "Fetching values")

<pre>
SELECT 
  'column_B' AS key, 
  COALESCE(
    (SELECT (r.changes -> 'column_B') 
       FROM pgmemento.row_log r
       JOIN pgmemento.table_event_log e ON r.event_id = e.id
       JOIN pgmemento.transaction_log t ON t.txid = e.transaction_id
       WHERE t.txid >= 6
         AND r.audit_id = f.audit_id
         AND (r.changes ? 'column_B')
         ORDER BY r.id LIMIT 1
    ),
    (SELECT COALESCE(to_json(column_B), NULL)::jsonb 
       FROM schema_Z.table_A
       WHERE audit_id = f.audit_id
    )
  ) AS value;
</pre>

By the end I would have a series of keys and values, like for example:
* '{"ID":1}' (--> first entry found in 'row_log' for column 'ID' has ID 99)
* '{"column_B":"new_value"}' (--> first entry found in 'row_log' for column 'ID' has ID 66)
* '{"column_C":"abc"}' (--> first entry found in 'row_log' for column 'ID' has ID 77)
* '{"audit_id":555}' (--> first entry found in 'row_log' for column 'ID' has ID 99)

These fragments can be put in alternating order and passed to the 
`json_build_object` function to generate a complete replica of the 
row as JSONB.

<pre>
<font color='lightgreen'>-- end of WITH block, that collects valid audit_ids</font>
)
SELECT v.log_entry FROM valid_ids f 
JOIN LATERAL ( <font color='lightgreen'>-- for each audit_id do:</font>
  SELECT json_build_object( 
    q1.key, q1.value, 
    q2.key, q2.value,
    ...
    )::jsonb AS log_entry 
  FROM ( <font color='lightgreen'>-- query for values</font>
    SELECT q.key, q.value FROM ( 
      SELECT 'id' AS key, r.changes -> 'id' 
      FROM pgmemento.row_log r 
      ...
      )q 
    ) q1,
    SELECT q.key, q.value FROM (
      SELECT 'column_B' AS key, r.changes -> 'column_B'
      FROM pgmemento.row_log r ...
      )q 
    ) q2, 
    ...
) v ON (true)
ORDER BY f.audit_id
</pre>


#### 5.5.4. Recreate tables from JSONB logs (done internally)

The last step on the list would be to bring the generated JSONB objects
into a tabular representation. PostgreSQL offers the function 
`jsonb_populate_record` to do this job.

<pre>
SELECT * FROM jsonb_populate_record(null::table_A, jsonb_object);
</pre>

But it cannot be written just like that because we need to combine it with
a query that returns numerous JSONB objects. There is also the function
`jsonb_populate_recordset` to return a set of records but it needs all
JSONB objects to be aggregated which is a little overhead. The solution
is to use a LATERAL JOIN:

<pre>
<font color='lightgreen'>-- previous steps are executed within a WITH block</font>
)
SELECT p.* FROM restore rq
  JOIN LATERAL (
    SELECT * FROM jsonb_populate_record(
      null::table_A, <font color='lightgreen'>-- template table</font>
      rq.log_entry   <font color='lightgreen'>-- reconstructed row as JSONB</font>
    )
  ) p ON (true)
</pre>


#### 5.5.5. Table templates

If I would use 'table_A' as the template for `json_populate_recordset` 
I would receive the correct order and the correct data types of columns but 
could not exclude columns that did not exist at date x. I need to know the 
structure of 'table_A' at the requested date, too.

Unfortunately pgMemento can yet not perform logging of DDL commands, like
`ALTER TABLE [...]`. It forces the user to do a manual column backup. 
By executing the procedure `pgmemento.create_table_template` an empty copy
of the audited table is created in the pgmemento schema (concatenated with
an internal ID). Every created table is documented (with timestamp) in 
`pgmemento.table_templates` which is queried when restoring former table 
states.

IMPORTANT: A table template has to be created before the audited table
is altered.


#### 5.5.6. Work with the past state

If past states were restored as tables they do not have primary keys 
or indexes assigned to them. References between tables are lost as well. 
If the user wants to work on the restored table or database state - 
like he would do with the recent state - he can use the procedures
`pgmemento.pkey_table_state`, `pgmemento.fkey_table_state` and `pgmemento.index_table_state`. 
These procedures create primary keys, foreign keys and indexes on behalf of 
the recent constraints defined in the certain schema (e.g. 'public'). 
If table and/or database structures have changed fundamentally over time 
it might not be possible to recreate constraints and indexes as their
metadata is not logged by pgMemento. 


6. Future Plans
--------------------------------------

First of all I want to to share my idea with the PostgreSQL community
and discuss the scripts in order to improve them. Let me know what you
think of it.

I would be very happy if there are other PostgreSQL developers out 
there who are interested in pgMemento and willing to help me to improve it.
Together we might create a powerful, easy-to-use versioning approach 
for PostgreSQL.

However, here are some plans I have for the near future:
* Do more benchmarking
* Catch changes on DDL level (ALTER TABLE) by using Event Triggers and have a
  better way to manage table templates
* Have another table to store metadata of additional created schemas
  for former table / database states.
* Develop an alternative way to the row-based 'generate_log_entry' function
  to have a faster restoring process.
* Take more consideration in reverting transaction because it's very simple 
  at the moment
* Better protection for log tables?


7. Media
--------------------------------------

I gave a presentation in german at FOSSGIS 2015:
https://www.youtube.com/watch?v=EqLkLNyI6Yk

Slides can be found here (will make an english version soon):
http://slides.com/fxku/pgmemento#/


8. Developers
-------------

Felix Kunde


9. Contact
----------

fkunde@virtualcitysystems.de


10. Special Thanks
-----------------

* Adam Brusselback --> benchmarking and bugfixing
* Hans-Jürgen Schönig (Cybertech) --> recommend to use a generic JSON auditing
* Christophe Pettus (PGX) --> recommend to only log changes
* Claus Nagel (virtualcitySYSTEMS) --> conceptual advices about logging
* Ollyc (Stackoverflow) --> Query to list all foreign keys of a table
* Denis de Bernardy (Stackoverflow, mesoconcepts) --> Query to list all indexes of a table
* Ugur Yilmaz --> feedback and suggestions


11. Disclaimer
--------------

pgMemento IS PROVIDED "AS IS" AND "WITH ALL FAULTS." 
I MAKE NO REPRESENTATIONS OR WARRANTIES OF ANY KIND CONCERNING THE 
QUALITY, SAFETY OR SUITABILITY OF THE SKRIPTS, EITHER EXPRESSED OR 
IMPLIED, INCLUDING WITHOUT LIMITATION ANY IMPLIED WARRANTIES OF 
MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, OR NON-INFRINGEMENT.

IN NO EVENT WILL I BE LIABLE FOR ANY INDIRECT, PUNITIVE, SPECIAL, 
INCIDENTAL OR CONSEQUENTIAL DAMAGES HOWEVER THEY MAY ARISE AND EVEN IF 
I HAVE BEEN PREVIOUSLY ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.
STILL, I WOULD FEEL SORRY.