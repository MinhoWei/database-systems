# The PostgreSQL Catalog
## Important PostgreSQL System Catalog Tables
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
        rec record;              -- record: hold a row of data (a composite data type)
        current_rel text := '';  -- the current table name
        att text := '';          -- a set of column names
        out text := '';          -- the output line
	    len integer := 0;        -- keep track of the output length
begin
	for rec in
		select relname,attname
		from   pg_class t, pg_attribute a, pg_namespace n
		where  t.relkind='r'           -- include only ordinary tables
			and t.relnamespace = n.oid
			and n.nspname = 'public'
			and attrelid = t.oid
			and attnum > 0             -- column number > 0
		order by relname,attnum
	loop
	    -- if relation name is different from current relation name
		if (rec.relname <> current_rel) then
			if (current_rel <> '') then
				out := current_rel || ' (' || att || ')';
				return next out;
			end if;
			-- resets for the new table
			current_rel := rec.relname;
			att := '';
			len := 0;
		end if;

		if (att <> '') then
			att := att || ', ';
			len := len + 2;
		end if;
        
		-- handles output line length
		if (len + length(rec.attname) > 70) then
			att := att || E'\n        ';
			len := 0;
		end if;

		att := att || rec.attname;
		len := len + length(rec.attname);
	end loop;

	-- deal with last table
	if (current_rel <> '') then
		out := current_rel || ' (' || att || ')';
		return next out;
	end if;
end;
$$ language plpgsql;
```

The following is a modified version of schema(), which also prints out the data type of each attribute.

```
drop type if exists SchemaTuple cascade;
create type SchemaTuple as ("table" text, "attributes" text);

create or replace function schema1() returns setof SchemaTuple
as $$
declare
        rec   record;             -- record to hold each row returned by the query
        current_rel   text := ''; -- current table name
	    attr  text := '';         -- attribute name and type 
        attrs text := '';         -- attribute strings for a single table
        out   SchemaTuple;        -- output line
	    len   integer := 0;       -- output length
begin
	for rec in
		select r.relname, a.attname, t.typname, a.atttypmod
		from   pg_class r
			join pg_namespace n on (r.relnamespace = n.oid)
			join pg_attribute a on (a.attrelid = r.oid)
			join pg_type t on (a.atttypid = t.oid)
		where  r.relkind='r'          -- regular table
			and n.nspname = 'public'
			and attnum > 0            -- column number > 0
		order by relname,attnum
	loop
		if (rec.relname <> current_rel) then
			if (current_rel <> '') then
				out."table" := current_rel;
				out."attributes" := attrs;
				return next out;
			end if;
			current_rel := rec.relname;
			attrs := '';
			len := 0;
		end if;

		if (attrs <> '') then
			attrs := attrs || ', ';
			len := len + 2;
		end if;

        -- add the type names
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
	if (current_rel <> '') then
		out."table" := current_rel;
		out."attributes" := attrs;
		return next out;
	end if;
end;
$$ language plpgsql;
```
