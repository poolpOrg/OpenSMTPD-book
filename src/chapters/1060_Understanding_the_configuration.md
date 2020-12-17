# Understanding the configuration

    XXX
    -- General Lee


## Understanding the configuration philosophy

Like other software coming from the OpenBSD universe,
OpenSMTPD has a philosophy of keeping things simple and stupid:
things should just work out of the box,
with sane defaults and minimal need of tuning.
If tuning is absolutely necessary,
then it should be as simple as reading a man page and looking at the examples.

A postmaster should be able to setup an MX in just a few minutes,
without prior knowledge of the software,
and be able to spot configuration errors by reading the configuration out loud.
MX are a particularly bad choice of software to misconfigure:
they operate both as server and client and mistakes can cause a wide variety of problems ranging from rejection of legitimate e-mails to spreading of spam to the world.
You can't prevent all configuration mistakes but you can try to make them as obvious as possible.

OpenSMTPD does not provide buttons to tweak every single aspect of the software, this would go against the philosophy.
Whenever users request new ways to tweak the daemon's behaviour,
developers debate to understand the use-case and figure if a better default behaviour could not be provided instead of adding a new knob.
New configuration options are only added if there's consensus that the use-case is legitimate,
that it can't be handled by default,
and that adding a new option is the best option for simpler code.
Sometimes new use-cases extend previous ones and the grammar evolves slightly by replacing a previous option.

This way of integrating new features allows the configuration file to remain very simple,
only providing the options that are truly necessary.


## The default _smtpd.conf_ file
OpenSMTPD is configured through _smtpd.conf_ which contains the entire configuration.
It may also include other files so that the configuration can be split into separate modules,
but most configuration files are very small and this feature is rarely used in practice.

The default _smtpd.conf_ provides a functional,
yet very restrictive,
configuration.
It allows local users to send mail to other users,
local and remote,
but only accepts submissions from the local machine.
It is suitable for desktops,
laptops or servers that send mail notifications to other machines but do not accept mail from other mail exchangers.
The following is what's used on OpenBSD systems after install,
these six lines give you a fully working local server:

```
listen on lo0

table "aliases" file:/etc/mail/aliases

action "local_users" mbox alias <aliases>
action "relay_users" relay

match from any for local action "local_users"
match from local for any action "relay_users"
```

It can't get much simpler...

Let's go through the same file with comments to describe what it does:

```
# Allow network connections on interface lo0 (on Linux, eth0)
# This automagically listens on IPv4 and IPv6.
#
listen on lo0

# A table named "aliases" may be used to query the file /etc/mail/aliases
# The name "aliases" was freely chosen, we could have named it "bananas".
#
table "aliases" file:/etc/mail/aliases

# Define two actions, also freely named:
# "local_users", delivers to a mailbox using the alias table "aliases"
# "relay_users", relays to another MX
#
action "local_users" mbox alias <aliases>
action "relay_users" relay

# Define two matching patterns:
# first matches any source for local recipients and assigns action "local_users"
# second matches local sources for any recipient and assigns action "relay_users"
#
match from any for local action "local_users"
match from local for any action "relay_users"
```

Most common use-cases are slight variations from this default configuration:
listening on a different interface,
delivering to a different mailbox format and/or
accepting envelopes for specific domain names.
The configuration difference between this default "local" mail exchanger and an internet-facing mail exchanger is relatively small.
To illustrate,
here's an example configuration that no longer operates as a "local" mail exchanger but as a real internet-facing mail exchanger for the _poolp.org_ domain:

```
# listen on all interfaces
# on OpenBSD, interface groups may be used (ie: listen on all)
#
listen on 0.0.0.0

table "aliases" file:/etc/mail/aliases

action "local_users" mbox alias <aliases>
action "relay_users" relay

match from any for domain "poolp.org" action "local_users"
match from any for local action "local_users"
match from local for any action "relay_users"
```

This new configuration,
which only differs from the default configuration by two lines,
is enough to run an internet-facing mail exchanger that accepts mail for my domain.


## _smtpd.conf_ is a ruleset
In most MTA software,
the configuration file consists of configuration options to set MTA parameters and enable or disable features.
A hostname is set here,
an option is disabled there,
and the combination of these determine result in a global configuration which tells the MTA what it should do.

OpenSMTPD takes a different approach:
it considers the configuration file as a ruleset similar in many ways to a firewall configuration.
The configuration has a few bits to setup the MTA parameters,
such as which interface it should listen to and if it should do TLS,
but most of the configuration defines a set of rules describing envelope templates that it wants to handle.

This makes a lot of sense because a mail exchanger is really just a router:
it receives messages and decides if it wants to route them closer to their destination.
So when an envelope is submitted by a client to OpenSMTPD,
it will compare it to each rule of the ruleset.
If a match is found,
the envelope is accepted and the action associated to the rule is used to handle the delivery,
otherwise the envelope is rejected.

What needs to be highlighted is that the ruleset is evaluated with a _first-match policy_.
An envelope goes through the rules in sequential order until a rule matches it and causes the evaluation to stop.
This means that rules must be written from the most specific to the most generic:

```
# client 192.168.1.1 will match first rule,
# others will match second rule
#
match from src 192.168.1.1 for any action "action1"
match from any for any action "action2"

# clients always match the first rule,
# the second rule is never evaluatred
#
match from any for any action "action1"
match from src 192.168.1.1 for any action "action2"
```

The ruleset matches _envelopes_, not sessions.
This is a key concept to grasp because it means that rules expect entire envelopes,
with a sender e-mail address and a recipient e-mail address to take their decision.
They are not used to filter sessions,
they can't be used to reject a client coming from a blacklisted IP address right after it connects.
This is the role of a different mechanism,
_filtering_,
which is meant to control how a session is allowed to progress and if it can even reach the ruleset evaluation.
The _filtering_ mechanism will be described in an upcoming chapter.

Furthermore,
if you recall the explanations about how SMTP works,
the decision to accept an envelope is taken individually for each recipient address _before_ the message is emitted by the client.
For this reason,
rules can only evaluate envelopes using protocol-level informations:
they can describe that a valid envelope must come from a specific IP address,
over a TLS-secured channel,
from a specific sender domain to a specific recipient...
but they can't use the actual message content as that would require a trip back and forth to the future.
Taking a decision to reject a message is also the role of _filtering_.

A very good analogy for that is an actual physical mail sorting center:
the decision to move mail from a city to another or to return undeliverable mail to its sender is (in mooooooost cases) taken using the envelope,
not by reading the letter inside the envelope.
The ruleset is just a software simulation of a sorting center.

Finally,
matching a rule doesn't immediately trigger the action associated to it:
the message is not available yet to actually do something with it.
The ruleset only determines if an envelope will be accepted for processing,
it doesn't perform the delivery.
The ruleset associates the action to the envelope so that it can be evaluated once the message is accepted.


## A few words about tables
For an envelope to be accepted,
OpenSMTPD must determine that it can do something with it by looking at the ruleset.
However,
part of the decision relies on information that can't realistically be inlined in the ruleset itself.
For instance looking at the following ruleset:

```
listen on 0.0.0.0

table "aliases" file:/etc/mail/aliases

action "local_users" mbox alias <aliases>
action "relay_users" relay

match from any for local action "local_users"
match from local for any action "relay_users"
```

We can't expect the action "local_users" to list all the aliases which on my system range around 80,
this would result in an action listing all possible aliases as shown below:

```
action "local_users" mbox alias { root = gilles, postmaster = gilles, ...}
```

This would not only be highly impractical after the first few dozens of entries but it would also mean that the configuration file needs to be edited whenever there's a change made to the list.
Then the same would be required for system accounts,
since OpenSMTPD needs to know if _gilles_ exists on the system before accepting the mail.
The exhaustive list of system users would need to be listed in the configuration and synchronized with the system user database.
The ruleset is a static configuration but some parts of it require a bit of dynamism and the ability to perform runtime lookups to external resources.

To solve this,
the "tables" mechanism was introduced to allow querying information at runtime.
In the "aliases" example above,
a table is declared as a query interface to _/etc/mail/aliases_,
and an action is told to use that table when it wants to lookup aliases.
At runtime,
when the ruleset engine wants to lookup if an alias exists,
it asks the table to perform the lookup and return results,
if any.

The same applies to system users.
Unless the configuration requests lookup of users through a table:
```
  listen on 0.0.0.0

  table "aliases" file:/etc/mail/aliases
  table "userbase" file:/etc/mail/userbase

  action "local_users" mbox alias <aliases> userbase <userbase>
  action "relay_users" relay

  match from any for local action "local_users"
  match from local for any action "relay_users"
```

OpenSMTPD uses an internal table that operates as an interface to the system user database,
querying it at runtime about usernames it wants to lookup so that they don't have to be listed in the configuration.

The information doesn't have to be known in advance by the configuration file,
the daemon doesn't have to be restarted to be made aware of it,
the table will take care of dynamically resolving the lookup after the daemon is started.

The next chapter will cover tables with more details.
