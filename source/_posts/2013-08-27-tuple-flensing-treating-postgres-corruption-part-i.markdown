---
layout: post
title: "Tuple Flensing I: Treating Postgres Corruption"
date: 2013-08-27 01:25
comments: true
categories: [postgres, corruption]
---
## Part One: Egress

While Postgres is an incredibly robust and defensively-written system,
occasionally one does run into problems with data corruption. This can
happen due to hardware issues, filesystem or other OS bugs, and
sometimes, yes, even Postgres bugs. However, even when data corruption
does occur, Postgres will generally contain the damage to specific
chunks of data, and manual intervention will let you recover anything
that was not affected.

This is the story of lessons learned from treating an extensive case
of such corruption in the course of my job with the [Heroku Postgres](https://addons.heroku.com/heroku-postgresql)
Department of Data. This post describes recognizing the problems and 
getting the data out of a corrupt system; the next details restoring it.

<small>All data and metadata is mocked out; no customer data was used in the making of this post.</small>

<!-- more -->

## Preface

Josh Williams has [a fantastic post](http://blog.endpoint.com/2010/06/tracking-down-database-corruption-with.html)
about treating corruption; Josh's post and tips from my colleagues
(and Postgres contributors) [Daniel Farina](https://twitter.com/danfarina)
and [Peter Geogheghan](https://twitter.com/sternocera) were instrumental
in helping me wrap my head around all the nuts and bolts relevant here.

Now before we dig in, note that the easiest recovery is one you don't need
to perform at all. So how can you get a pristine copy of your database? Try your
[standby](http://www.postgresql.org/docs/current/static/high-availability.html). There's
a chance that the corruption only affects the primary, because it
never made it into the WAL stream and was not present when the
replica's base backup was taken. Don't have a standby, but have a
[base backup](http://www.postgresql.org/docs/current/static/continuous-archiving.html#BACKUP-BASE-BACKUP)
and the required WAL segments? Try making a fresh replica (this can
take some time when the base backup is old, but it's better than the
alternative). You can try a fresh replica in parallel with other
recovery efforts. Validate these with `pg_dump`, as discussed below,
and then run a dump and restore (again, see below) to ensure that the
corruption is quarantined. [Seriously](http://tvtropes.org/pmwiki/pmwiki.php/Main/ZombieInfectee),
don't delude yourself into thinking it's going to be okay.

This is by far the easiest and safest mechanism for treating
corruption. It's not a panacea, but we've found it to be unreasonably
effective.

In my case, the existing standby and fresh standbys all failed and
extensive manual recovery was required, but at least it makes for an
interesting story.

If you do need to treat corruption manually, there are a few things
you should keep in mind:

 * Avoid writes: Postgres is in a precarious state; don't tempt fate
   by asking it to do more than necessary. In particular, this is a good
   point to turn off autovacuum.
 * Stop Postgres and make a filesystem copy of the full data
   directory, ideally to another disk if you suspect the hardware. The
   recovery process *will* involve clobbering data; you want to be
   able to start over or cross-reference with the original state of
   the system if you run into problems.
 * Your final step in a corruption recovery should almost always be a
   dump and restore via pg_dump. This is the only way to ensure that
   you're actually running on a sane system. This can take many hours
   for a large database, but if you skip this you might be back at
   square one before too long.
 * The [zero_damaged_pages](http://www.postgresql.org/docs/current/static/runtime-config-developer.html)
   setting may be an easier way to recover. It doesn't always work, but
   it basically does what is outlined here for you.
 * This ain't Kansas anymore: this is effectively [Byzantine failure](http://en.wikipedia.org/wiki/Byzantine_fault_tolerance)
   territory. The advice here might work or everything might go
   horribly wrong. There are no fool-proof step-by-step
   instructions. You're on your own. Furthermore, what's outlined here is a
   general-purpose, fairly coarse approach to recovery. If you need to
   take a more precise approach or just need someone to blame,
   consider hiring a [Postgres consultancy](http://www.postgresql.org/support/professional_support/).

## A bad sign

In this particular case, the problem was almost certainly caused by a
filesystem or hardware issue. Postgres keeps track of the state of
your database via three main sets of data: the heap, where all the
physical data resides; the c(ommit)log, which tracks which
transactions have been committed and which haven't; and the
write-ahead log (WAL), which does all the bookkeeping to ensure
crash-safety and is a key mechanism in Postgres replication.

After intially noticing the corruption, I poked at the affected files
with `hexdump`, and noticed that one of the `clog` files, which normally
look like this:

```console
$ hexdump -C pg_clog/0000 | head
00000000  40 55 55 55 55 55 55 55  55 55 55 55 55 55 55 55  |@UUUUUUUUUUUUUUU|
00000010  55 55 55 55 55 55 55 55  55 55 55 55 55 55 55 55  |UUUUUUUUUUUUUUUU|
*
000000b0  55 55 55 55 55 55 55 55  55 55 65 55 55 59 55 95  |UUUUUUUUUUeUUYU.|
000000c0  55 55 55 55 55 55 59 55  55 55 55 55 55 55 55 55  |UUUUUUYUUUUUUUUU|
000000d0  55 95 55 55 55 55 55 55  55 55 55 55 55 55 55 55  |U.UUUUUUUUUUUUUU|
000000e0  55 65 55 55 95 55 55 95  55 55 55 55 55 55 59 55  |UeUU.UU.UUUUUUYU|
000000f0  55 55 55 55 55 55 55 55  55 55 55 55 55 55 55 55  |UUUUUUUUUUUUUUUU|
*
00014920  55 55 55 55 55 55 55 55  95 55 55 95 55 55 95 55  |UUUUUUUU.UU.UU.U|
```

...looked more like this instead:

```console
$ hexdump -C pg_clog/0002 | head -20
00000000  40 55 55 55 55 55 55 55  55 55 55 55 55 55 55 55  |@UUUUUUUUUUUUUUU|
00000010  55 55 55 55 55 55 55 55  55 55 55 55 55 55 55 55  |UUUUUUUUUUUUUUUU|
*
00000040  02 00 00 00 00 00 00 00  00 00 00 00 00 00 03 00  |................|
00000050  01 00 16 00 07 09 20 ff  7f 0d 00 00 00 00 00 00  |...... .........|
00000060  e7 04 00 00 16 00 00 00  00 00 00 00 10 00 00 00  |................|
00000070  0c 18 66 bf 01 00 02 00  03 00 00 00 62 00 00 00  |..f.........b...|
00000080  98 02 00 00 98 02 00 00  00 00 00 00 da 00 00 00  |................|
00000090  a4 01 00 00 04 01 00 03  01 bc 02 00 00 64 08 00  |.............d..|
000000a0  00 00 01 10 7b 1f 50 3d  00 65 e1 43 3b 05 dd df  |....{.P=.e.C;...|
000000b0  3a 88 c4 e5 a7 0f 04 13  05 dd 5f 0f 04 ff 01 0f  |:........._.....|
000000c0  04 36 33 01 00 00 00 00  00 00 00 bc 02 00 00 01  |.63.............|
000000d0  00 00 00 01 00 00 00 38  ef 4b 3e 00 00 00 00 00  |.......8.K>.....|
000000e0  ee 0d 00 00 c8 06 00 00  04 01 00 03 01 19 00 00  |................|
000000f0  00 64 08 00 00 00 01 10  4c 00 00 00 00 61 67 67  |.d......L....agg|
00000100  72 65 67 61 74 00 65 5f  64 75 6d 6d 79 00 00 48  |regat.e_dummy..H|
00000110  00 00 00 73 65 6c 65 00  63 74 20 24 32 20 2b 20  |...sele.ct $2 + |
00000120  00 24 31 00 00 2c 00 00  00 00 74 65 78 74 6c 65  |.$1..,....textle|
```

Note the ASCII view of the last few lines. If you see anything
human-readable in the `clog` bitmaps, you should either start buying
lottery tickets, or something very bad has happened.

I originally checked the file after seeing an unexpected error message
that led me to [file a Postgres bug](http://www.postgresql.org/message-id/E1V4LCV-0007AE-8p@wrigleys.postgresql.org).
(The Postgres community, by the way, takes data integrity issues very
seriously and addresses them quickly, so don't hesitate to file a bug
if you think you're running into a legitimate Postgres problem.)

## Trouble with transactions

The above happened on a replica, and checking the master revealed a
different--but just as serious--problem in the logs:

```console
PG::Error: ERROR: could not access status of transaction 2616655872
DETAIL: Could not open file "pg_clog/09BF": No such file or directory.
```

The `clog` files are bitmaps that live in the `pg_clog` subdirectory of
your Postgres data directory and track which transactions have been
committed and which have aborted.

You can take a look in the subdirectory and see the various `clog`
files. If you don't know where your data directory is, you can run the
following Postgres command (this must be run as a Postgres supseruser,
e.g. `postgres` in most installations):

```console
maciek# show data_directory;
        data_directory        
------------------------------
 /var/lib/postgresql/9.2/main
(1 row)
```

In that directory, you'll see a number of `clog` files

```console
$ sudo ls -l /var/lib/postgresql/9.2/main/pg_clog
total 1072
-rw------- 1 postgres postgres 262144 Jan  9  2013 0000
-rw------- 1 postgres postgres 262144 Mar 12 11:53 0001
-rw------- 1 postgres postgres 262144 May  8 11:07 0002
...
-rw------- 1 postgres postgres 262144 Aug 12 13:41 0011
-rw------- 1 postgres postgres  32768 Aug 13 16:38 0012
```

These are sequential, and end *way* before the `09BF` file that the
error message complained about, so that's not a great sign. So what
can we do? A reasonable assumption might be that everything
referencing transactions in that suspect `clog` file committed
(actually, that's a terrible assumption, but it's easy to fake
transaction metadata like that, and that's the only way to see the
effects of these alleged transactions, so let's roll with it).  In
order to gin up a hunky-dory `clog` file, where everything committed, we
can use the trusty `dd` tool. If you're not familiar with `dd`, you
can think of it as a bit scalpel for file surgery:

```console
$ sudo -u postgres dd if=/dev/zero of=/var/lib/postgresql/9.2/main/pg_clog/09BF bs=256k count=1
1+0 records out
262144 bytes (262 kB) copied, 0.000437751 s, 599 MB/s
```

Here, `of` is the output `clog` file you need to generate (dd has a
somewhat unconventional way to pass arguments). Everything else (block
size, block count, and input file--the handy `/dev/zero` special file)
can stay the same. Note that you'll probably need to run with sudo as
whatever user owns the other `clog` files (typically `postgres`).

## Checking sanity

So does this fix our problem? One way to check is to re-run the query
that triggered the error in the first place. A more thorough check is
to just run `pg_dump` to `/dev/null`:

```console
$ pg_dump --verbose >/dev/null my_db
```

This works because generally, corruption manifests as either
references to bogus transactions (which can be addressed as above), or
some sort of failure when attempting to read a specific set of data
from disk. Postgres organizes data on disk in pages, normally 8192
bytes each. When things get corrupted, problems tend to be
page-aligned--either a page will be fine, or it's wholy
suspect. Failure to read a page indicates that either the Postgres
page headers are corrupt, or that the data is mangled in such a way as
to cause errors in the per-datatype `recv` deserialization functions.

Occasionally, a page will be corrupt in such a way as to crash
Postgres when one attempts to read it. This is problematic for
recovery, but not a showstopper--if we can figure out what pages are
causing crashes, we can surgically excise them.

In either case, pg_dump attempts to read every single live page of
data, so if you make it through that, you're on your way to a
functioning system. Note, however, that this does *not* validate
the integrity of your indexes: another good reason to always go
through a dump and restore.

Note also that the most pernicious aspects of corruption occur when
none of the above trigger: when bits get shuffled around, but in such
a way as to still represent valid data (just not the data you put
there). Unfortunately, prior to Postgres 9.3's
[checksums](http://michael.otacoo.com/postgresql-2/postgres-9-3-feature-highlight-data-checksums/)
feature, there's not much you can do about this. Fortunately, this is
also the least likely problem, as corruption tends to cause chaos and
very few datatypes are resilient to bit scrambling (numeric types
being a notable exception).

## Pages behaving badly

In this case, the pg_dump still failed with the same error a few more
times. After applying the same remedy a few times to different `clog`
files, I eventually got to a different error:

```console
ERROR:  invalid memory alloc request size 2667865904
STATEMENT:  COPY public.some_table (col1 int, col2 text) TO stdout
```

While Postgres supports [pretty big](http://www.postgresql.org/about/) rows
and fields, its internal memory management is limited to working with chunks
up to [1GB](https://github.com/postgres/postgres/blob/263865a48973767ce8ed7b7788059a38a24a9f37/src/include/utils/memutils.h#L23-L39).
This request for 2.5GB seems shady. It's likely the metadata about the
size of the value stored on this page is corrupted.

Without significantly more digging, we have to give up on this data as
lost and nuke the page. There are a few other errors you may see that
likely point to corruption:

```console
pg_dump: SQL command failed
pg_dump: Error message from server: ERROR: timestamp out of range
pg_dump: The command was: COPY public.table (id, created_at) TO stdout
```

and

```console
pg_dump: SQL command failed
pg_dump: Error message from server: ERROR:  invalid page header in block 29 of relation "foo"
pg_dump: The command was: COPY public.foo (id, bar) TO stdout
```

and

```console
pg_dump: SQL command failed
pg_dump: Error message from server: ERROR: missing chunk number 0 for toast value 118848655 in pg_toast_2619
pg_dump: The command was: COPY public.bar (id, baz, quux) TO stdout
```

So how do we track down the offending page? Fortunately, all rows in
Postgres have a hidden column called `ctid` that is effectively the
"physical" row id. If you have the `ctid`, you can figure out where
the page lives on disk. You can project the `ctid` just like any other
column:

```console
maciek=# select ctid, relname from pg_class;
  ctid  |                  relname                   
--------+--------------------------------------------
 (0,1)  | pg_statistic
 (0,2)  | pg_type
 (0,38) | pg_toast_75018
 (0,39) | pg_toast_2619
 (0,40) | pg_toast_2619_index
 (0,41) | pg_authid_rolname_index
 (0,42) | pg_authid_oid_index
 (0,43) | pg_attribute_relid_attnam_index
 (0,44) | pg_attribute_relid_attnum_index
...
 (7,73) | pg_toast_11620
 (7,77) | pg_toast_11620_index
(320 rows)
```

The `ctid` values you see are a (page, offset) pair. Chances are,
projecting just the `ctid` from an affected table is not going to
trigger the memory allocation error we saw above (because Postgres
doesn't have to read any of the affected data).

With that, you can go row by row, projecting every value to see if it
causes problems. Doing this manually can take forever on a good-sized
table, so I wrote a PL/pgSQL function to do it (the `FETCH_COUNT` trick
from Josh's post could also have helped, but I didn't think of that
at the time):

```sql
CREATE OR REPLACE FUNCTION public.check_ctids(t regclass)
 RETURNS void
 LANGUAGE plpgsql
AS $function$
DECLARE
  tquoted text;
  ctid tid;
  attr name;
BEGIN
SELECT quote_ident(t::text) INTO tquoted;
FOR ctid IN EXECUTE 'SELECT ctid FROM ' || tquoted LOOP
  RAISE NOTICE 'dumping data for ctid %', ctid;
  FOR attr IN EXECUTE 'SELECT quote_ident(attname) FROM pg_attribute WHERE attrelid = $1 and attnum > 0' USING t LOOP
    BEGIN
      PERFORM 'SELECT ' || attr || ' FROM ' || tquoted || ' WHERE ' || tquoted || '.ctid = check_ctids.ctid';
    EXCEPTION
      WHEN internal_error THEN
        RAISE NOTICE 'data for column % at ctid % is corrupt', attr, ctid;
    END;
  END LOOP;
END LOOP;
END;
$function$;
```

This looks at all the `ctid`s in the target table, and for each one,
tries to project the values in every column for that row. The columns
are projected one at a time, so that if one exhibits a problem and we
have to purge the row, we can still check the valid ones and save
that data beforehand.

This works for small and medium-sized tables, but takes forever on
anything sizable. I tried to optimize it first by doing a binary
search for bad tuples across `ctid`s, and then by taking advantage of
the fact that corruption is--as mentioned above--typically
page-oriented. In the end, neither of these panned out (for reasons
I'll get to below), but I learned some interesting PL/pgSQL. In any
case, this approach was serviceable for a while and did help me make
progress. The output looks something like this:

```console

maciek=# select check_ctids('pg_class');
NOTICE:  dumping data for ctid (0,1)
NOTICE:  dumping data for ctid (0,2)
NOTICE:  dumping data for ctid (0,38)
NOTICE:  dumping data for ctid (0,39)
NOTICE:  dumping data for ctid (0,40)
...
NOTICE:  dumping data for ctid (6,4)
NOTICE:  data for column col1 at ctid (6,4) is corrupt
NOTICE:  data for column col2 at ctid (6,4) is corrupt
NOTICE:  dumping data for ctid (6,5)
NOTICE:  dumping data for ctid (6,6)
NOTICE:  data for column col1 at ctid (6,6) is corrupt
NOTICE:  data for column col3 at ctid (6,6) is corrupt
NOTICE:  dumping data for ctid (6,9)
NOTICE:  data for column col1 at ctid (6,9) is corrupt
NOTICE:  data for column col2 at ctid (6,9) is corrupt
NOTICE:  data for column col3 at ctid (6,9) is corrupt
NOTICE:  dumping data for ctid (6,13)
NOTICE:  data for column col1 at ctid (6,13) is corrupt
NOTICE:  data for column col2 at ctid (6,13) is corrupt
NOTICE:  data for column col3 at ctid (6,13) is corrupt
NOTICE:  dumping data for ctid (6,14)
...
 check_ctids
-------------
 
(1 row)

```

Trouble on page 6! At this point, you can try to project the other
columns from the affected rows (and any potentially unaffected rows)
so you can save whatever data is still recoverable. 

Once we find an affected page, we need to figure out where it lives on
disk. Again, we find what we're looking for in the data directory.
The `base` subdirectory contains all the "actual" data (as opposed to
metadata) in your Postgres instance, so this is where we look. The
first level looks like this:

```console
$ sudo ls -l /var/lib/postgresql/9.2/main/base
total 60
drwx------ 2 postgres postgres 12288 Jun 30 15:52 1
drwx------ 2 postgres postgres  4096 Jun 26  2012 12035
drwx------ 2 postgres postgres  4096 Aug  9 15:20 12040
drwx------ 2 postgres postgres 12288 Aug 15 16:59 32775
drwx------ 2 postgres postgres 20480 Aug 16 17:36 75568
drwx------ 2 postgres postgres  4096 Aug 18 02:25 76255
drwx------ 2 postgres postgres  4096 Jun 20 21:51 pgsql_tmp
```

This contains all the different databases (as in `CREATE
DATABASE ...`) in your Postgres instance. This query can help you
figure out the right subdirectory:

```console
maciek=# select oid, datname from pg_database;
  oid  |     datname      
-------+------------------
     1 | template1
 12035 | template0
 12040 | postgres
 32775 | maciek
 75568 | observatory_test
 76255 | pqgotest
(6 rows)
```

(`oid` is another hidden column, a surrogate logical identifier for
many system tables.) In there, you'll see files for your tables and
other aspects of your database:

```console
$ sudo ls -l /var/lib/postgresql/9.2/main/base/32775
total 6796
-rw------- 1 postgres postgres 163840 Aug 18 01:50 11777
-rw------- 1 postgres postgres  24576 Aug  7 19:33 11777_fsm
-rw------- 1 postgres postgres   8192 Aug  7 19:33 11777_vm
...
-rw------- 1 postgres postgres   8192 Aug  7 19:31 75207
-rw------- 1 postgres postgres   8192 Aug  7 19:31 75208
-rw------- 1 postgres postgres      0 Aug 15 16:59 80343
-rw------- 1 postgres postgres      0 Aug 15 16:59 80346
-rw------- 1 postgres postgres    512 Jun  3 14:37 pg_filenode.map
-rw------- 1 postgres postgres 111220 Aug  9 15:21 pg_internal.init
-rw------- 1 postgres postgres      4 Jun  3 14:37 PG_VERSION
```

To find the right table, you might assume you can use the oid again (I
certainly did), but this won't always work. The `pg_class` system
catalog table can show you the actual file you need (adjust the namespace
as needed):

```console
maciek=# select relfilenode from pg_class c
  inner join pg_namespace ns on c.relnamespace = ns.oid
  where relname = 'foo' and nspname = 'public';
 relfilenode 
-------------
       80343
(1 row)
```

So we have the subdirectory for the database, the file for the table,
and the page and offset from the `ctid`. Now we need to do some bit
surgery, just as with the `clog` files above. This is where `dd`'s
"addressing" feature (via its `blocksize` and `seek` parameters) comes
in really handy: `dd` makes it easy to treat a file as a sequence
of contiguous chunks, and to overwrite just one of these chunks.

Let's go to town:

```console
$ sudo dd if=/dev/zero bs=8192 count=1 seek=6 \
    of=/var/lib/postgresql/9.2/main/base/32775/80343 conv=notrunc
1+0 records out
262144 bytes (262 kB) copied, 0.000437751 s, 599 MB/s
```

Here, `bs=8192` indicates the Postgres page size, `count=1` means we
overwrite just one page, and `seek=6` means we want to seek to the 6th
8192-byte page. Again, we use `/dev/zero` as a source. The very
important `conv=notruc` means don't truncate the file; just write to
the middle. Remarkably, overwriting a page like this in Postgres is enough
to wipe it out with no ill effects. That's a fantastic property for what
is effectively a massive binary serialization format.

In any case, doing these one-by-one can get tedious (and
error-prone). If you find a lot of these up front, you can even script
the `dd` invocations:

```bash
for bad_page in 89 203 221 227;
do
  sudo dd if=/dev/zero of=/var/lib/postgresql/9.2/main/base/16385/17889 \
    bs=8192 seek=${bad_page} count=1 conv=notrunc
done
```

Be extremely careful with this, though--it's easy to fat-finger your
way to wiping out large swathes of your database.

## Trouble in purgatory

After a while of making progress like this, I ran into a problem that
I couldn't solve with PL/pgSQL:

```console
LOG:  server process (PID 23544) was terminated by signal 11
LOG:  terminating any other active server processes
FATAL:  the database system is in recovery mode
LOG:  all server processes terminated; reinitializing
```

Postgres is written defensively, but if its data is scrambled badly
enough, it will still occasionally wig out. With luck, it will recover in
a few seconds and you can continue treating the corruption, but
subsequent attempts to read the affected pages are likely to cause
repeat crashes. This means that any PL/pgSQL functions that rely on
being able to scan a whole table can't really cope with server crashes.

Bash (and shell scripting in general) gets a bad rap--in many cases,
deservedly so--but it's still a fantastic way to glue together other
programs. It seemed like an appropriate tool here:

```bash
#!/bin/bash

# Fill in with everything your own psql invocation needs
psql="psql -U <user> -d <database> -p <port> -h <host>"

relation=${1-"Usage: $0 relation"}

last_page=-1
for ctid in $(<<< "select ctid from $relation" $psql -qAt)
do
    page=$(echo $ctid | sed -r 's/\(([0-9]+),[0-9]+\)/\1/')
    if [ "$page" -eq "$last_page" ]
    then
        continue
    fi
    echo "dumping page $page"
    if ! <<< "select * from $relation where ctid >= '($page,1)' and ctid < '($((page+1)),1)'" $psql > /dev/null
    then
        echo "Failed on page $page"
        until <<< "select 1" $psql >/dev/null; do :; done
    fi
    last_page="$page"
done
```

This selects all `ctid`s from a table and then iterates over them.
For each, it dumps all data on a page, and then skips `ctid`s until
the next page (yes, it would probably be faster to query for only
distinct pages and iterate over just that; this was fast enough).
This prints errors on failed pages (I no longer cared about individual
columns anymore; these few pages were a mess and I couldn't get
anything out), but the really nice part is if that Postgres crashes,
it also just prints an error and waits for it to come back up. This
lets you scan a full table even if Postgres crashes on every page,
and you can apply the page-zapping technique above.

## Victory (sort of)

After a few more rounds of dealing with a potpourri of the issues above,
I was eventually able to clear out all the errors and get pg_dump to
complete successfully. Only a few dozen pages were affected, but
sorting through everything took hours and was very mentally demanding:
this is where any abstraction provided by your database breaks down,
and you need to start thinking of your data in a completely different
way.

I've also elided some of the more frustrating inconsistencies and
gotchas in dealing with a system in this state. Occasionally corrupt
pages that I had cleared would come back (autovacuum and other
connections were off, so I'm still not sure what caused these),
and in one case a manual `COPY` worked even though `pg_dump`'s invocation
of the same command failed. Be prepared for weirdness.

Overall, though, Postgres held up admirably: it's very impressive how
well it does in the face of serious data corruption.

Unfortunately, after all this, the dump I got would still not restore
cleanly. I'll go over the (much simpler) steps I had to take to get
things back into Postgres in the next post.
