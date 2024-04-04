# The PostgreSQL Catalog
## Important PostgreSQL System Catalog Tables
Note: Details on the other tables, and complete details of the given tables, are available in the PostgreSQL documentation: *https://www.postgresql.org/docs/16/catalogs.html*.

![](https://github.com/MinhoWei/database-systems/blob/main/catalog1.png)
These are tables that contain metadata about the database. Each table or view stores information about a different aspect of the database, such as users, databases, tables, columns, and data types.

**pg_authid**: This contains information about database roles. Roles can be thought of as users, with columns for role name, password, and privileges such as creating a database, role, or if it can login.

**pg_database**: Stores information about the databases within the PostgreSQL instance, including their names, encoding, and access privileges.

**pg_namespace**: Holds data about the namespaces (also known as schemas) in the database. Each namespace contains objects like tables and functions, and is owned by a particular role.

**pg_class**: Contains data about the database classes, which in PostgreSQL are primarily tables and indexes. This includes the table name, the namespace it belongs to, its owner, its physical storage file, and various other properties like its visibility and access methods.

**pg_attribute**: Details information about table columns. Each entry represents a column in a table with its name, data type, and other attributes such as whether it's a primary key, not null, etc.

**pg_type**: Stores information about data types in PostgreSQL. Custom data types, along with standard ones like integer or varchar, are listed here with their properties.

## schema() Function
The following is the code for schema() function, which prints out the schemas of tables in a database.
```
create or replace function schema() returns setof text
as $$
declare
        rec record;
        rel text := '';
        att text := '';
        out text := '';
	len integer := 0;
begin
	for rec in
		select relname,attname
		from   pg_class t, pg_attribute a, pg_namespace n
		where  t.relkind='r'           -- regular tables
			and t.relnamespace = n.oid
			and n.nspname = 'public'   -- in the 'public' schema
			and attrelid = t.oid
			and attnum > 0
		order by relname,attnum
	loop
		if (rec.relname <> rel) then
		    -- finalizes the output string for the previous table
			-- starts processing the new one
			if (rel <> '') then
				out := rel || '(' || att || ')';
				return next out;
			end if;
			rel := rec.relname;
			att := '';
			len := 0;
		end if;
		if (att <> '') then
			att := att || ', ';
			len := len + 2;
		end if;
		if (len + length(rec.attname) > 70) then
			att := att || E'\n        ';
			len := 0;
		end if;
		att := att || rec.attname;
		len := len + length(rec.attname);
	end loop;
	-- deal with last table
	if (rel <> '') then
		out := rel || '(' || att || ')';
		return next out;
	end if;
end;
$$ language plpgsql;
```
The usage and output are as follows:

![](https://github.com/MinhoWei/database-systems/blob/main/catalog2.png)

The following is a modified version of schema(), which also prints out the data type of each attribute.
```
-- create a new type SchemaTuple
drop type if exists SchemaTuple cascade;
create type SchemaTuple as ("table" text, "attributes" text);

create or replace function schema1() returns setof SchemaTuple
as $$
declare
        rec   record;
        rel   text := '';
	    attr  text := '';
        attrs text := '';
        out   SchemaTuple;
	    len   integer := 0;
begin
	for rec in
		select r.relname, a.attname, t.typname, a.atttypmod
		from   pg_class r
			join pg_namespace n on (r.relnamespace = n.oid)
			join pg_attribute a on (a.attrelid = r.oid)
			join pg_type t on (a.atttypid = t.oid)
		where  r.relkind='r'
			and n.nspname = 'public'
			and attnum > 0
		order by relname,attnum
	loop
		if (rec.relname <> rel) then
			if (rel <> '') then
				out."table" := rel;
				out.attributes := attrs;
				return next out;
			end if;
			rel := rec.relname;
			attrs := '';
			len := 0;
		end if;
		if (attrs <> '') then
			attrs := attrs || ', ';
			len := len + 2;
		end if;
		if (rec.typname = 'varchar') then
			rec.typname := 'varchar('||(rec.atttypmod-4)||')';
		elsif (rec.typname = 'bpchar') then
			rec.typname := 'char('||(rec.atttypmod-4)||')';
		elsif (rec.typname = 'int4') then
			rec.typname := 'integer';
		elsif (rec.typname = 'float8') then
			rec.typname := 'float';
		end if;
		attr := rec.attname||':'||rec.typname;
		if (len + length(attr) > 50) then
			attrs := attrs || E'\n';
			len := 0;
		end if;
		attrs := attrs || attr;
		len := len + length(attr);
	end loop;
	-- deal with last table
	if (rel <> '') then
		out."table" := rel;
		out.attributes := attrs;
		return next out;
	end if;
end;
$$ language plpgsql;
```
The usage and output are as follows:

![](https://github.com/MinhoWei/database-systems/blob/main/catalog3.png)
