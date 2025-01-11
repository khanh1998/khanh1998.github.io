---
date: '2024-11-13T13:58:54+07:00'
draft: false
title: 'Postgres Collation'
tags: ['postgres', 'database']
---
From my [previous post]({{< ref "./which-operator-is-faster-like-vs-equal.md" >}}), I realized that pattern matching operators (`LIKE`, `ILIKE`) do not utilize indexes. As I explored further, I came across the concept of collation and decided to take some notes in this post.

# Encoding
Encoding maps human-readable characters to numbers so computers can understand them. Essentially, it assigns a unique number to each character. Common encodings include UTF-8 and ASCII.

- **ASCII**: Represents 256 unique characters.
- **UTF-8**: Represents 1,112,064 characters, covering almost all characters from any language.

Most modern programming languages, such as Go, natively support UTF-8. Unlike ASCII, which uses 1 byte per character, UTF-8 uses up to 4 bytes. Strings in programming languages are typically represented as byte arrays. In ASCII, the number of bytes corresponds to the number of characters. However, this is not true for UTF-8.

In Go, iterating through a string’s byte array will produce incorrect results for non-ASCII characters. Instead, Go uses the `rune` type to represent characters.

```go
// https://go.dev/blog/strings
package main

import (
	"fmt"
	"unicode/utf8"
)

func main() {
	const nihongo = "日本語"
	fmt.Printf("string = %s: byte = %d, rune(character) = %d\n", nihongo, len(nihongo), utf8.RuneCountInString(nihongo))

	fmt.Println("The wrong way")
	for i := 0; i < len(nihongo); i++ {
		fmt.Printf("%#U starts at byte position %d\n", nihongo[i], i)
	}
	fmt.Println("The right way")
	for index, runeValue := range nihongo {
		fmt.Printf("%#U starts at byte position %d\n", runeValue, index)
	}
}
```
**Result:**
```
string = 日本語: byte = 9, rune(character) = 3
The wrong way
U+00E6 'æ' starts at byte position 0
U+0097 starts at byte position 1
U+00A5 '¥' starts at byte position 2
U+00E6 'æ' starts at byte position 3
U+009C starts at byte position 4
U+00AC '¬' starts at byte position 5
U+00E8 'è' starts at byte position 6
U+00AA 'ª' starts at byte position 7
U+009E starts at byte position 8
The right way
U+65E5 '日' starts at byte position 0
U+672C '本' starts at byte position 3
U+8A9E '語' starts at byte position 6
```

# Collation
## Why Do We Need Collation?
In programming, comparing two strings or characters often involves comparing their numeric representations from the encoding table. For example, in ASCII:

- `a` (decimal 97) > `A` (decimal 65).

However, this approach does not work well for all languages. For instance, in the Vietnamese alphabet, `a` has variations like `â`, `ă`, `á`, `à`, `ạ`, and `ả`. Comparing `â` and `b` using UTF-8’s numeric representation gives:

- `â` (decimal 226) > `b` (decimal 98).

But in Vietnamese, `â` < `b` because `â` comes earlier in the alphabet.

Additionally, determining the lowercase of `Á` requires knowing the language and region. For example, in `vi_VN.UTF-8`, the lowercase of `Á` is `á`. Collation provides rules for such comparisons.

### Vietnamese Alphabet with UTF-8 Decimal Code Points
The table below shows the order of letters in the Vietnamese alphabet and their UTF-8 decimal codes. Notice how the order does not correspond to the numeric codes:

| Letter | UTF-8 Decimal Code |
|--------|---------------------|
| A      | 65                  |
| Á      | 193                 |
| À      | 192                 |
| Â      | 226                 |
| Ã      | 195                 |
| Ă      | 258                 |
| E      | 69                  |
| Ê      | 202                 |
| I      | 73                  |
| O      | 79                  |
| Ô      | 212                 |
| Õ      | 213                 |
| Ở      | 472                 |
| U      | 85                  |
| Ứ      | 372                 |
| Y      | 89                  |

### What Is Collation?
Collation defines a set of rules for comparing strings, affecting operations like ordering, comparison operators (`<`, `>`), and pattern matching (`LIKE`, `ILIKE`, regex). It also influences capitalization functions like `UPPER()`, `LOWER()`, and `INITCAP()`.

Check the default collation in PostgreSQL:
```sql
postgres=# SHOW lc_collate;
 lc_collate
------------
 en_US.utf8
(1 row)

postgres=# SHOW lc_ctype;
  lc_ctype  
------------
 en_US.utf8
(1 row)
```
In this case, the default collation is `en_US.UTF-8`. Without explicitly specifying a collation, PostgreSQL uses the default.

## Why Don’t Pattern Matching Operators Use Indexes?
Using locales other than `C` or `POSIX` impacts performance. Pattern matching operators like `LIKE` do not utilize ordinary indexes under non-C collations.

> The drawback of using locales other than C or POSIX in PostgreSQL is its performance impact. It slows character handling and prevents ordinary indexes from being used by LIKE. For this reason use locales only if you actually need them.
> As a workaround to allow PostgreSQL to use indexes with LIKE clauses under a non-C locale, several custom operator classes exist. These allow the creation of an index that performs a strict character-by-character comparison, ignoring locale comparison rules. Refer to Section 11.10 for more information. Another approach is to create indexes using the C collation, as discussed in Section 23.2.
> - [PostgreSQL documentation](https://www.postgresql.org/docs/current/locale.html#LOCALE-BEHAVIOR)

## Making Pattern Matching Operators Use Indexes
There are three options:
1. Collation `C`
2. `text_pattern_ops`
3. Trigram (`pg_trgm`)

Let's prepare some data for incoming demonstrations:
```sql
CREATE TABLE grades (id serial PRIMARY KEY,
                                       g int, name text);

CREATE INDEX g_index ON grades(g);

INSERT INTO grades (g, name)
SELECT random()*100,
       substring(md5(random()::text), 0, floor(random()*31)::int)
FROM generate_series(0, 1000000);

VACUUM (ANALYZE,
        VERBOSE,
        FULL);
```

If you check the collation of the table's columns and see it's empty, it means the collation of the column is the default one:
```sql
postgres=# \d+ grades
                                                       Table "public.grades"
 Column |  Type   | Collation | Nullable |              Default               | Storage  | Compression | Stats target | Description 
--------+---------+-----------+----------+------------------------------------+----------+-------------+--------------+-------------
 id     | integer |           | not null | nextval('grades_id_seq'::regclass) | plain    |             |              | 
 g      | integer |           |          |                                    | plain    |             |              | 
 name   | text    |           |          |                                    | extended |             |              | 
Indexes:
    "grades_pkey" PRIMARY KEY, btree (id)
Access method: heap
```

### Collation `C`
Collation `C` ignores locale rules and performs byte-wise comparisons using `memcmp`. It is faster but lacks localization.

The [code snippet](https://doxygen.postgresql.org/varlena_8c.html#a418147ce1a2a44279466b4a24ddeb964) below shows how they handle collation in Postgres:
```c
int varstr_cmp	(	const char * 	arg1,
int 	len1,
const char * 	arg2,
int 	len2,
Oid 	collid 
)	{
    int         result;
    pg_locale_t mylocale;
 
    check_collation_set(collid);
 
    mylocale = pg_newlocale_from_collation(collid);
 
    if (mylocale->collate_is_c)
    {
        result = memcmp(arg1, arg2, Min(len1, len2));
        if ((result == 0) && (len1 != len2))
            result = (len1 < len2) ? -1 : 1;
    }
    else
    {
        /*
         * memcmp() can't tell us which of two unequal strings sorts first,
         * but it's a cheap way to tell if they're equal.  Testing shows that
         * memcmp() followed by strcoll() is only trivially slower than
         * strcoll() by itself, so we don't lose much if this doesn't work out
         * very often, and if it does - for example, because there are many
         * equal strings in the input - then we win big by avoiding expensive
         * collation-aware comparisons.
         */
        if (len1 == len2 && memcmp(arg1, arg2, len1) == 0)
            return 0;
 
        result = pg_strncoll(arg1, len1, arg2, len2, mylocale);
 
        /* Break tie if necessary. */
        if (result == 0 && mylocale->deterministic)
        {
            result = memcmp(arg1, arg2, Min(len1, len2));
            if ((result == 0) && (len1 != len2))
                result = (len1 < len2) ? -1 : 1;
        }
    }
 
    return result;
}
```

Since collate C compares the byte value of strings, its comparing result differs from other collations.

In the collation `en_US.UTF-8`, `á` is less than `z`:
```sql
postgres=# select 'á' > 'z';
 ?column? 
----------
 f
(1 row)
```

However, in the collate C, `á` is greater than `z`, since byte value of `á` is greater than byte value of `z`.
```sql
postgres=# select 'á' > 'z' collate "C";;
 ?column? 
----------
 t
(1 row)
```

When using character capitalization functions like `LOWER` or `UPPER`, the C collation only supports characters in the a-z range. Characters outside this range remain unaffected:

```sql
postgres=# select lower('É' collate "C");
 lower 
-------
 É
(1 row)

postgres=# select lower('É' collate "en_US");
 lower 
-------
 é
(1 row)
```

Additionally, the C collation does not validate Unicode code points. It simply performs byte-by-byte comparisons, which can lead to surprising results:
```sql
postgres=# select ('A' < E'\u0378' collate "C");
 ?column? 
----------
 t
(1 row)

postgres=# select ('A' < E'\u0378' collate "en_US");
 ?column? 
----------
 f
(1 row)
```
Here, Unicode point `U+0378` does not exist, but because the C collation performs a raw byte comparison, it treats it as valid and determines the result accordingly.

References:
[1](https://dba.stackexchange.com/a/241018)

#### Example
Create an index with collation `C`:
```sql
DROP INDEX IF EXISTS name_index1;
CREATE INDEX name_index1 ON grades (name COLLATE "C");
```

Query with collation `C`:
```sql
postgres=# explain analyze select * from grades where name like 'aba%' collate "C";
                                                       QUERY PLAN                                                       
------------------------------------------------------------------------------------------------------------------------
 Index Scan using name_index1 on grades  (cost=0.42..8.45 rows=90 width=23) (actual time=0.199..2.673 rows=229 loops=1)
   Index Cond: ((name >= 'aba'::text) AND (name < 'abb'::text))
   Filter: (name ~~ 'aba%'::text COLLATE "C")
 Planning Time: 0.194 ms
 Execution Time: 2.721 ms
(5 rows)
```

It is important to add `collate "C"` into the query when you are using collate C indexes, otherwise, the operator like equal (`=`) will not utilize the index.

### Text Pattern Ops
```sql
DROP INDEX IF EXISTS name_index2;
CREATE INDEX name_index2 ON grades (name text_pattern_ops);
```

Query:
```sql
postgres=# explain analyze select * from grades where name like 'abc%';
                                                       QUERY PLAN                                                       
------------------------------------------------------------------------------------------------------------------------
 Index Scan using name_index2 on grades  (cost=0.42..8.45 rows=90 width=23) (actual time=0.066..0.317 rows=213 loops=1)
   Index Cond: ((name ~>=~ 'abc'::text) AND (name ~<~ 'abd'::text))
   Filter: (name ~~ 'abc%'::text)
 Planning Time: 0.503 ms
 Execution Time: 0.343 ms
(5 rows)
```

### Note on Collate C and Text Pattern Ops
> Note that you should also create an index with the default operator class if you want queries involving ordinary <, <=, >, or >= comparisons to use an index. Such queries cannot use the xxx_pattern_ops operator classes. (Ordinary equality comparisons can use these operator classes, however.) It is possible to create multiple indexes on the same column with different operator classes. If you do use the C locale, you do not need the xxx_pattern_ops operator classes, because an index with the default operator class is usable for pattern-matching queries in the C locale.
> - [Indexes Opclass](https://www.postgresql.org/docs/current/indexes-opclass.html)

That mean you should also create a normal index on the same column with the text_pattern_ops index, like this:
```sql
CREATE INDEX name_index21 ON grades (name);
```
The ordinary <, <=, >, or >= comparisons will (probably) utilize the normal index.

### Trigram
Trigram indexing is suitable for non-left-anchored patterns like `%abc%` but adds overhead. Use it for advanced pattern matching:
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX name_trgm_idx ON grades USING gin (name gin_trgm_ops);
```

Query:
```sql
postgres=# explain analyze select * from grades where name like 'abd%';
                                                        QUERY PLAN                                                        
--------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=52.70..382.61 rows=90 width=23) (actual time=0.986..7.877 rows=226 loops=1)
   Recheck Cond: (name ~~ 'abd%'::text)
   Rows Removed by Index Recheck: 5
   Heap Blocks: exact=229
   ->  Bitmap Index Scan on name_trgm_idx  (cost=0.00..52.67 rows=90 width=0) (actual time=0.799..0.800 rows=231 loops=1)
         Index Cond: (name ~~ 'abd%'::text)
 Planning Time: 0.205 ms
 Execution Time: 7.940 ms
(8 rows)
```

# References
- [PostgreSQL Documentation - Locale Behavior](https://www.postgresql.org/docs/current/locale.html)
- [PostgreSQL Documentation - Collation](https://www.postgresql.org/docs/current/collation.html)
- [PostgreSQL Documentation - Indexes and Operator Classes](https://www.postgresql.org/docs/current/indexes-opclass.html)
- https://stackoverflow.com/questions/68384666/b-tree-index-does-not-seem-to-be-used/68385039#68385039
- https://dba.stackexchange.com/questions/53811/why-would-you-index-text-pattern-ops-on-a-text-column

