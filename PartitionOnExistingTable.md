<B>Problem: </B>
<P>Assume there is a table with very huge data and the size of the table grows eventually. When queried on this data, this may lead to performance issues. 
For example, if there is table that stores the notifications in a data intense application, that is used to save the notifications of all the tenants. This table will increase gradually within no time. The data for tenants can grow drastically and over a period of time, it could result in performance issues. In the first place, it’s not a good design to have the data saved in single table.</P>

 ![image](https://user-images.githubusercontent.com/16013592/156284718-e2969d84-53f8-4a48-b9ba-be54058e3f6f.png)

Below is the DDL of the table <br>

CREATE TABLE IF NOT EXISTS notification ( <br>
&nbsp;&nbsp;&nbsp;&nbsp;notification_id uuid NOT NULL DEFAULT uuid_generate_v4(), <br>
&nbsp;&nbsp;&nbsp;&nbsp;tenant_id character varying(40) NOT NULL, <br>
&nbsp;&nbsp;&nbsp;&nbsp;notification_type_id uuid NOT NULL, <br>
&nbsp;&nbsp;&nbsp;&nbsp;description text, <br>
&nbsp;&nbsp;&nbsp;&nbsp;processor character varying(255), <br>
&nbsp;&nbsp;&nbsp;&nbsp;status character varying(255), <br>
&nbsp;&nbsp;&nbsp;&nbsp;context_details jsonb, <br>
&nbsp;&nbsp;&nbsp;&nbsp;context_id character varying(40), <br>
&nbsp;&nbsp;&nbsp;&nbsp;CONSTRAINT notification_key PRIMARY KEY (notification_id) <br>
 )

There is no index on tenant_id column. Insert some million records in the table. Fetch the records based on tenant id.

<B>QUERY</B> : <i>select * from notification where tenant_id = '8a5b6545-03a6-43a4-801d-a35580da0c67' </i>

<B>Execution Plan:</B>
 
 <img width="451" alt="image" src="https://user-images.githubusercontent.com/16013592/156284747-c3705f43-0c9b-47c4-8595-deefd3cd3b03.png">


<B>Output:</B> <i>Successfully run. Total query runtime: 3 min 25 secs.
524561 rows affected.</i>
<br>
<B>Adding an Index:</B>
Check if indexing alone helps. Now add index to the tenant_id column of the table.<br>

<i>CREATE INDEX tenant_indx on notification(tenant_id);</i>

<B>QUERY :</B> <i>select * from notification where tenant_id = '8a5b6545-03a6-43a4-801d-a35580da0c67' </i>

<B>Output:</B> <i>Total query runtime: 3 min 13 secs.
524561 rows affected.</i>

<B>Addition of Partition:</B>
<p>
Now the table can be partitioned based on the table. First we will have to create a partition based on List. Then for the unique values of tenant, we create a new partition by suffixing the tenant_id to the partition name. <p>
<br>
Below script will <br>
&nbsp;&nbsp;&nbsp;&nbsp;1)	Rename the old table <br>
&nbsp;&nbsp;&nbsp;&nbsp;2)	Create a new table with partition <br>
&nbsp;&nbsp;&nbsp;&nbsp;3)	Reads the unique tenant values from old table <br>
&nbsp;&nbsp;&nbsp;&nbsp;4)	Creates new partition tables considering tenant as the partition value. Also the partition table name will be suffixed with the tenant Id. <br>
	
<p><i>
do <br>
$$ <br>
declare <br>
&nbsp;&nbsp;&nbsp;   tenant text; <br>
&nbsp;&nbsp;&nbsp;   table_tenant text;<br>
&nbsp;&nbsp;&nbsp;   rec record;<br>
begin<br>
EXECUTE 'ALTER TABLE notification rename to notification_old'<br>
EXECUTE 'CREATE TABLE IF NOT EXISTS notification<br>
(<br>
 &nbsp;&nbsp;&nbsp;&nbsp;  notification_id uuid NOT NULL DEFAULT uuid_generate_v4(),<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   tenant_id character varying(40) NOT NULL,<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   notification_type_id uuid NOT NULL,<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   description text,<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   processor character varying(255),<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   status character varying(255),<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   context_details jsonb,<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   context_id character varying(40),<br>
 &nbsp;&nbsp;&nbsp;&nbsp;   CONSTRAINT notification_key PRIMARY KEY (notification_id)<br>
) PARTITION BY LIST(tenant_id)'<br>
FOR rec IN <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     select DISTINCT tenant_id from notification_old <br>
 &nbsp;&nbsp;  loop <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     tenant := rec.tenant_id; <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     table_tenant := 'notification_'|| REPLACE (rec.tenant_id, '-', '_'); <br>
 &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;     execute 'CREATE TABLE IF NOT EXISTS ' || table_tenant || ' PARTITION OF notification FOR VALUES IN ('''|| tenant ||''')'; <br>
 &nbsp;&nbsp;  END LOOP; <br>
end; <br>
$$ language plpgsql </i><p> <br>
<br>
Then to copy the data from older table to newer table, 
	<B>Query:</B> <i>insert into notification
	select * from notification_old </i>
<br>
Execution Plan:

<img width="924" alt="image" src="https://user-images.githubusercontent.com/16013592/156395631-763f0f14-7088-4e44-9e30-29a0790f85c9.png">


<br>
The query executes directly on the partitioned value with given tenant_id <br>
<b>Output</b> : <i>Successfully run. Total query runtime: 2 min 6 secs.
	524561 rows affected.</i>


<b>Conclusion:</b> Applying partitions on tables will have very good performance improvements when data is filtered on partition value. However, if entire data needs to be fetched from the parent table without any filter, the query has to union all the partition tables.

<b>References:<b><br>
https://www.postgresql.org/docs/10/ddl-partitioning.html <br>
https://www.enterprisedb.com/postgres-tutorials/how-use-table-partitioning-scale-postgresql <br>
