PgDBF is a program for converting XBase databases - particularly FoxPro tables with memo files - into a format that PostgreSQL can directly import.  It's a compact C project with no dependencies other than standard Unix libraries.  While the project is relatively tiny and simple, it's also heavily optimized via profiling - routine benchmark were many times faster than with other Open Source programs.  In fact, even on slower systems, conversions are typically limited by hard drive speed.

# Features
PgDBF was designed with a few core principles:

* Simplicity.  This code should be understandable by anyone who wants to hack it.
* Robustness.  Every syscall is checked for success.
* Speed.  PgDBF was born to be the fastest conversion available anywhere.
* Completeness.  It has full support for FoxPro memo files.
* Portability.  PgDBF runs on 32- and 64-bit systems, and both little-endian (eg x86) and big-endian (eg PowerPC) architectures.

# Performance

PgDBF's speed is generally limited by how fast it can read your hard drives.  A striped RAID of quick disks can keep PgDBF pretty well fed on a single-processor system.  One problem area is with memo files, which may become very internally fragmented as memo fields are created, deleted, and updated.  For best results, consider placing the DBF and FPT files on a RAM drive so that there's no seek penalty as there is with spinning hard drives, or using a filesystem such as ZFS that caches aggressively.

One particularly fragmented 160MB table with memo fields used to take over three minutes on a FreeBSD UFS2 filesystem.  Moving the files to a RAM disk dropped the conversion time to around 1.2 seconds.

A certain test table used during development comprises a 280MB DBF file and a 660MB memo file. PgDBF converts this to a 1.3 million row PostgreSQL table in about 11 seconds, or at a rate of almost 120,000 rows per second.

# Downloading

Downloadable tarballs of numbered releases are available at [the SourceForge download page](http://sourceforge.net/project/showfiles.php?group_id=257285). Development happens on GitHub at [https://github.com/kstrauser/pgdbf](https://github.com/kstrauser/pgdbf).

# Release Notes

Version 0.6.3 (January 21, 2018) adds the autoindex (autoincrement) field identifier 0x31 and 0x32, and outputs "V" fields as varchar. Handles wide varchars. Can ignore fields with "-i". Can disable automatic removal of field padding. Correct column time for currencies. Format FLOAT fields like NUMERIC.

Version 0.6.2 (September 30, 2012) adds built-in iconv translation from the given encoding to UTF-8. Thanks to Philipp Wollermann, PgDBF's new co-author/co-maintainer, for the heavy lifting!

Version 0.6.1 (March 1, 2012) is exactly identical in operation to 0.6.0, but I'm an awful release engineer and always forget to update the version number in the man page, etc. This bumps the revision to indicate that files have changed.

Version 0.6.0 (February 29, 2012) breaks compatibility with older versions by creating NUMERIC fields as type NUMERIC instead of TEXT by default. Use the "-N" flag to get the old behavior. Use the new(ish) PRId64 definition from inttypes.h to print 64-bit CURRENCY fields. Adds an optional progress bar. Adds 64-bit file offset support for Linux (and possibly Solaris, although it's not tested). Increased the default buffer and batch sizes from 1MB and 128KB to 4MB and 16MB respectively. Cleaned up lots of endian-handling code.

Version 0.5.5 (February 3, 2011) adds the "-q" option to place quotation marks around the name of the table when using it in statements like "CREATE TABLE", "DROP TABLE", etc.

Version 0.5.4 (November 16, 2010) does away with the single memofile size check at the beginning of the run and checks for valid memo record offsets every time one is requested.

Version 0.5.3 (November 15, 2010) fixes an unsigned int comparison bug.

Version 0.5.2 (November 3, 2010) adds tests for the validity of specified memofiles. Older versions would possibly segfault if an invalid memofile was given.

Version 0.5.1 (March 9, 2010) enables the "IF EXISTS" (the "-e" option) to "DROP TABLE" by default.  If you are using PgDBF with a version of PostgreSQL older than 8.2 (which was released in December 2006), you will need to use the "-E" flag to disable it.  Also adds "-u" and "-U" flags to control whether a "TRUNCATE TABLE" statement is issued to clear the table before copying data.

Version 0.5.0 (November 24, 2009) is not a drop-in replacement for older versions.  Specifically:


    * It no longer tries to guess the names of memofiles.  If a table has a memofile, it must be specified with the "-m" command line argument.
    * The "N" (numeric) and "F" (float) datatypes now emit \N (NULL) instead of empty strings.
  
# Building and Installation

```shell
./configure; make; make install
```

# Usage

Use PgDBF to convert your XBase tables into a format suitable for piping into the psql client.  PgDBF will generate the commands necessary to create near-exact replicas of your tables. Its output looks something like: 

```sql
BEGIN;
SET statement_timeout=60000; DROP TABLE IF EXISTS invoice; SET statement_timeout=0;
CREATE TABLE invoice (invoiceid INTEGER, note VARCHAR(50), billdate DATE, submitted TIMESTAMP, memofield TEXT, approved BOOLEAN);
\COPY invoice FROM STDIN
...
...
...
\.
COMMIT;
CREATE INDEX invoice_invoiceid ON invoice(invoiceid);
```

Note that the entire process is usually wrapped inside a transaction so that other clients will have access to the old data until the transaction is completed. 


## Command Line

```
Usage: pgdbf [-cCdDeEhnNpPqQtTuU] [-s encoding] [-m memofilename] [-i fieldname1,fieldname2,fieldnameN] [-o tablename] filename [indexcolumn ...]
```

The only required argument is the filename of the table to be converted.  If the table has a memo field, then use the "-m" option to specify the path to the memo file.

The "-h" flag prints this usage information and then exits.

The "-c" flag cause PgDBF to print a "CREATE TABLE" statement to make a table with similar datatypes and column names as the DBF file.  This is the default behavior.

The "-C" suppresses the "CREATE TABLE" statement.

The "-d" flag causes PgDBF to print a "DROP TABLE" statement before the "CREATE TABLE" command.  This is useful for replacing the contents of a table that already exists in PostgreSQL.  This is the default behavior.  If "-C" was used to disable the "CREATE TABLE" statement, then "-d" will be silently ignored (as it makes no sense to drop a table and then try to insert into it).

The "-D" flag causes PgDBF *not* to print the "DROP TABLE" statement.

The "-e" flag changes the "DROP TABLE" statement to "DROP TABLE IF EXISTS".  PostgreSQL will return an error when attempting to drop a table that does not exist.  Version 8.2 and newer can use the "IF EXISTS" modifier to drop the table only if it's already defined, and otherwise continue without error. This is the default.

The "-E" flag disables the "IF EXISTS" modifier to "DROP TABLE" for compatibility with old versions of PostgreSQL.

The "-i" flag gives a comma-separated list of field names to remove from the output.

Use the "-m" argument to specify the memofile (if any) associated with the table.

The "‐n" flag creates NUMERIC fields with type NUMERIC. **This is a new default and different from old versions of PgDBF!**

The "‐N" flag creates NUMERIC fields with type TEXT. This is the historical default setting. Use this if rows contain invalid number data in NUMERIC fields, which  are  essentially CHARACTER fields behind the scenes, and you receive errors when importing the data into PostgreSQL.

The "-o" flag uses the user-defined table name.
 
The "‐p" argument shows a progress bar during the conversion process.

The default "-P" argument hides the progress bar.

The "‐q" flag encloses the name of the table in quotation marks in statements like "CREATE TABLE", "DROP TABLE", and so on. This is useful  in cases  where  the  table name is a PostgreSQL reserved word, and almost certainly harmless in all other cases.

The "‐Q" flag doesn't enclose  the  name  of  the  table  in  quotation  marks. This is the default.

The "-s" flag uses iconv to convert from the named encoding to UTF-8. This is useful for importing databases originally encoded in non-ASCII charsets without losing the non-ASCII data.

The "-t" flag wraps the entire script in a transaction.  Since transaction commits are atomic, there will never be an instant in time where the table appears empty to other clients.  Instead, the old table data will seem to be instantaneously replaced with the new contents.  This is the default.

The "-T" flag removes the wrapper transaction.  This is generally not a good idea as it causes the table to be completely empty at times.

The "-u" flag generates a "TRUNCATE TABLE" statement to quickly empty a table without dropping and re-creating it.

"-U" disables the "TRUNCATE TABLE" statement.  This is the default.

Indices are automatically created if you specify the columns (or expressions!) you want indexed on the command line. For example, 

```shell
pgdbf foo.dbf rowid "substr(textfield,1,4)" price
```

will create three indices on the new foo table: one each for the rowid and price columns, and one for the substr() expression. It tries to give each index a reasonable name.

# Note about character encodings

Older versions of PostgreSQL defaulted to the SQL_ASCII character encoding which accepted pretty much any string of bytes you could throw at it. Newer versions of PostgreSQL default to using UTF-8 encoding. PgDBF outputs data exactly as it's stored in the XBase tables, and this can result in dumps that PostgreSQL can't directly import.

PgDBF's "-s" flag can handle this for you by using `libiconv` to convert an XBase table's output to UTF-8. If you're not sure which encoding your table uses, consider using the `chardet` or `uchardet` program to examine PgDBF's output and suggest the most likely encoding.

Sometimes this is insufficient, usually because the input table files contain corrupted data. In this case, if you can afford to lose the corrupted bytes, the separate `iconv` command can be used like so:

```shell
pgdbf table.dbf | iconv -c -f UTF-8 -t UTF-8 | psql targetdatabase
```

Note that this will *discard* data that can't be stored in UTF-8.

# Bugs

When multiple incompatible interpretations of a type are available, such as the 'B' type which can mean 'binary object' in dBASE V or 'double-precision float' in FoxPro, I've used the FoxPro version.

Not all XBase datatypes are supported right now.  As of this writing, PgDBF can handle boolean, currency, date, double-precision float, float, general (although only outputs empty strings; it's unclear how to resolve OLE objects at this time), integer, memo, numeric, timestamp, and varchar fields.  If you need other datatypes, send a small sample database for testing.

# Contributors

PgDBF was originally written by [Kirk Strauser](kirk@strauser.com">kirk@strauser.com) and approved for release under the GPLv3 by the owner of his company, Brandon Day. Philipp Wollermann joined the team leading up to the release of version 0.6.2.

Special thanks to the following for suggestions and bug reports:

* John Easton
* Stephan Kasdorf
* Alan Polinsky
* Fabrízio de Royes Mello
* Gordon Sweet

# Support

The mailing list home page is at https://groups.google.com/d/forum/pgdbf
