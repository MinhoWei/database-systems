# Adding New Data Types to PostgreSQL
## Types in PostgreSQL

PostgreSQL has several distinct kinds of types:

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
```
-- input function
CREATE FUNCTION complex_in(cstring)
   RETURNS complex
   AS '_OBJWD_/complex'
   LANGUAGE C IMMUTABLE STRICT;
   -- IMMUTABLE means the function always returns the same result given the same input
   -- STRICT means the function will return null if any of the input is null

-- output function
-- to convert a complex type to text representation that is easier to read
CREATE FUNCTION complex_out(complex)
   RETURNS cstring
   AS '_OBJWD_/complex'
   LANGUAGE C IMMUTABLE STRICT;

-- now, we can create the type
CREATE TYPE complex (
   internallength = 16, -- a complex type occupies 16 bytes (real part + imagiary part)
   input = complex_in,
   output = complex_out,
   alignment = double
);

-- complex_add function (also in complex.c)
CREATE FUNCTION complex_add(complex, complex)
   RETURNS complex
   AS '_OBJWD_/complex'
   LANGUAGE C IMMUTABLE STRICT;

CREATE OPERATOR + (
   leftarg = complex,
   rightarg = complex,
   procedure = complex_add,
   commutator = +
);

-- create aggregate function (sum)
CREATE AGGREGATE complex_sum (
   sfunc = complex_add,
   basetype = complex,
   stype = complex,
   initcond = '(0,0)'
);

-- define the required operators
CREATE FUNCTION complex_abs_lt(complex, complex) RETURNS bool
   AS '_OBJWD_/complex' LANGUAGE C IMMUTABLE STRICT;
CREATE FUNCTION complex_abs_le(complex, complex) RETURNS bool
   AS '_OBJWD_/complex' LANGUAGE C IMMUTABLE STRICT;
CREATE FUNCTION complex_abs_eq(complex, complex) RETURNS bool
   AS '_OBJWD_/complex' LANGUAGE C IMMUTABLE STRICT;
CREATE FUNCTION complex_abs_ge(complex, complex) RETURNS bool
   AS '_OBJWD_/complex' LANGUAGE C IMMUTABLE STRICT;
CREATE FUNCTION complex_abs_gt(complex, complex) RETURNS bool
   AS '_OBJWD_/complex' LANGUAGE C IMMUTABLE STRICT;

CREATE OPERATOR < (
   leftarg = complex, rightarg = complex, procedure = complex_abs_lt,
   commutator = > , negator = >= ,
   restrict = scalarltsel, join = scalarltjoinsel
);
CREATE OPERATOR <= (
   leftarg = complex, rightarg = complex, procedure = complex_abs_le,
   commutator = >= , negator = > ,
   restrict = scalarlesel, join = scalarlejoinsel
);
CREATE OPERATOR = (
   leftarg = complex, rightarg = complex, procedure = complex_abs_eq,
   commutator = = ,
   -- leave out negator since we didn't create <> operator
   -- negator = <> ,
   restrict = eqsel, join = eqjoinsel
);
CREATE OPERATOR >= (
   leftarg = complex, rightarg = complex, procedure = complex_abs_ge,
   commutator = <= , negator = < ,
   restrict = scalargesel, join = scalargejoinsel
);
CREATE OPERATOR > (
   leftarg = complex, rightarg = complex, procedure = complex_abs_gt,
   commutator = < , negator = <= ,
   restrict = scalargtsel, join = scalargtjoinsel
);

-- create the support function
-- compares two values of the complex type and determines their order
CREATE FUNCTION complex_abs_cmp(complex, complex) RETURNS int4
   AS '_OBJWD_/complex' LANGUAGE C IMMUTABLE STRICT;

-- now we can make the operator class
CREATE OPERATOR CLASS complex_abs_ops
    DEFAULT FOR TYPE complex USING btree AS
        OPERATOR        1       < ,
        OPERATOR        2       <= ,
        OPERATOR        3       = ,
        OPERATOR        4       >= ,
        OPERATOR        5       > ,
        FUNCTION        1       complex_abs_cmp(complex, complex);

-----------------------------
-- Using the new type:
--	user-defined types can be used like ordinary built-in types.
-----------------------------
CREATE TABLE test_complex (
	a	complex,
	b	complex
);

INSERT INTO test_complex VALUES ('(1.0, 2.5)', '(4.2, 3.55)');
INSERT INTO test_complex VALUES ('(33.0, 51.4)', '(100.42, 93.55)');

SELECT * FROM test_complex;

SELECT (a + b) AS c FROM test_complex;

-- you may find it useful to cast the string to the desired
-- type explicitly. :: denotes a type cast.
SELECT  a + '(1.0,1.0)'::complex AS aa,
        b + '(1.0,1.0)'::complex AS bb
   FROM test_complex;

SELECT complex_sum(a) FROM test_complex;

INSERT INTO test_complex VALUES ('(56.0,-22.5)', '(-43.2,-0.07)');
INSERT INTO test_complex VALUES ('(-91.9,33.6)', '(8.6,3.0)');

CREATE INDEX test_cplx_ind ON test_complex
   USING btree(a complex_abs_ops);

SELECT * from test_complex where a = '(56.0,-22.5)';
SELECT * from test_complex where a < '(56.0,-22.5)';
SELECT * from test_complex where a > '(56.0,-22.5)';

-- clean up the example
DROP TABLE test_complex;
DROP TYPE complex CASCADE;
```

### complex.c
```
#include "postgres.h"
#include "fmgr.h"
#include "libpq/pqformat.h"		/* needed for send/recv functions */

PG_MODULE_MAGIC;

typedef struct Complex
{
	double		x;
	double		y;
}Complex;


/*****************************************************************************
 * Input/Output functions
 *****************************************************************************/
// input function
PG_FUNCTION_INFO_V1(complex_in);

Datum // a generic data type PostgreSQL uses to represent values of different types
complex_in(PG_FUNCTION_ARGS)
{
	char	   *str = PG_GETARG_CSTRING(0);
	double		x,
				y;
	Complex    *result;

    // parse the input string
	// hoping to have value format (real, imaginary)
	if (sscanf(str, " ( %lf , %lf )", &x, &y) != 2)
		ereport(ERROR,
				(errcode(ERRCODE_INVALID_TEXT_REPRESENTATION),
				 errmsg("invalid input syntax for type %s: \"%s\"",
						"complex", str)));

	result = (Complex *) palloc(sizeof(Complex));
	result->x = x;
	result->y = y;
	PG_RETURN_POINTER(result);
}

// output function
PG_FUNCTION_INFO_V1(complex_out);

Datum
complex_out(PG_FUNCTION_ARGS)
{
	Complex    *complex = (Complex *) PG_GETARG_POINTER(0);
	char	   *result;

    // psprintf: allocate memory automatically
	result = psprintf("(%g,%g)", complex->x, complex->y);
	PG_RETURN_CSTRING(result);
}

// addition
PG_FUNCTION_INFO_V1(complex_add);

Datum
complex_add(PG_FUNCTION_ARGS)
{
	Complex    *a = (Complex *) PG_GETARG_POINTER(0); // the first argument
	Complex    *b = (Complex *) PG_GETARG_POINTER(1); // the second argument
	Complex    *result;

    // palloc stands for 'PostgreSQL allocator'
	result = (Complex *) palloc(sizeof(Complex));
	result->x = a->x + b->x;
	result->y = a->y + b->y;
	PG_RETURN_POINTER(result);
}

// compare two complex numbers based on their magnitudes
#define Mag(c)	((c)->x*(c)->x + (c)->y*(c)->y)

static int
complex_abs_cmp_internal(Complex * a, Complex * b)
{
	double		amag = Mag(a),
				bmag = Mag(b);

	if (amag < bmag)
		return -1;
	if (amag > bmag)
		return 1;
	return 0;
}

// operators
PG_FUNCTION_INFO_V1(complex_abs_lt);

Datum
complex_abs_lt(PG_FUNCTION_ARGS)
{
	Complex    *a = (Complex *) PG_GETARG_POINTER(0);
	Complex    *b = (Complex *) PG_GETARG_POINTER(1);

	PG_RETURN_BOOL(complex_abs_cmp_internal(a, b) < 0);
}

PG_FUNCTION_INFO_V1(complex_abs_le);

Datum
complex_abs_le(PG_FUNCTION_ARGS)
{
	Complex    *a = (Complex *) PG_GETARG_POINTER(0);
	Complex    *b = (Complex *) PG_GETARG_POINTER(1);

	PG_RETURN_BOOL(complex_abs_cmp_internal(a, b) <= 0);
}

PG_FUNCTION_INFO_V1(complex_abs_eq);

Datum
complex_abs_eq(PG_FUNCTION_ARGS)
{
	Complex    *a = (Complex *) PG_GETARG_POINTER(0);
	Complex    *b = (Complex *) PG_GETARG_POINTER(1);

	PG_RETURN_BOOL(complex_abs_cmp_internal(a, b) == 0);
}

PG_FUNCTION_INFO_V1(complex_abs_ge);

Datum
complex_abs_ge(PG_FUNCTION_ARGS)
{
	Complex    *a = (Complex *) PG_GETARG_POINTER(0);
	Complex    *b = (Complex *) PG_GETARG_POINTER(1);

	PG_RETURN_BOOL(complex_abs_cmp_internal(a, b) >= 0);
}

PG_FUNCTION_INFO_V1(complex_abs_gt);

Datum
complex_abs_gt(PG_FUNCTION_ARGS)
{
	Complex    *a = (Complex *) PG_GETARG_POINTER(0);
	Complex    *b = (Complex *) PG_GETARG_POINTER(1);

	PG_RETURN_BOOL(complex_abs_cmp_internal(a, b) > 0);
}

PG_FUNCTION_INFO_V1(complex_abs_cmp);

Datum
complex_abs_cmp(PG_FUNCTION_ARGS)
{
	Complex    *a = (Complex *) PG_GETARG_POINTER(0);
	Complex    *b = (Complex *) PG_GETARG_POINTER(1);

	PG_RETURN_INT32(complex_abs_cmp_internal(a, b));
}
```
