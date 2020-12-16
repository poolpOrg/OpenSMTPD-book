# Understanding tables

    When I look back on my childhood,
    my fondest memories are those surrounding the dinner table.''
    -- Katie Lee


## What is a table ?
Tables are the mechanism through which OpenSMTPD performs all[<sup>1</sup>](#1) of its lookups at runtime.
A _lookup_ process isolates them,
requesting the table API on behalf of pretty much every part of the daemon,
and forwarding back the answers.
The concept is so central to the software that it really needs to be addressed before we dive into actual configuration:
__all configurations__ rely on tables wether they are used explicitely or not.

A table is a _named_ resource that is capable of answering requests.
How it does so is completely opaque to OpenSMTPD:
the table may be looking into a file,
a database or calling a web service,
it doesn't really matter.
What matters is that the table can lookup something somewhere and return a result out of it.
The table names are referenced at various places in the configuration,
these places determine what _kinds_ of lookups and results the table is expected to serve.
Because different contexts can use the same data some tables may be used in several different places,
while other tables may only be used in specific places.

The lookup kinds will be discussed as they are used in future chapters so I won't detail them here.
The general idea is that if a table is used in a specific place of the configuration,
then it will receive a specific kind of lookup that expects a correctly crafted answer.
If a table is used as parameter to `for src`,
then it will be queried to lookup for IP addresses and the results are expected to parse as valid IP addresses.
This type checking in the table API ensures that tables don't return bogus values,
their results may be wrong due to misconfiguration but they are valid for a context.
The `table(5)` man page describes the various types with examples of use,
it is a good reference if you forget about the format of a valid entry.

Data kind set aside,
tables support two data structures: lists and key-value mappings.
This makes sense because uses-cases for tables in OpenSMTPD pretty much all fall in either one of:
iterating through a list of values,
checking if a value is part of a list,
retrieving a value associated to a key.


## Explicit and Implicit tables
There are explicit tables,
which a postmaster will explicitly declare in the configuration file,
and which provide a configuration to detail how the table is populated.
Whenever the `table` keyword is found in the configuration,
it means an explicit table was declared.
An example of such table is the "aliases" table which is part of the default configuration,
and which declares an explicit table backed by a file holding a mapping of aliases to system accounts:

```
$ grep ^table /etc/mail/smtpd.conf
table "aliases" file:/etc/mail/aliases
$ grep ^root /etc/mail/aliases
root: gilles
```

There are also implicit tables,
which a postmaster will not necessarily know about,
and that are created internally by OpenSMTPD to make some of its own lookups more convenients.
Unlike explicit tables,
they are not visible in the configuration and can't be referenced in rules.
They also use a different naming convention,
one that uses characters forbidden in explicit table names to ensure there is no risk of collisions.
Examples of such table include the "getpwnam" table which is used to lookup local users,
or the "localaddrs" table which is filled at startup with all the IP addresses of the local machine to ease lookup of local interfaces.

Implicit tables will not be mentionned much more in this book as they are merely a commodity to OpenSMTPD developers,
however knowing that they exist can help understand how some features are implemented.

In practice,
a postmaster will only ever see and deal with explicit tables.


## Static and Dynamic tables
When defining tables in the configuration,
a postmaster may rely on using static or dynamic tables,
both having their advantages and disadvantages.

A static table is one that is declared with all of its values in _smtpd.conf_.
It is simple to define,
what you see is what you get,
there is no risk of the values changing at runtime and looking at the configuration is enough to understand what the table does.
Once a static table is defined and tested to work,
you know the results will be replicable again and again.
However,
updating the table is not possible at runtime,
it requires editing the configuration file and restarting the daemon to catch up the change:

```
table "foobar" { foo, bar, baz }
table "barbaz" { foo = bar, baz = qux }
```

A dynamic table is one that is declared with a name and an external resource attached to it.
The _values_ provided by the table are not visible in _smtpd.conf_,
they can be changed at runtime without the daemon having to know or reload.
Because they are not part of the configuration but served by external resources,
these tables can contain arbitrarily large sets of values backed by more or less complex mechanisms (think databases):

```
table "foobar" file:/etc/mail/foobar
table "barbaz" db:/etc/mail/barbaz.db
```

It is not uncommon to mix both static and dynamic tables depending on the context,
where the table values originates from,
how many entries are part of the table and how frequently it will be updated.
None is "better" than the other,
it is just a matter of what seems to be the right choice given the context.

For example,
it makes sense using a dynamic table for aliases which may contain numerous records that change every now and then,
while at the same time using a static table to list a couple IP addresses that won't change:

```
table "aliases" file:/etc/mail/aliases
table "myaddrs" { 192.168.1.2, 192.168.1.3 }
```

Still,
there would be nothing wrong using a file for the list of IP addresses,
if maybe the list was slightly larger,
or meant to be dynamically changed every now and then,
or even if it was fed by an external tool:

```
table "aliases" file:/etc/mail/aliases
table "myaddrs" file:/etc/mail/myaddrs
```

In some situations the choice is obvious,
you really want the table content to be dynamic and not part of the configuration,
and in other situations it is just a matter of convenience or taste.
The only thing to keep in mind is that static tables are loaded in memory at startup,
whereas dynamic tables are looked up at runtime.
Once OpenSMTPD is running,
a static table lookup will always succeed if the table content in the configuration file was valid,
whereas a dynamic table lookup may fail if there's a problem with the external resource.

Regardless of which table is used,
they are always referenced the same in _smtpd.conf_.
Wherever a table is acceptable,
the table can be referenced by its name enclosed in less-than/greater-than signs such as `<aliases>` or `<myaddrs>`:
```
action "foobar" maildir alias <aliases>

match from src <myaddrs> for any action "foobar"
```


## Inlined tables
In the previous section,
all tables were declared with a name regardless of if they were static or dynamic.
In practice there are times when giving a name to a table is cumbersome when it only has one or two static values used only once in a single place.
For this case,
static tables support inlining.
This is essentially declaring the values directly where they are going to be used,
rather than declaring a named table...
just so the name can be referenced.

To write a rule matching two source IP addresses,
we can rely on a static named table:

```
table "myaddrs" { "192.168.1.2", "192.168.1.3" }
match from src <myaddrs> for domain "example.org" [...]
```

But if "myaddrs" is only used in this single place,
we can avoid having to declare a named table and inline the table as such:

```
match from src { "192.168.1.2", "192.168.1.3" } for domain "example.org" [...]
```

The mechanism is the same,
the two examples are identical in terms of how they work,
but one allows reusing the table whereas the other doesn't.
In practice,
what happens is that a named table is created with a generated name and used where the values are inlined.

Now take a deep breath as I reveal a secret only known to a few...

    Pretty much every configurable parameter in the configuration file is an inlined static table.

When you see something like:
```
match for domain "example.org" [...]
```

The parameter to `for domain` is an inlined table consisting of a single entry and is equivalent to writing:

```
match for domain { "example.org" } [...]
```

or writing:

```
table "my_domain" { "example.org" }
match for domain <my_domain> [...]
```

This may look like a small detail but because of this almost every configurable parameter can be replaced with a multi-entry inlined static table:
```
match for domain { "example.org", "example.net" } [...]
```

And since named tables may be used wherever a static table is used,
the ruleset to match a single domain and the ruleset to match hundreds of domains only differ by a table:

```
match for domain "example.org" [...]
match for domain <list_of_hundreds_of_domains> [...]
```

This is what allows OpenSMTPD configuration files to be so concise.
A ruleset rarely grows big because the same few lines of configurations can be used on a laptop to handle a single domain,
or on a mail exchanger to handle thousands of domains for virtual hosting,
simply by having different table contents.


## Dynamic table backends
As mentionned earlier,
dynamic tables are opaque from OpenSMTPD's perspective.
It doesn't really care how they do the lookups as long as they can return a properly crafted answer for a kind of lookup.
Dynamic tables have backends,
pieces of code that implement a specific way of performing a lookup.
There are two official backends,
_file_ and _db_.

The _file_ backend reads values from a file and loads them into a memory representation.
The representation of the values is very efficient and,
if the table values can fit into memory,
using this backend is the most performant choice.
It supports atomic updates so making changes to the file will not update the in-memory view.
Instead a specific command allows a reload that will either be successful and replace the table content,
or fail and leave the previous table content in place.
Once a _file_ table has mapped its content in memory,
it behaves like a static table:
the content is part of OpenSMTPD and lookups can never fail if the content was valid.

The _db_ backend reads values from a Berkeley DB style database.
Instead of reading a file and loading it into a memory representation,
the file is used as the source to compile a binary file database with a structure that's efficient for lookups.
The _db_ backend is slightly less efficiently than the _file_ backend,
but it is better suited for large sets of data as it allows entries to remain on disk rather than in memory.
As it uses the _file_ backend configuration to build its databases,
it is very easy to switch from one to another.
I usually recommend that people start using the _file_ backend and switch to _db_ backend if the values grow large enough that memory footprint becomes an issue.

Other backends exist as third-party addons.
Among them,
the popular _sqlite_ backend,
but there's also a _mysql_ backend,
a _postgresql_ backend and even an _ldap_ backend.
All work very similarly to what was shown in this chapter,
but their resource file is an addon-specific configuration file:

```
table aliases sqlite:/etc/mail/sqlite.conf
table aliases mysql:/etc/mail/mysql.conf
```

They are managed as separate projects to ensure that OpenSMTPD doesn't inherit all of their dependencies.
Anyone can write a custom table backend,
it is fairly easy and doesn't require the cooperation of the daemon.

<hr />

[1](#1) At the exception of DNS lookups, for now.
