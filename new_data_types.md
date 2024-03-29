# Adding New Data Types to PostgreSQL
## Types in PostgreSQL

**base types**: defined via C functions, and providing genuine new data types; built-in types such as integer, date and varchar(n) are base types; users can also define new base types;

**domains**: data types based on a constrained version of an existing data type;

**enumerated types**: defined by enumerating the values of the type; values are specified as a list of strings, and an ordering is defined on the values based on the order they appear in the list;

**composite types**: these are essentially tuple types; a composite type is composed of a collection of named fields, where the fields can have different types; a composite type is created implicitly whenever a table is defined, but composite types can also be defined without the need for a table;

**polymorphic types**: define classes of types (e.g. anyarray), and are used primarily in the definition of polymorphic functions;

**pseudo types**: special types (such as trigger) used internally by the system; polymorphic types are also considered to be pseudo-types.

## Create a Type for Days of a Week
We want to create a new type for days of a week (suppose Monday is the first day of the week). There are two ways to do this:
```
create domain Days1 as varchar(9)
       check (value in ('Monday','Tuesday','Wednesday',
                        'Thursday','Friday','Saturday','Sunday'));
```
```
create type Days2 as enum
       ('Monday','Tuesday','Wednesday',
        'Thursday','Friday','Saturday','Sunday');
```
Note: A domain is essentially a data type with constraints. An enum (enumerated type) is a data type that comprises a static, ordered set of values. The main differences are these: Domains allow for more complex constraints and can be based on any data type, not just character strings. Enums are limited to the specified list of values. Also, domains are based on existing data types and carry the same storage characteristics, whereas enums are stored as integers and have their own distinct storage characteristics.

Also, Days2 type is more appropriate in this case. Because enumerated type is a _ordered set_ of values.

## Define a New Base Data Type
In order to define a new base data type, a user needs to provide:

 - input and output functions (in C) for values of the type
 - C data structure definitions to represent type values internally
 - an SQL definition for the type, giving its length, alignment and i/o functions
 - SQL definitions for operators on the type
 - C functions to implement the operators

### complex.source
Creating a new type: We are going to create a new
