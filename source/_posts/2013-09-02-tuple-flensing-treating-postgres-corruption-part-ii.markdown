---
layout: post
title: "Tuple Flensing II: Treating Postgres Corruption"
date: 2013-09-02 10:21
comments: true
categories: [postgres, corruption]
---
## Part Two: Ingress

This is the second part of the saga of treating a nasty case of
Postgres corruption. This post discusses reloading a set of data that was
wrenched from the jaws of oblivion with careful manual intervention
(read all about *that* in [Part I](/blog/2013/08/27/tuple-flensing-treating-postgres-corruption-part-i/)).

## Integrity constraints for fun and profit

If your recently-generated dump from a corrupt database does not
restore correctly, the first thing you should do is thank your lucky
stars. Every constraint check that fails when restoring is one you
don't have to deal with later, in the application, where it's much
harder to track down, and probably much harder to reason about.

Unfortunately, Postgres does not process database integrity
constraints when generating a dump (to do so would be handy but would
involve significant overhead). They're only checked when
restoring. But in an immediate dump/restore situation like corruption
recovery, this is not a big deal.

Postgres [integrity constraints](http://www.postgresql.org/docs/current/static/ddl-constraints.html)
probably can't represent all the data semantics in your application,
but features like `NOT NULL`, foreign keys, [enumerated types](http://www.postgresql.org/docs/current/static/datatype-enum.html),
unique constraints, and general `CHECK` constraints can help you avoid
shooting yourself in the foot by forbidding some nonsensical data from
entering your system. Sometimes, integrity constraints can be a pain to
set up and update, especially for a fast-moving schema, but consider
looking back at parts of your system where the dust has settled and
adding these as sanity checks there.

The first thing that hit me in this case was the perenial favorite, `NULL`.

## What do we want? NULL semantics! When do we want 'em? False!

`NULL` semantics in SQL are a perennial source of bugs (Jeff Davis has
[a great post](http://thoughts.davisjeff.com/2008/08/13/none-nil-nothing-undef-na-and-sql-null/)
about their gotchas and inconsistencies). What bit me here was more pedestrian:

```console
$ sudo -u postgres pg_restore --dbname my_db < /tmp/corruption-recovery.dump
pg_restore: [archiver (db)] Error while PROCESSING TOC:
pg_restore: [archiver (db)] Error from TOC entry 3075; 0 16810 TABLE DATA my_table my_user
pg_restore: [archiver (db)] COPY failed for table "my_table": ERROR:  null value in column "id" violates not-null constraint
DETAIL:  Failing row contains (null, null, null, null, null, null, null, null, null, null, null, null, null).
CONTEXT:  COPY my_table, line 14339: "\N	 \N    \N    \N	   \N	 \N    \N    \N	   \N	 \N    \N	\N	\N"
```

Yeah, good call, Postgres: that doesn't look quite right. It's
probably safe to say that anything with this much `NULL` is
nonsense. Unfortunately, we can't delete those rows directly in the
existing dump file, and going back to face the corrupt database again
for a fresh dump does not sound like fun. So let's gin up a surrogate
table, drop constraints on that, and temporarily swap it in for the
original table:

```sql
CREATE TABLE foo_surrogate(LIKE foo);
ALTER TABLE foo_surrogate ALTER COLUMN id DROP NOT NULL;
ALTER TABLE foo_surrogate ALTER COLUMN created_at DROP NOT NULL;
ALTER TABLE foo_surrogate ALTER COLUMN updated_at DROP NOT NULL;

ALTER TABLE foo RENAME TO foo_orig;
ALTER TABLE foo_surrogate RENAME TO foo;
```
(Note the handy [CREATE TABLE ... LIKE](http://www.postgresql.org/docs/current/static/sql-createtable.html) syntax.) Then try to restore just this one table again:

```console
$ sudo -u postgres pg_restore --dbname my_db --table foo --data-only < /mnt/tmp/resource3381199-archive2.dump
$
```

Success! Now let's purge the bad data:

```console
=> BEGIN; DELETE FROM foo WHERE id IS NULL OR updated_at IS NULL OR created_at IS NULL;
BEGIN
DELETE 5
```

Now make sure that a reasonable number of rows were affected (5 in
this case), and...

```console
=> COMMIT;
COMMIT
```

I always do anything I might regret in explicit transactions when
working with production data; it's a good habit to get into.

Now we can reinstate the constraints:

```sql
INSERT INTO foo_orig SELECT * FROM foo;
ALTER TABLE foo RENAME TO foo_surrogate;
ALTER TABLE foo_orig RENAME TO foo;
```

I chose to do it by just copying all the data to the original table
since it was fairly small. You could probably also just add the
original constraints to the surrogate table instead, but this gets
dicier if you have foreign key or other constraints to maintain.

## Like a snowflake

A primary key guarantees uniqueness of a piece of data. It ensures
that there is one canonical piece of data describing a user, comment,
or other entity in your system.

In Postgres, this is enforced by an index, and when restoring data,
that index is only built after the data has been loaded (this is more
efficient, as the index does not have to be updated individually for
each row).

In this case, the index would not rebuild because there were several
entries for certain primary key values (this is the next error from
the original restore; it'll keep going and you'll get these all at
once):

```console
pg_restore: [archiver (db)] Error from TOC entry 2986; 2606 16712 CONSTRAINT baz_pkey my_user
pg_restore: [archiver (db)] could not execute query: ERROR:  could not create unique index "baz_pkey"
DETAIL:  Key (id)=(48343) is duplicated.
    Command was: ALTER TABLE ONLY baz
    ADD CONSTRAINT baz_pkey PRIMARY KEY (id);
```

Fortunately, this table had an `updated_at` column, so I simply copied
all duplicates to a surrogate table, assuming that newer was legit
(not always true, in the case of rolled-back transactions, but I had
nothing better to go on):

```sql
CREATE TABLE baz_dupes AS
  SELECT
    DISTINCT ON(id) *
  FROM
    baz
  WHERE
    id IN (SELECT id FROM baz GROUP BY id HAVING count(*) > 1)
  ORDER BY
    id, updated_at;
```

And then I deleted them from the main table (note that `ctid`s make an
appearance here again):

```console
=> BEGIN; DELETE FROM baz WHERE ctid IN (
  SELECT
    DISTINCT ON(id) ctid
  FROM
    baz
  WHERE
    id IN (SELECT id FROM baz GROUP BY id HAVING count(*) > 1)
  ORDER BY
    id, updated_at);
BEGIN
DELETE 8 
```

Then sanity check again, and...

```console
=> COMMIT;
COMMIT
```

Now we can restore the original index:

```console
=> ALTER TABLE ONLY baz
    ADD CONSTRAINT baz_pkey PRIMARY KEY (id);
ALTER TABLE
```

I had to resolve the same primary key issue on another table, but
after resolving that, the restore was finally completed.

### Victory (for real)

At this point, the original data was back in a sane Postgres system,
running smoothly. Chances are, there was still a lot of work to do in
resolving data model issues at the application level (basically,
ensuring any semantic constraints at the application level that were
not represented with database constraints). Luckily, I was free of
that, since I was not the application owner. I handed off the restored
database and explained the duplicate row issues and the potential
outstanding issues with invalid data. I did not get any further
requests or complaints, so I hope the remaining data was in reasonably
good shape.

Overall, this whole process, while rather stressful and somewhat
terrifying, was an incredible learning experience. Thanks to
knowledgeable colleagues, relevant posts archived in the Postgres
mailing lists, some other blogs posts about corruption, and Postgres'
impressive robustness, I was able to save most of the affected
database. I hope the notes in these two posts come in useful should
others be in the unfortunate position of dealing with corruption
issues like these.
