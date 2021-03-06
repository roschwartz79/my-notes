# Postgres Notes

Taken from [O'Reilly](https://learning.oreilly.com/library/view/learn-postgresql/9781838985288/13643208-aa04-48a7-b956-d7c6d838fcd6.xhtml) along with other sources across the intersphere

# Installation 
- Use brew! `https://formulae.brew.sh/formula/postgresql`
    * `brew services start postgres` or `brew services start postgres@12` depending on what version you downloaded
    * Only want to start 1! Check running services with `brew services list`
- I have been using postgres12
- Set an env variable in .zshrc PGDATA="/usr/local/var/postgres"

## Starting thoughts
* 2 types of users: superusers and normal users
* Postgres uses the underlying filesystem to implement persistance and store data.
* All of the content (data and internal status) is stored in a single directory named PGDATA

Starting Terminology
* Cluster: The whole PostgresSQL instance
* Postmaster: First process the cluster executes, keeps track of the whole cluster
* Database: isolated data container that users can connect to. 

## Cluster
- pg_ctl is a command line utility that performs actions on a cluster such as start and stop
- Status command: `pg_ctl status` returns `pg_ctl: no server running`
- `pg_ctl start` returns `waiting for server to start....
                          2019-12-17 19:31:48.421 CET [96724] LOG:  starting PostgreSQL 12.1 on amd64-portbld-freebsd12.0, compiled by FreeBSD clang version 6.0.1 (tags/RELEASE_601/final 335540) (based on LLVM 6.0.1), 64-bit
                          2019-12-17 19:31:48.421 CET [96724] LOG:  listening on IPv6 address "::1", port 5432
                          2019-12-17 19:31:48.422 CET [96724] LOG:  listening on IPv4 address "127.0.0.1", port 5432
                          2019-12-17 19:31:48.423 CET [96724] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
                          2019-12-17 19:31:48.429 CET [96724] LOG:  ending log output to stderr
                          2019-12-17 19:31:48.429 CET [96724] HINT:  Future log output will go to log destination "syslog".
                           done
                          server started`
                          
- `pg_ctl stop` will shutdown the server. There are options for stop, as this is the most problematic part of working with the postgres server.
    * smart, fast and immediate modes. Specify mode with `-m smart`
- Side note: Postgres must be run with unprivelaged users on the host machine. No privelages users are allowed to run a postgresql server. 

### Connecting to the cluster
* To interact with the cluster, we need to connect to it! We aren't connecting to the whole cluster, rather 1 database that 
the cluster is serving.
* The first databases created are called template databases. They are common databases that we can connect to on a freshly installed cluster. 
* template1 is the first db created, then it is cloned into template0 should anything happen to template1 (template0 can't be connected to, purely a backup)
* Inspect the available databases with `psql -l`
    * A db name "postgres" is also created. This serves as a db admin which is a common space for connections instead of the templates. 
* Templates are used when you create a new databse, they are clones of the templates! So if we add objects into template1, whenever we 
create a new db, that db will also contain those objects. Helpful for creating base databases.
* Template databases are not required, but if they are destroyed, other dbs will need to act as templates for new dbs

### psql command line
- psql accepts many commands, such as -d (db name) -u (username) and -h (host, ipaddr or hostname)
- To start a connection `psql`. If no args are presented, postgres will assume the name and dbuser of the host machine.
- Explicitly, it might look lile `psql -U osusername -d postgres`
- A `#` indicates a superuser and a `>` indicates a normal user.
- To connect to a different DB, specify it! `psql -d template1`
- psql has the ability to write queries acorss multiple lines, look for - vs = on the left side! Statements must end with ;
- To get help on a command, use \h `\h SELECT`
- A connection string can be used as well! `psql postgresql://Z00438B@localhost/template1`

#### Notes about PGDATA

- In the PGDATA directory (e.g. `/usr/local/var/postgres`), there are lots of files, config files and further directories. This is where postgresql data is stored.
- There are objects in the directory that have numbers, these correspond to the databases you have. Use `oid2name` to see the correlation. 

Table Spaces

- Data is not confined to the PGDATA directory! Info can be stored outside this directory sometimes, by means of "tablespaces".
- You can create tablespaces to store less frequently used data on slow disk space and other space on fast disk storage!
- To view the tablespaces `oid2name -s`. There are two defualt tablespaces:
    *   pg_default is the default tablespace that acts as default storage for every object when nothing is explicityly specifically.
    Every object that is stored directly under the PGDATA directory is attached to the pg_default tablespace.
    * pg_global is the tablespace used for system-wide objects. 
    
    
Wrapping up the cluster section, a cluster can handle several databases, all contained in a single PGDATA directory. The pg_ctl and pgql 
commands are great and easy ways to interact with the cluster and its databases. Config of the cluster is handled via text based config
files, such as the postmaster, all which are specified in the PGDATA directory.

## Managing Users and Connections

By default, a single admin user named postgres is created. The concepts of user accounts is encompassed with roles.
- A role can bea single user or even groups of accounts, or even both!

- There are 3 main commands `CREATE ROLE`, `ALTER ROLE`, and `DROP ROLE`

- To create new roles, use the following `CREATE ROLE Rob WITH LOGIN PASSWORD 'xxx';` which will create a new user! The LOGIN is
crucial -> without it the user will not be able to login interactively. 

- There are tons of options, such as a VALID UNTIL option! Check [here](https://www.postgresql.org/docs/current/sql-createrole.html)

You can use roles as groups, which are roles that contain other roles! You can create a role without the login option,
then add the members one by one to the containing super role. Adding a role to a containing role will make a group. We 
can use the `IN ROLE` option to do this. 
*   `CREATE ROLE book_authors WITH NOLOGIN` then `CREATE ROLE rob WITH LOGIN PASSWORD 'xxx' IN ROLE book_authors`

Alternatively, you can "grant" members access to a group with 
* `GRANT ROLE boo-authors TO rob`

You can even have admins for certian groups, which can add new members to the group with the `ADMIN` option. This can be used
with `GRANT` as well.

To remove a role, use `DROP ROLE rob`. Postgres will error if you try and drop a role taht doesn't exist. To prevent errors, 
you can use `IF EXISTS` in the drop to check before you execute a drop. Dropping a group role will not consequently drop
all the roles associated with it. 

#### Inspecting existing roles

- To see what role you are currently running, `SELECT current_role`
- We want more info than that though! We can see more with `\du`. That's better!
- For even more specific info about a user, we can query pg_roles with `SELECT rolname, rolcanlogin, rolconnlimit, rolpassword 
FROM pg_roles WHERE rolname = 'rob`. The password doesn't show for security reasons, but we can modify the ending of the 
query to get a little more info with `FROM pg_authid WHERE rolname = 'rob`.

The pg_hba.conf file
- This file contains login information for the cluster. There are rulesets that are sepcified to determine each time 
a login attempt is made, if the connection will be allowed or not. 
    * Order matters.... e.g. once a rule is met (top down), the rest are skipped. So make sure you have your login rules
    in the right order!
    An example rule is `host    all       rob     p1       scram-sha-256` where the syntax is `<connection-type> <database> <role> <remote-machine> <auth-method>`
    * So the rule would indicate the user rob can connect to every database coming from a host named p1, where the user and password provided
    can pass the SCRAM sha authentication method. 
- You can also merge multiple rules into a single rule, use `,` to give the same role access to multiple, but not all, databases!
- To give login permissions to a group, and groups are themselves roles, we can prefix the role name with a +, like `+book_authors`
  in order to give all the roles associated with the group login permissions. 
- If we want to allow all users except 1 to the database? pg_hba.conf would look like 
```
host db1 rob           all reject
host db1 +book_authors all scram-sha-256
```
This will reject the role rob from the dbs then allow everyone, it is order specific!

Something helpful- you can put roles into a text file that is either comma or line separated, in rejected_users.txt.
Then to reject all these users, we can use `host forumdb @rejected_users.txt   all reject` in our pg_hba file!

In summary, there is alot of ways we can create roles and cluster groups of roles together (into other roles!). We can also
monitor the login authenticity and ability for useres to access certain dbs in our cluster which is super cool.

## Interacting with the DB

Start by logging into the postgres db! `psql -d postgres`, optionally use -U if you want to use a specific role, I used my admin. 

- To create a DB, `CREATE DATABSE databasename` What does this do behind the scenes? It creates a copy of template1, and assigns the 
given DB name to the database just copied!

- To list all of the databases in the cluster, use `\l`
- To connect to another DB within the cluster `\c dbname`, then you can list all the tables in a db with `\d`
- To drop a table or database, simply use `drop table tablename` and `drop database dbname`
- If you want to copy a database, use `create database newdbname template olddb`

How can we see the databse size? Turn on expanded display `\x` and then `\l+ dbname`, or in sql `select pg_size_pretty(pg_database_size('forumdb'));`

So what is going on behind the scenes? Execute `select * from pg_database where datname='forumdb';` and you'll notice that
it gives us an oid value! Familiar from the PGDATA directory on our host filesystem right! If we go into PGDATA dir, and 
into the base directory, we see that exact oid value there! In postgresql, databases are directories!

### Managing tables

- There are 3 types of tables: temporary tables, unlogged tables, and logged tables. 
- So lets create our first table in a database, call the db whatever you want.
```
CREATE TABLE users (pk int GENERATED AWAYS AS IDENTITY
	,username text NOT NULL
	,gecos text
	,email text NOT NULLL
	,PRIMARY KEY ( pk )
	, UNIQUE ( username )
	);
```
 Then perform `\d users` to see what we created! More on indexes later!
 
 After we've experimented with this, go ahead and drop the table `drop table users;`
 
 Back to Create table... there are lots of useful options (IF NOT EXISTS, TEMP, UNLOGGED, etc etc. )
 
 - To create a temp table `create temp table ......`. which will only be available in the session it was created in.
 - We can also have a table visible only within our transaction.
    - A session is a set of transactions, and each session is isolated, and a transaction is isolated from everything else. 
    - Whatever happens in a transaction can not be seen from outside the transaction until the transaction ends. 
    - To start a transaction `begin work;` If you create a table, do work, then `commit work`, you won't be able to see your table anymore, as long as 
    during table creation, you included `on commit drop` at the end of the statment. 
    
Unlogged tables are much faster than classic (logged) tables, but they are not crash safe!
- Use `create unlogged table .....` to create one. This can be a good alternative to having fast permanent and temporary tables!

Behind the scenes, where are tables stored? within the base/{oid} directory! In postgresql, tables are stored in 1 or more files!
- Each file will be a max 1gb in size, and the names of files after the first one will be named {oid}.1, {oid}.2 and so on...

## Table manipulation... finally!
- Everything before this sectionw as realted to admin, setup, stuff like that. Now how to we work with data in our db? The fun part!

To insert `insert into dbname (vcolumn names here) value (actual values here!);` fill in the parameters you have!

- Null values, always important!! Null values can come up all over the place, and in postgresql they mean that a value doesn't
exist in the database. By default, null values just show up blank in postgres :/ run `\pset null NULL` to show null values as 'NULL'

- use `is null` and `is not null` in queries to work with null values in your data
- Or put the nulls last! `select * from categories order by description NULLS last` However, this is done by default.

Creating a table from another table `create temp table temp_categories as select * from categories;`

Updating data! `update temp_categories set title='peach' where pk = 14;`

And of course, deleting data `delete from temp_categories where pk=10;`
- If we want to delete all the data in the table, just use truncate. `truncate table temp_categories ;`

The select statement can be used to filter our results.
- Using the like clause, we can find records with titles that start with a with `select * from categories where title like 'a%';` We can 
also use this in a similar way to find all titles that end with e, for example.
- Be careful, because like searches are case-sensitive!

- The `coalesce` function takes two or more parameters returns the first value that is not null. So `select coalesce(NULL,'test');` 
will return test because the first arg is null, and test is not null. 
- Now for the expression `select description,coalesce(description,'No description') from categories order by 1;` which will return 2 columns,
the first will be the description in the db, then the coalesce function will be the description if it is not null, otherwise it will 
show the string No Description because that is clearly not null. 
- The distinct keyword is used to return only distinct values, such as `select distinct coalesce(description,'No description') as description from categories order by 1;` 

Limit is used to limit the number of records returned by the query while offset skips a specific number of rows returned by the query, it is where to start. 
- `elect * from categories order by pk limit 2;`
- `select * from categories order by pk offset 1 limit 1;`

A cool function of the limit function is we can create a new table based off a current one, limit 0 and just keep the data structure, no data!

Subqueries are also very useful, such as `select pk,title,content,author,category from posts where category in (select pk from categories where title ='orange');` where
the subquery is `select pk from categories where title ='orange'`

### Partitioning!

Partitioning can be a very powerful way to split data up in your database betweeen "sub-tables" with specific data. There are different kinds of partitioning: 
1. Range Partitioning
2. List Paritioning
3. Hash Partitioning

A key idea is table inheritance. This is where a a parent table will have access to all records in a child table. But a child table will not have access to the records in the parent table, only it's own table and any child tables. 

To do this we simply perform `create table table_b () inherits (table_a);`
If we want to perform a check on a partition when creating children, we can do `CREATE TABLE part_tags_level_0 ( CHECK(level = 0 )) INHERITS (part_tags);`

The check command does 2 things -> It prunes the data correctly and makes sure no incorrect data can get into the child tables! Pretty nifty. It also makes the constraint_exclusion feature in postgres possible. If we query `select * from part_tags where level = 1` we will only query the child table where check = 1. Also a big performance improver. 

So we can go even further and create some a trigger, that will call a function whenever we insert into the parent table, that will correctly insert the data into the right child table. We create the function like 
```
postgres_playground=> create or replace function insert_part_tags () returns trigger as
postgres_playground-> $$
postgres_playground$> begin
postgres_playground$> if NEW.level = 0 then
postgres_playground$> insert into part_tags_level_0 values (NEW.*);
postgres_playground$> elsif NEW.level = 1 then
postgres_playground$> insert into part_tags_level_1 values (NEW.*);
postgres_playground$> else
postgres_playground$> raise exception 'Error in part_tags level out of range';
postgres_playground$> end if;
postgres_playground$> return null;
postgres_playground$> end;
postgres_playground$> $$
postgres_playground-> language 'plpgsql';` and we can create the trigger with `create trigger insert_part_tags_trigger before insert on part_tags for each row execute procedure insert_part_tags();
```

The trigger will move the record into the child table and return null to the parent table, so no one record will be inserted in the parent table. Thanks inheritance! If we insert some data, then look at it with `select * from part_tags;` we will see all the records we inserted. Aren't they only supposed to be in the child tables? YES! However this query will take everything from the child tables. To see what's only in the parent table we can use `select * from only part_tags;` Notice the only part, so we only take from the table. To see the child tables, which only have access to their it's own table data and any children, so we can do `select * from part_tags_level_0;`.


When we want to update an entry to stay within it's current table or delete an entry, we can simply do that from the parent table and it will be updated or removed properly from the specific partition. Now the problem comes into play when we update an entry that will cause it to need to switch which child table to be in. In order to do this, we need a new function, and triggers on each of our child tables, we can do this like 
```
CREATE OR REPLACE FUNCTION update_part_tags() RETURNS TRIGGER AS
$$
BEGIN
 IF (NEW.level != OLD.level) THEN
     DELETE FROM part_tags where pk = OLD.PK;
     INSERT INTO part_tags values (NEW.*);
 END IF;
 RETURN NULL;
END;
$$
LANGUAGE 'plpgsql';

CREATE TRIGGER update_part_tags_trigger BEFORE UPDATE ON part_tags_level_0 FOR EACH ROW EXECUTE PROCEDURE update_part_tags();
CREATE TRIGGER update_part_tags_trigger BEFORE UPDATE ON part_tags_level_1 FOR EACH ROW EXECUTE PROCEDURE update_part_tags();
```

This shows how we can partition tables and data using table inheritance, lots of use cases.

To drop a parent and its child tables, `drop table if exists part_tags cascade`

### Declarative Partitioning
We will start with how to perform list partitioning. We create a table using list partitioning 
```
CREATE TABLE part_tags (
 pk INTEGER NOT NULL DEFAULT nextval('part_tags_pk_seq') , 
 level INTEGER NOT NULL DEFAULT 0,
 tag VARCHAR (255) NOT NULL,
 primary key (pk,level)
)
PARTITION BY LIST (level);
```
It's important to understand that the field used to partition must be the primary key or be part of the pk. 

Now we can make the child tables, similar to inheritance but different. 
```
CREATE TABLE part_tags_level_0 PARTITION OF part_tags FOR VALUES IN (0);
CREATE TABLE part_tags_level_1 PARTITION OF part_tags FOR VALUES IN (1);
```

I'm also going to add indexes on the parrent and child tables usning the GIN method:
`CREATE INDEX part_tags_tag on part_tags using GIN (tag gin_trgm_ops);` and make sure to create the extension `CREATE EXTENSION pg_trgm;`

So try and insert some data and then query around in the DB for it! Notice how we have every entry in the parent table, but the partitions are separated out correctly. Let's start fresh and try Range partitioning. `drop table if exists part_tags cascade`

### Range Partitioning
Range partitioning is pretty self explanatory, we partiton based on a range. Here we will use a data range, something applicable to so many services. So let's create our parent table and child tables!

```
CREATE TABLE part_tags (
     pk INTEGER NOT NULL DEFAULT nextval('part_tags_pk_seq'),
     ins_date date not null default now()::date,
     tag VARCHAR (255) NOT NULL,
     level INTEGER NOT NULL DEFAULT 0,
     primary key (pk,ins_date)
)
PARTITION BY RANGE (ins_date);

CREATE TABLE part_tags_date_01_2020 PARTITION OF part_tags FOR VALUES FROM ('2020-01-01') TO ('2020-01-31');
CREATE TABLE part_tags_date_02_2020 PARTITION OF part_tags FOR VALUES FROM ('2020-02-01') TO ('2020-02-28');
```
Add a gin index if you'd like. For this example it isn't super necessary but for large datasets, it's a must. SO insert some data and query to see if all is as expected! 

Partition Maintenance:
We will want to do lots of things to our partitioned data, such as add a partition or delete a partition without resetting the whole thing. To add a new partition at any time we simply perform `CREATE TABLE part_tags_date_05_2020 PARTITION OF part_tags FOR VALUES FROM ('2020-05-01') TO ('2020-05-30');`

To detach a partition, we can perform `ALTER TABLE part_tags DETACH PARTITION part_tags_date_05_2020 ;`

To attach an already existing table to the parent table, keep in mind this may be rare and should align with the data structure in the parent table: `ALTER TABLE part_tags ATTACH PARTITION part_tags_already_exists FOR VALUES FROM ('1970-01-01') TO ('2019-12-31');`

### Triggers and Rules
I'm doing this a little out of order but that's ok. We can use server side programming techniques to perform actions for us on the DB side vs the client side.

We can use functions and various languages to create server side programs and code! Pretty cool that we can use some languages like java and python in a postgresql server. Functions are defined as 
```
CREATE FUNCTION function_name(p1 type, p2 type,p3 type, ....., pn type)    function name and params here
 RETURNS type AS                                                           specify the return the data type 
BEGIN                                                                      
 -- function logic                                                         Put our code here 
END;
LANGUAGE language_name                                                     define the language that was used
```

SQL fuctions are the easiest way to write functions in postgres and we can use any sql command inside them. An easy example to follow -> 
```
CREATE OR REPLACE FUNCTION my_sum(x integer, y integer) 
RETURNS integer AS 
$$
 SELECT x + y;
$ 
LANGUAGE SQL;
```
The $$ can be considered lables, this is where the code is placed between. We can call this function with `select my_sum(1,2);`

To return a list of elements we replace the return type as a `setof integer` using integer for example. But we can return a set of anything we want! Can a function return a table? Yes! Kinda weird tho... `returns table (ret_key integer,ret_title text) AS`

We can go in depth with the PL/pgSQL language as well, but for now I'm just going to stick with my PL/SQL language as it serves my purpose right now. I'm sure I'll need it in the future! (PL/pgSQL can do all kinds of loops/exception handling/etc etc)


So now we can talk about triggers and rules!
- Rules are simple event handlers, they modify the flow of an event. "If we are given an event, what can we do when certain conditions occur"
- To create a rule we follow the syntax `CREATE [ OR REPLACE ] RULE name AS ON event TO table [ WHERE condition ]  DO [ ALSO | INSTEAD ] { NOTHING | command | ( command ; command ... ) }`
- Example! `create or replace rule r_tags1 
                as on INSERT to tags 
                where NEW.tag ilike 'a%' DO ALSO 
                insert into a_tags(pk,tag,parent)values (NEW.pk,NEW.tag,NEW.parent);`
               Everytime a record is inserted to tags where the new tag starts with the letter a, also insert the record into a_tags.
- We can use the DO INSTEAD NOTHING and DO INSTEAD to perform actions instead of the original action provided it meets the requirements in the rule. 

What's the difference between rules and triggers? Well rules will fire before the trigger, always. Also, with triggers it is possible to handle insert update and delete and truncate statements before the happen or after they have happened. The trigger syntax is:
```
CREATE [ CONSTRAINT ] TRIGGER name { BEFORE | AFTER | INSTEAD OF } { event [ OR ... ] }
 ON table_name
 [ FROM referenced_table_name ]
 [ NOT DEFERRABLE | [ DEFERRABLE ] [ INITIALLY IMMEDIATE | INITIALLY DEFERRED ] ]
 [ REFERENCING { { OLD | NEW } TABLE [ AS ] transition_relation_name } [ ... ] ]
 [ FOR [ EACH ] { ROW | STATEMENT } ]
 [ WHEN ( condition ) ]
 EXECUTE { FUNCTION | PROCEDURE } function_name ( arguments )

where event can be one of:

 INSERT
 UPDATE [ OF column_name [, ... ] ]
 DELETE
 TRUNCATE
```

Note that the trigger will call a function which must already be defined!

Example of a trigger: `CREATE TRIGGER trigger_name BEFORE INSERT on table_name FOR EACH ROW EXECUTE PROCEDURE function_name.`

Similar to our rule about inserts starting with a, we can use a trigger and a function:
```
The functon:
CREATE OR REPLACE FUNCTION f_tags() RETURNS trigger as
$$
BEGIN
 IF lower(substring(NEW.tag from 1 for 1)) = 'a' THEN
 insert into a_tags(pk,tag,parent)values (NEW.pk,NEW.tag,NEW.parent);
 END IF; 
 RETURN NEW;
END; 
$$
LANGUAGE 'plpgsql';

And the trigger:
CREATE TRIGGER t_tags BEFORE INSERT on new_tags FOR EACH ROW EXECUTE PROCEDURE f_tags();
```
Note how this will be executed BEFORE the insert. SO we can deal with our data before anything happens in the DB.

Just to chew on in the future, here's an example of a more complex function and trigger: 
```
CREATE OR REPLACE FUNCTION f3_tags() RETURNS trigger as
$$
BEGIN
 IF lower(substring(NEW.tag from 1 for 1)) = 'a' THEN
     insert into a_tags(pk,tag,parent)values (NEW.pk,NEW.tag,NEW.parent);
     RETURN NEW;
 ELSIF lower(substring(NEW.tag from 1 for 1)) = 'b' THEN
     insert into b_tags(pk,tag,parent)values (NEW.pk,NEW.tag,NEW.parent);
     RETURN NULL;
 ELSE
     RETURN NEW;
 END IF; 
END; 
$$
LANGUAGE 'plpgsql';

CREATE TRIGGER t3_tags BEFORE INSERT on new_tags FOR EACH ROW EXECUTE PROCEDURE f3_tags();
```
Something useful might be the TG_OP value, which stores the operation that is currently trying to be performed. Just like NEW and OLD hold the row data, PG_OP hold the operation. So we can check if the operation is as expected and handle different operations within one function!
```
CREATE OR REPLACE FUNCTION fcopy_tags() RETURNS trigger as
$$
BEGIN
IF TG_OP = 'INSERT' THEN
     IF lower(substring(NEW.tag from 1 for 1)) = 'a' THEN
         insert into a_tags(pk,tag,parent)values (NEW.pk,NEW.tag,NEW.parent);
     ELSIF lower(substring(NEW.tag from 1 for 1)) = 'b' THEN
         insert into b_tags(pk,tag,parent)values (NEW.pk,NEW.tag,NEW.parent);
     END IF;
     RETURN NEW;
 END IF;
IF TG_OP = 'DELETE' THEN
     IF lower(substring(OLD.tag from 1 for 1)) = 'a' THEN
         DELETE FROM a_tags WHERE pk = OLD.pk;
     ELSIF lower(substring(OLD.tag from 1 for 1)) = 'b' THEN
         DELETE FROM b_tags WHERE pk = OLD.pk;
     END IF;
     RETURN OLD;
END IF;
IF TG_OP = 'UPDATE' THEN
    IF (lower(substring(OLD.tag from 1 for 1)) in( 'a','b') ) THEN
         DELETE FROM a_tags WHERE pk=OLD.pk;
         DELETE FROM b_tags WHERE pk=OLD.pk;
         DELETE FROM new_tags WHERE pk = OLD.pk;
         INSERT into new_tags(pk,tag,parent) values (NEW.pk,NEW.tag,NEW.parent);
     END IF;
     RETURN NEW;
END IF;
END;
$$
LANGUAGE 'plpgsql';

CREATE TRIGGER tcopy_tags_ins 
    BEFORE INSERT on new_tags FOR EACH ROW EXECUTE PROCEDURE fcopy_tags();
CREATE TRIGGER tcopy_tags_del 
    AFTER DELETE on new_tags FOR EACH ROW EXECUTE PROCEDURE fcopy_tags();
CREATE TRIGGER tcopy_tags_upd 
    AFTER UPDATE on new_tags FOR EACH ROW EXECUTE PROCEDURE fcopy_tags();
```
This is long, but it is a complete example of how to properly configure a function and triggers, both before and after!

There are also event triggers in postgresm which means that event triggers trigger on something changing the data but not the data layout or table properties. I'm not going to get into this right now, but the docs are out there!

