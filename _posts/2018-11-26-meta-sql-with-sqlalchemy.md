---
title: "Meta SQL with Python &amp; SQLAlchemy"
published: true
---

\* Sorry about the mangled codeblocks. In the meantime, this post is still available [over here](https://github.com/gregn610/plpgsql/wiki/meta-sql-with-python-&-sqlalchemy)


## Meta SQL Intro
I’m a big fan of meta SQL, what I call it when you use SQL to write SQL. Something like:

~~~ plpgsql

SELECT
    'INSERT INTO zzzdba.rowcount_census SELECT now(), ' ||
    table_name || ', count(*) AS kount FROM '||
    table_schema ||'.' || table_name || ';' AS metasql
FROM
    information_schema.tables
WHERE
    table_schema = 'accounts'
AND table_name ~ 'current_account_\d\d\d\d'
;
    
~~~

returning:
{% raw %}
~~~ txt
-----------------------------------------------------------------------------------------------------
  INSERT INTO zzzdba.rowcount_census                                                                +
         SELECT now(), 'current_account_2017', count(*) AS kount FROM accounts.current_account_2017;+

  INSERT INTO zzzdba.rowcount_census                                                                +
         SELECT now(), 'current_account_2010', count(*) AS kount FROM accounts.current_account_2010;+

  INSERT INTO zzzdba.rowcount_census                                                                +
         SELECT now(), 'current_account_2011', count(*) AS kount FROM accounts.current_account_2011;+

/* SNIP */

  INSERT INTO zzzdba.rowcount_census                                                                +
         SELECT now(), 'current_account_2018', count(*) AS kount FROM accounts.current_account_2018;+

  INSERT INTO zzzdba.rowcount_census                                                                +
         SELECT now(), 'current_account_2019', count(*) AS kount FROM accounts.current_account_2019;+

(10 rows)
~~~
{% endraw %}

Here a SQL query is using the database schema to create other SQL queries that in turn provide the desired results. It’s a handy trick and can be quickly and easily used for ad-hoc queries. 

But meta SQL is often taken further. Using a build process, the output is sent to files and a lot of code is very quickly generated. This technique can be used to create hundreds of CRUD stored procedures for a .Net data access layer, for generating SQL code for data warehouse slowly changing dimensions or perhaps access control views. And it’s when meta SQL is used in a more systemic way that problems can arise. Typically this meta SQL does a lot of string manipulation that isn’t easy to iterate on, and quickly becomes unwieldy and difficult to maintain. A first improvement might be to move from string concatenation to printf style formatted strings.

## Solution 1 
### SQL:
{% raw %}
~~~ plpgsql
SELECT
    format($$ INSERT INTO zzzdba.rowcount_census
    SELECT now(), '%1s', count(*) AS kount FROM %2I.%3I;$$,
        table_name,
        table_schema,
        table_name)
FROM
    information_schema.tables
WHERE
    table_schema = 'accounts'
AND table_name ~ 'current_account_\d\d\d\d'
;
~~~
{% endraw %}

Here, printf style formatting has been used to remove the string concatenation and quotation jiggery-pokery of the previous example, but readability is subjectively worse. Even with this approach though, things can quickly get unmanageable. Imagine the scenario where some or all of the column names and their data types are needed in the meta SQL.

The next approach could be to use a scripting language and a templating library to produce the SQL, eg. [Python](https://www.python.org/) and [Jinja2](http://http://jinja.pocoo.org/ ).

## Solution 2 
### Python:
{% raw %}
~~~ python3
import os
import psycopg2
from psycopg2.extras import NamedTupleCursor     # so we can use column names
from jinja2 import Environment, FileSystemLoader

# just load templates for the current directory for demo purposes
fs_loader = FileSystemLoader(os.path.dirname(__file__))
env = Environment(loader=fs_loader)
template = env.get_template('40_psycopg2.j2')

# connect to a database
with psycopg2.connect(host='localhost', database='postgres', user='postgres', password='postgres') as conn:
    cur = conn.cursor(cursor_factory=NamedTupleCursor)
    sql = '''
SELECT
    table_schema, table_name, column_name, data_type
FROM
    information_schema.columns
WHERE
    table_schema = 'accounts'
    AND table_name = 'current_account_2018'
ORDER BY
    table_schema, table_name, ordinal_position; '''
    # run the SQL query
    cur.execute(sql)
    rows = cur.fetchall()

# render the results, would usually be to file
ddl = template.render(dbd=rows)
print(ddl)
~~~
{% endraw %}

### Jinja2:
{% raw %}
~~~ jinja2
CREATE OR REPLACE VIEW app.{{ dbd[0].table_schema }}_{{ dbd[0].table_name }}_v AS
SELECT
{%- for col in dbd %}
    {{ col.table_name }}.{{col.column_name}}{% if not loop.last %},{% endif %}
{%- endfor %}
FROM {{ dbd[0].table_schema }}.{{ dbd[0].table_name }}
WHERE fn_access_control(‘{{ dbd[0].table_schema }}.{{ dbd[0].table_name }}’) = CURRENT_USER
;
~~~
{% endraw %}

### Output
{% raw %}
~~~ plpgsql
CREATE OR REPLACE VIEW app.accounts_current_account_2018_v AS
SELECT
    current_account_2018.id,
    current_account_2018.account_number,
    current_account_2018.account_type_id,
    current_account_2018.balance
FROM accounts.current_account_2018
WHERE fn_access_control(‘accounts.current_account_2018’) = CURRENT_USER
;
~~~
{% endraw %}

In this solution, the meta SQL and the target SQL template have been broken out into their own files and can then be version controlled individually. The _FOR_ loop over the list of column names and the _IF_ statement show how the template can benefit from being able to use the power of flow control and expressions provided by Jinja2. The intention of the template shines through, it's more legible and debuggable without all that string concatenation; more so with some J2 syntax highlighting extensions in your IDE of choice. 

But look at how meta data is passed into the template. In this example, the variable “_dbd_” is used to pass a list of named records provided by psycopg2’s _NamedTupleCursor_. This has the nice benefit that the column names from the SQL query are usable inside the template and keeps the template fairly comprehensible. The downside is that we’ve bound the template to an arbitrary data structure of a list of tuples, “_dbd_”. Consider how that data structure would need to change when new requirements come along. For example, what if the template needs to know which column(s) are the primary key and which are foreign keys. Or what if two tables are needed in the template to write out a JOIN clause. Very quickly, the data structures grow and become convoluted, effectively creating an undocumented API used in the templates and you're spending a lot of effort keeping track.

But there is an API readily available in the form of [SQLAlchemy](https://www.sqlalchemy.org/features.html ). SQLAlchemy comes in two parts, the core and the ORM. It’s the core that provides a structured, documented and battle tested core and that provides just about everything needed to replace the meta-SQL with database reflection.

## Solution 3 
### Python:
{% raw %}
~~~ python3
import os
from sqlalchemy import create_engine, MetaData, Table
from jinja2 import Environment, FileSystemLoader

# just load templates for the current directory for demo purposes
fs_loader = FileSystemLoader(os.path.dirname(__file__))
env = Environment(loader=fs_loader)
template = env.get_template('50_sqlalchemy_table.j2')

pg_engine = create_engine('postgresql://postgres:postgres@localhost:5432/postgres')

meta = MetaData(schema="accounts")
meta.reflect(bind=pg_engine)
acc2018_table = meta.tables['accounts.current_account_2018']

ddl = template.render(tbl=acc2018_table)
print(ddl)
~~~
{% endraw %}
### Jinja2:
{% raw %}
~~~ jinja2
CREATE OR REPLACE VIEW app.{{ tbl.fullname | replace('.','_') }}_v AS
SELECT
{%- for col in tbl.columns %}
    {{ tbl.fullname}}.{{ col.name }}{% if not loop.last %},{% endif %}
{%- endfor %}
FROM {{ tbl.fullname | replace('.','_') }}
INNER JOIN accounts.account_type USING (account_type_id)
WHERE fn_access_control(‘{{ tbl.fullname }}’) = CURRENT_USER
;
~~~
{% endraw %}
### Output
{% raw %}
~~~ plpgsql
CREATE OR REPLACE VIEW app.accounts_current_account_2018_v AS
SELECT
    accounts.current_account_2018.id,
    accounts.current_account_2018.account_number,
    accounts.current_account_2018.account_type_id,
    accounts.current_account_2018.balance
FROM accounts_current_account_2018
INNER JOIN accounts.account_type USING (account_type_id)
WHERE fn_access_control(‘accounts.current_account_2018’) = CURRENT_USER
;
~~~
{% endraw %}

Now the psycopg2 library and the information schema meta SQL query have been replaced with "_meta.refect()_" provided by SQLAlchemy. The full metadata of the _account.current_account_2018_ table is passed into the template. Inside the template, there is ready access to the table’s name, the list of it’s columns and all of their details via _tbl.columns_. Similarly, the table’s primary key details via _tbl.primary_key_and the foreign key details via _tbl.foreign_keys_, indexes etc are all available inside the template in an orderly fashion. All nicely documented by [Sqlalchemy](https://docs.sqlalchemy.org/en/latest/core/metadata.html) or explorable from your favourite IDE debugger. Add a quick _FOR_  loop to the python script and you could have the template rendered for every table in the database or any subset thereof.

## Conclusion:
Which solution is most appropriate will vary from use case to case. But if you find meta SQL escaping the occasional ad-hoc query, getting into your build or CI process, I hope the techniques above can be useful.

/g 

