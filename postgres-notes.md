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

 

    



