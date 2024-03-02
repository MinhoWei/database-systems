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
