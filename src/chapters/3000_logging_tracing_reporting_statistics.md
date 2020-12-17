# Logging, Tracing, Reporting and Statistics

    Being able to see an activity log of where a kid has been going on the Internet is a good thing.
    -- Bill Gates

    Statistics are no substitute for judgment.
    -- Henry Clay


## The internet is chaos
The internet is pure chaos and,
when you run an MX in pure chaos,
you come to realize that _strange things_[<sup>1</sup>](#1) happen every now and then that you need to troubleshoot.
In most situations, there is a very simple explanation:
a user deleting by mistake an e-mail and assuming it was never received,
another assuming an e-mail was lost on your server when it was not received in the first place,
complaints that e-mails sent were not received by the recipients when they are either stuck in queue due to a temporary destination error...
or already accepted by a destination that may be facing the same _strange things_.

Sometimes,
the issue does not have a simple explanation but needs a full investigation,
one that can be particularly stressful if it happens while people are losing their e-mails[<sup>2</sup>](#2).
There is nothing more irritating than having to battle with software to extract informations during an incident so OpenSMTPD comes with multiple means of understanding what is happening at different levels.
Brief and verbose logging as well as
runtime tracing, profiling, states and statistics can provide enough details to shed some light on even the most chaotic situations.
Particular care was taken to make all information available in formats that can easily be pared and processed by standard utilities,
allowing the creation of dashboards for monitoring and the use of command-line utilities for troubleshooting:
pretty much any information is available two `grep(1)` away.


## Logging
As is the case with other MTA software,
OpenSMTPD logs all of its actions.
It doesn't write directly to a file but relies on `syslog(3)` and the `syslogd(8)` daemon,
allowing the system administrator to configure exactly how and where the logs are written,
as well as how they're supposed to be rotated.
The default is to log to a file,
usually _/var/log/maillog_,
but `syslogd(8)` may very well be configured to write the log entries to a third-party application or even send them to a centralized server through the network.

Before OpenSMTPD 6.4,
the logs were designed to be read by both humans and scripts.
The idea was to write the logs with humans in mind,
but in a format that would ease writing of log processing tools.
Huge work was done to come up with the proper format,
one that would be readable and parseable while carrying as much information as possible on a line so scripts could get a lot of value from a single line.
This led to a nice and perfectly usable format,
thought a bit dense for humans and a bit suboptimal for machines.
Users liked it and the format remained in use for many years,
extended every now and then to add new information to some of the log lines.

With OpenSMTPD 6.4,
some of the lines grew with so much information that they became overwhelming.
It became hard to keep trying to accomodate both humans and scripts:
too much information is unreadable to humans but not enough information is a waste for scripts.
A different approach was taken:
logs are for humans and humans only, tools need to use a different mechanism targeted at them.
That doesn't mean that the log format becomes hard to parse but it means that lines will carry the information a human needs to read,
not the information a script wants to gather.
The reporting mechanism for scripts will be discussed in this chapter.

OpenSMTPD supports two log levels: _brief_ and _debug_.
They are logged with `syslog(3)` using the _LOG_MAIL_ facility and the _LOG_INFO_ and _LOG_DEBUG_ levels.
The _debug_ log level is a super-set of the _brief_ log level,
activating a lot[<sup>3</sup>](#3) of additional logs.

### Brief logging
By default,
OpenSMTPD will use _brief_ logging which is a log format that will record any event with just the right level of details to understand what is happening.
The format is straightforward as shown in the following sample:

```
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp connected address=local host=poolp.org
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp message msgid=c405e39a size=303 nrcpt=1 proto=ESMTP
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp envelope evpid=c405e39ad6bbc64f from=<gilles@poolp.org> to=<gilles@poolp.org>
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp disconnected reason=quit
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c9250bba470 mda delivery evpid=c405e39ad6bbc64f from=<gilles@poolp.org> to=<gilles@poolp.org> rcpt=<gilles@poolp.org> user=gilles delay=0s result=Ok stat=Delivered
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c972116a049 smtp connected address=local host=poolp.org
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c972116a049 smtp message msgid=3d4c4a42 size=319 nrcpt=1 proto=ESMTP
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c972116a049 smtp envelope evpid=3d4c4a424797fe79 from=<gilles@poolp.org> to=<gilles.chehade@gmail.com>
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c972116a049 smtp disconnected reason=quit
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9b982c840c mta connecting address=smtp://127.0.0.1:10027 host=localhost
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9b982c840c mta connected
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9c42abeddb smtp connected address=127.0.0.1 host=localhost
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9c42abeddb smtp message msgid=f6e94209 size=802 nrcpt=1 proto=ESMTP
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9c42abeddb smtp envelope evpid=f6e94209918f7e7b from=<gilles@poolp.org> to=<gilles.chehade@gmail.com>
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9b982c840c mta delivery evpid=3d4c4a424797fe79 from=<gilles@poolp.org> to=<gilles.chehade@gmail.com> rcpt=<-> source="127.0.0.1" relay="127.0.0.1 (localhost)" delay=0s result="Ok" stat="250 2.0.0: f6e94209 Message accepted for delivery"
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta connecting address=smtp://173.194.76.27:25 host=ws-in-f27.1e100.net
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta connected
Jul  6 06:01:47 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta tls ciphers=TLSv1.2:ECDHE-RSA-CHACHA20-POLY1305:256
Jul  6 06:01:48 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta delivery evpid=f6e94209918f7e7b from=<gilles@poolp.org> to=<gilles.chehade@gmail.com> rcpt=<-> source="212.83.129.132" relay="173.194.76.27 (ws-in-f27.1e100.net)" delay=2s result="Ok" stat="250 2.0.0 OK  1562385708 z11si6726563wmb.62 - gsmtp"   
```

Each line consists of a date, machine name, process name and identifier that are prepended by `syslogd(8)`:
```
Jul  6 06:01:48 ams-1 smtpd[75295]   
```

Then the log line is appended and follows a simple pattern:
```
5df87c903d709659 smtp connected address=local host=poolp.org
5df87c903d709659 smtp message msgid=c405e39a size=303 nrcpt=1 proto=ESMTP
5df87c903d709659 smtp envelope evpid=c405e39ad6bbc64f from=<gilles@poolp.org> to=<gilles@poolp.org>
```

Each log line is prefixed with a unique session identifier shared by all related log lines,
followed by the subsystem that generated the session,
then the event being logged and finally a serie of event-related informations displayed in key and value pairs.
Lines carry only the information related to the event they are logging which means that most information is never duplicated in multiple lines.

So, how do we work with that.

If you are trying to understand what has happened during a session,
you can search for all events that occurred in the session using the session identifier:

```
$ grep 5df87c903d709659 /var/log/maillog                                                                             
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp connected address=local host=poolp.org
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp message msgid=c405e39a size=303 nrcpt=1 proto=ESMTP
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp envelope evpid=c405e39ad6bbc64f from=<gilles@poolp.org> to=<gilles@poolp.org>
Jul  6 06:01:40 ams-1 smtpd[75295]: 5df87c903d709659 smtp disconnected reason=quit

$ grep 5df87c9ffd0e785d /var/log/maillog
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta connecting address=smtp://173.194.76.27:25 host=ws-in-f27.1e100.net
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta connected
Jul  6 06:01:47 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta tls ciphers=TLSv1.2:ECDHE-RSA-CHACHA20-POLY1305:256
Jul  6 06:01:48 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta delivery evpid=f6e94209918f7e7b from=<gilles@poolp.org> to=<gilles.chehade@gmail.com> rcpt=<-> source="212.83.129.132" relay="173.194.76.27 (ws-in-f27.1e100.net)" delay=2s result="Ok" stat="250 2.0.0 OK  1562385708 z11si6726563wmb.62 - gsmtp"
Jul  6 06:01:58 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta disconnected reason=quit messages=1
```


If instead, you are trying to find a specific bit of information, you can search for specific events:

```
$ grep 'mta tls' /var/log/maillog | tail -5        
Jul  5 22:54:33 ams-1 smtpd[77583]: a48a61d795bbf406 mta tls ciphers=TLSv1.2:ECDHE-RSA-CHACHA20-POLY1305:256
Jul  5 23:26:27 ams-1 smtpd[77583]: a48a63397015c0fa mta tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
Jul  6 04:06:48 ams-1 smtpd[77583]: a48a6eea7778dfbd mta tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
Jul  6 04:39:44 ams-1 smtpd[77583]: a48a705113538487 mta tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
Jul  6 06:01:47 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta tls ciphers=TLSv1.2:ECDHE-RSA-CHACHA20-POLY1305:256

$ grep 'smtp tls' /var/log/maillog | tail -5
Jul  6 06:28:00 ams-1 smtpd[75295]: 5df87dbf30ab2c24 smtp tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
Jul  6 06:28:06 ams-1 smtpd[75295]: 5df87dc1b2315795 smtp tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
Jul  6 06:34:53 ams-1 smtpd[75295]: 5df87e01e48a743a smtp tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
Jul  6 06:34:58 ams-1 smtpd[75295]: 5df87e04567c803c smtp tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
Jul  6 06:54:35 ams-1 smtpd[75295]: 5df87ed36dee57ee smtp tls ciphers=TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
```


And of course,
you can chain your `grep` calls to track a specific event for a specific session:
```
$ grep 5df87c9ffd0e785d /var/log/maillog | grep 'mta connected'
Jul  6 06:01:46 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta connected

$ grep 5df87c9ffd0e785d /var/log/maillog | grep 'mta tls'
Jul  6 06:01:47 ams-1 smtpd[75295]: 5df87c9ffd0e785d mta tls ciphers=TLSv1.2:ECDHE-RSA-CHACHA20-POLY1305:256
```


As you can see, the format is straightforward, compact and very easily searchable.
It may be tempting to assume that you'll have no trouble finding the proper information in case of an incident,
but I cannot emphasize enough how important it is that you familiarize yourself with reading and understanding the mail log _before_ you actually face an incident.
You should send a few incoming and outgoing e-mails,
look at what events are being logged,
see if you can easily find them and correlate them with other events.
In other words: prepare.


### Tracing subsystems
Brief logging provides a big picture that is usually enough to understand the situation,
but unfortunately sometimes it isn't and you need to be able to have a closer look at what is happening.
For example, the brief log may show that a client is connecting then disconnecting without attempting any command:

```
Jul  6 07:30:31 ams-1 smtpd[75295]: 5df8806c16c7267c smtp connected address=127.0.0.1 host=localhost
Jul  6 07:30:31 ams-1 smtpd[75295]: 5df8806c16c7267c smtp disconnected reason=quit
```

While the big picture lets you understand that a connection was accepted and that the client sent a "QUIT" command before submitting a message,
it doesn't let you understand what happened in detail:
did the client even see the banner ?
did the client identifiy itself with HELO/EHLO ?
did the client QUIT after looking at EHLO extensions offered on the server ?

To deal with these cases,
OpenSMTPD provides a tracing mechanism which can be temporarily enabled for a subystem so it can log very detailed informations.
For the log lines above, enabling the SMTP tracing could have highlighted that the client sent a QUIT right after the banner:

```  
5df8806c16c7267c smtp connected address=127.0.0.1 host=localhost
smtp: 0x15d760462000: >>> 220 poolp.org ESMTP OpenSMTPD
smtp: 0x15d760462000: <<< QUIT
smtp: 0x15d760462000: >>> 221 2.0.0: Bye
smtp: 0x15d760462000: STATE_CONNECTED -> STATE_QUIT
5df8806c16c7267c smtp disconnected reason=quit
```

Or it could also have highlighted that the client sent a QUIT after requesting to see the EHLO extension:

```
3b358cf54b092021 smtp connected address=127.0.0.1 host=localhost
smtp: 0x16efa1f6e000: >>> 220 poolp.org ESMTP OpenSMTPD
smtp: 0x16efa1f6e000: <<< EHLO localhost
smtp: 0x16efa1f6e000: STATE_CONNECTED -> STATE_HELO
smtp: 0x16efa1f6e000: >>> 250-poolp.org Hello localhost [127.0.0.1], pleased to meet you
smtp: 0x16efa1f6e000: >>> 250-8BITMIME
smtp: 0x16efa1f6e000: >>> 250-ENHANCEDSTATUSCODES
smtp: 0x16efa1f6e000: >>> 250-SIZE 36700160
smtp: 0x16efa1f6e000: >>> 250-DSN
smtp: 0x16efa1f6e000: >>> 250 HELP
smtp: 0x16efa1f6e000: <<< QUIT
smtp: 0x16efa1f6e000: >>> 221 2.0.0: Bye
smtp: 0x16efa1f6e000: STATE_HELO -> STATE_QUIT
3b358cf54b092021 smtp disconnected reason=quit
```

Either way,
it provides a level of detail that is not available in brief logging and that is often not needed on a day-to-day basis.
In some cases,
tracing can be used to analyze the behavior of a session but most of the time it is used when facing a buggy SMTP implementation and trying to see what is really happening behind an e-mail exchange.
There are multiple tracing subsystems in OpenSMTPD that can help understand about anything that is happening inside the software,
from sessions to the scheduling of envelopes and their writing to a disk.

Some of the tracing susbsystems are really low-level and meant to be read by developers,
like the _imsg_, _mproc_ or _io_ traces which respectively trace all of the inter-process communication,
the packets used in inter-process communication and pretty much all of data input/output occuring in the daemon.
Others, like _smtp_, _mta_, _expand_, _lookup_ or _rules_ are used by both developers and users,
tracing respectively the content of incoming and outgoing sessions, the expansion of aliases, the lookup of values in tables or the ruleset matching.

The following are examples of _expand_, _lookup_ and _rules_ traces,
expanding the address _gilles@poolp.org_ to a local user,
looking up IP source address and recipient e-mail address in tables,
and matching a specific rule:

```
expand: 0x858351e0018: expand_insert() called for address:gilles@poolp.org[parent=0x0, rule=0x0]
expand: 0x858351e0018: inserted node 0x85768d68000
expand: lka_expand: address: gilles@poolp.org [depth=0]
lookup: check "local" as NETADDR in table static:<anyhost> -> found
lookup: check "poolp.org" as DOMAIN in table static:<anydestination> -> found
lookup: check "gilles@poolp.org" as MAILADDR in table static:shithole -> 0
lookup: check "local" as NETADDR in table static:<localhost> -> found
lookup: check "poolp.org" as DOMAIN in table static:<localnames> -> found
rule #2 matched: match from local for local => l01
expand: 0x858351e0018: expand_insert() called for username:gilles[parent=0x85768d68000, rule=0x857661fbf80, dispatcher=0x8583c5c8700]
expand: 0x858351e0018: inserted node 0x85811a68000
expand: lka_expand: username: gilles [depth=1, sameuser=0]
lookup: lookup "gilles" as ALIAS in table static:aliases -> 0
lookup: lookup "gilles" as USERINFO in table getpwnam:<getpwnam> -> "1000:1000:/home/gilles"
expand: 0x858351e0018: expand_insert() called for filter:/usr/local/bin/fdm -m -a mda fetch[parent=0x85811a68000, rule=0x857661fbf80, dispatcher=0x8583c5c8700]                                                                                                                                                       
expand: 0x858351e0018: inserted node 0x85811a69000
expand: lka_expand: filter: /usr/local/bin/fdm -m -a mda fetch [depth=2]
expand: 0x858351e0018: clearing expand tree
```

As you can see,
the tracing output format is much less user-friendly than brief logging.
It is not meant to be used regularly but rather to cope with abnormal situations and gather all of the information useful for a developer to understand what is happening.
Tracing is exposed to users because it can also help them understand issues,
but it is primarily a tool aimed at developers that's not meant to have a stable or pretty output.
Therefore the tracing subsystems are listed in `smtpctl(8)` but not much more documented as they can vary from a release ot another based on implementation details.

An interesting feature about tracing is that it doesn't require OpenSMTPD to be built with particular options,
it is a very light feature that can be enabled in production with no significant overhead.
Anyone facing what seems to be a bug in production can easily enable and disable tracing to gather debugging data without having to shutdown the server or rebuild the software.\\

Traces can be enabled either when OpenSMTPD is started with command line options,
or through the `smtpctl` command:

```
# smtpd -v -T smtp -T mta
# smtpctl trace smtp
# smtpctl trace mta
```

Active traces can be removed at runtime using `smtpctl`:

```
# smtpctl untrace smtp
# smtpctl untrace mta
```

At the time of this writing,
tracing requires OpenSMTPD to be in verbose mode but this is likely to change in the future.


### Verbose logging
Verbose logging,
also known as debugging mode,
is a scary beast:
it is the log level used during development and it logs at a level so great that it can even scare developers sometimes.

The verbose logging is what you resort to when everything else fails,
it provides so much details that you pretty much understand the code paths that the software has taken.
Just for the sake of comparison,
this is what is logged by brief logging when OpenSMTPD starts:

```
# smtpd -d
info: OpenSMTPD 6.4.0 starting
```
   
And this is what is logged by debug logging when the same OpenSMTPD starts:

```
# smtpd -dv
debug: init ssl-tree
debug: init ca-tree
debug: using "fs" queue backend
debug: using "ramqueue" scheduler backend
debug: using "ram" stat backend
info: OpenSMTPD 6.4.0 starting
[...] about three pages [...]
smtpd: setup done
debug: parent_send_config_ruleset: reloading
debug: parent_send_config: configuring pony process
debug: parent_send_config: configuring ca process
debug: ca_engine_init: using RSA privsep engine
debug: init private ssl-tree
debug: smtp: listen on IPv6:::1 port 25 flags 0x400 pki "" ca ""
debug: smtp: listen on IPv6:fe80::1%lo0 port 25 flags 0x400 pki "" ca ""
debug: smtp: listen on 127.0.0.1 port 25 flags 0x400 pki "" ca ""
debug: smtp: listen on IPv6:::1 port 587 flags 0x449 pki "mail.poolp.org" ca ""
debug: smtp: listen on IPv6:fe80::1%lo0 port 587 flags 0x449 pki "mail.poolp.org" ca ""
debug: smtp: listen on 127.0.0.1 port 587 flags 0x449 pki "mail.poolp.org" ca ""
debug: smtp: listen on IPv6:::1 port 10028 flags 0x400 pki "" ca ""
debug: smtp: listen on IPv6:fe80::1%lo0 port 10028 flags 0x400 pki "" ca ""
debug: smtp: listen on 127.0.0.1 port 10028 flags 0x400 pki "" ca ""
debug: pony: rsae_init
debug: pony: rsae_init
debug: pony: rsae_init
debug: pony: rsae_init
debug: pony: rsae_init
debug: pony: rsae_init
debug: smtp: will accept at most 473 clients
```

If tracing is not meant to be user-friendly,
well...
verbose logging is petting tracing for being sooooo cute.
Unless you are a developer or you are facing an issue that absolutely and necessarily requires verbose logging,
you will probably avoid it at all costs: it is the last chance tool.

At the moment,
verbose mode is necessary for tracing but since it's also a developer tools,
there's nothing incoherent here.
It is likely that tracing will no longer require verbose logging in the future because having to endure that level of logging just to see an SMTP session is inhumane.

Verbose logging can be enabled at start time through the command line options to `smtpd` or activated at runtime with the `smtpctl` command:

```
# smtpd -v
# smtpctl log verbose
```
   
It can be stopped at runtime using `smtpctl`:
```
# smtpctl log brief
```


## Statistics
OpenSMTPD comes with a volatile statistics subsystem.
The daemon will start with empty counters that the other subsystems will increase and decrease to provide a real-time overview of what is happening.
These counters can be about high-level concepts such as the number of clients currently connected,
just like they can be about low-level concepts such as the current memory usage of inter-process I/O buffers.
They provide valuable information which allows both postmasters and developers to track daemon behavior at runtime.

The statistics can be retrieved by using the `smtpctl` command:

```
# smtpctl show stats
buffer.lka.control=0
buffer.lka.mda=0
buffer.lka.parent=0
[... truncated ...]
buffer.smtp.mfa=0
buffer.smtp.parent=0
buffer.smtp.queue=0
control.session=1
mda.pending=0
mda.running=0
mta.connector=0
mta.domain=0
mta.envelope=0
mta.host=0
mta.relay=0
mta.route=0
mta.session=0
mta.source=0
mta.task=0
mta.task.running=0
queue.bounce=2
scheduler.delivery.ok=821
scheduler.delivery.permfail=2
scheduler.delivery.tempfail=125
scheduler.envelope=1
scheduler.envelope.incoming=0
scheduler.envelope.inflight=0
scheduler.ramqueue.envelope=1
scheduler.ramqueue.message=1
scheduler.ramqueue.update=0
smtp.kick=1
smtp.session=0
smtp.session.inet4=909
smtp.session.local=5
smtp.tls=0
uptime=94691
uptime.human=1d2h18m11s
#
```

Some people have used the counters to graph delivery versus bounces ratio to detect issues,
while developers have graphed scheduler logic and memory usage to validate their theories.
The statistics aggregation is very cheap and it is a good practice for developers to create counters and collect statistics when working on subsystems within the daemon.

To make them easier to use and process,
the output has been designed so that related counters share a common prefix that can be searched with a simple `grep` command.
The following shows how a postmaster can search for all counters related to the MTA subsystem:

```
# smtpctl show stats | grep ^mta\.
mta.connector=0
mta.domain=0
mta.envelope=0
mta.host=0
mta.relay=0
mta.route=0
mta.session=0
mta.source=0
mta.task=0
mta.task.running=0
#
```

The statistics subsystem is dynamic,
it doesn't keep track of a set of pre-defined counters but creates them on-the-fly during their first use.


## Advanced tracing: profiling
Sometime before the first release of OpenSMTPD,
a lot of work was poured into improving performances to bring it to the level of other SMTP servers.
There were two main areas of concern:
the very intensive inter-process communication imposed by the multi-process design and the cost of disk accesses to the queue.
Knowing the design of the software,
intuitively there was no doubt that these were the main bottlenecks but we lacked proof and numbers.

Two profiling traces were introduced:
one to measure the time spent in inter-process communication,
the other to measure the time spent in every single queue operation.
Using these traces it became possible to see that some processes communicated more than they should have,
exchanged too much data or were too slow processing it,
as well as which queue operations were called during delivery and which ones required to be worked on.

The precise measures allowed pinpointing operations that needed to be optimized,
sometimes down to the level of system call sequences,
but also which operations did not have constant execution time and needed to be rewritten to be predictable.
It made it possible to measure the maximum performance of an API and compare the current performances to it,
as well as determine which parts were performing ok,
which parts needed to be tuned up and which parts were already optimal.

With all the work that was done,
profiling traces are today essentially used to confirm if a change improves current performances or causes regressions.
However,
developers writing custom queues, schedulers, tables or filters can make use of them to make sure their implementations are efficient.
While they can be used to troubleshoot slow operations,
they are really a tool for developers willing to improve performances in the software or writing their own custom bricks.


### profile-imsg traces
OpenSMTPD is event-driven which means that everything it does results from an event it receives.
The events are triggered by messages sent from a process to another or from timers that tick at regular intervals.
When a message is received by a process,
a dispatching function will search which function handles that event and call it to perform the associated task.

By timing in the dispatch function right before it calls the event handler and right after it returns,
it is possible to measure precisely the time spent doing any task within the software.
When the _profile-imsg_ traces are enabled,
everything from a simple statistics increment to a DNS lookup or client connection gets its time of execution logged.
The example below shows measures of the "set a value to a statistics counter" task:

```
profile-imsg: control lka IMSG_STAT_SET 65 0.000908
profile-imsg: control lka IMSG_STAT_SET 68 0.001047
profile-imsg: control lka IMSG_STAT_SET 67 0.000908
profile-imsg: control lka IMSG_STAT_SET 66 0.001886
```

The format is simple,
it consists of the process receiving the message,
followed by the process sending the message,
the message itself,
the length of the message data and the time it took from entering to exiting its handler.
Similar messages should have very close timings as care was taken to make their execution time predictable,
but they should also be as small as possible to optimize performances.


### profile-queue traces
OpenSMTPD uses a very conservative approach and follows a specific sequence of operations to protect e-mails from being lost.
This sequence ensures that no matter what happens,
as long as the operating system and disk are not lying about their state,
the data is never in an inconsistent state:
either it is accepted because it was completely written to a persistent storage,
or it was not accepted.
There is no risk of it being accepted before it is written or only partially written,
even in the case of a power shutdown happening at a very _bad time_[<sup>4</sup>](#4).

This comes at a cost in terms of performances:
multiple operations to ensure everything is fine will necessarily cost more than just writing blindly to disk.
It doesn't mean that the queue is slow,
it just means that it is slower than an unsafe approach because we have to pay a performance penalty price for doing things right.
Since speed is a major goal,
it was necessary to measure the time spent in the queue operations to ensure that the penalty was as low as possible.
If a queue operation takes longer than the system calls it relies upon,
then it means the code is suboptimal and needs to be reworked.

The _profile-queue_ was written to understand precisely which queue operations were taking time and assessing if that time was expected,
if less pressure could be put on disks and if sequences of operations could be rearranged to be more efficient without sacrificing safety.
The following is an example of the output:

```
profile-queue: queue_envelope_walk 0.544201
profile-queue: queue_message_create 0.732912
profile-queue: queue_envelope_create 0.846962
profile-queue: queue_message_commit 1.668082
profile-queue: queue_message_fd_r 0.009569
profile-queue: queue_envelope_delete 0.805965
```

The format is simpler than imsg profiling,
it consists of the queue operation name followed by the timing for the call.
The profiling happens at an API level and not within the queue backend code,
meaning that profiling code can time any custom queue implementing the few basic operations required,
not just the official queue backend shipped with OpenSMTPD.


## Reporting
As explained earlier,
after 6.4 the logs were simplified for humans and this resulted in a bit of information loss for tools parsing them.

The reporting mechanism was introduced with OpenSMTPD 6.5,
providing a stream of events that tools can consume in a format that is easy to parse and that is also versioned.
As of this writing,
the reporting mechanism is only used to report incoming and outgoing SMTP events,
which is the void that was created when simplifying logs,
but the direction that is taken is to have all of the subsystems report their events.
When this is achieved,
we'll essentially have an event bus available and it will be possible to add a lot of new features to OpenSMTPD through external tools reacting to anything happening in the daemon,
without requiring changes made to it.
As a matter of fact,
the statistics mechanism described a few sections above is a first use-case of something that should be rebuilt on top of reporting.

Contrarily to human logs,
the reporting output is very verbose as shown in the following example for a single incoming SMTP session:

```
report|0.4|1572482919.566813|smtp-in|link-connect|cc7627cc19ffc047|mail-wm1-x32c.google.com|pass|[2a00:1450:4864:20::32c]:33521|[2001:19f0:6c01:3d9:5400:1ff:fee7:78b7]:25
report|0.4|1572482919.568165|smtp-in|filter-response|cc7627cc19ffc047|connected|proceed
report|0.4|1572482919.568196|smtp-in|protocol-server|cc7627cc19ffc047|220 in.mailbrix.mx ESMTP OpenSMTPD
report|0.4|1572482919.568198|smtp-in|link-greeting|cc7627cc19ffc047|in.mailbrix.mx
report|0.4|1572482919.579574|smtp-in|protocol-client|cc7627cc19ffc047|EHLO mail-wm1-x32c.google.com
report|0.4|1572482919.579855|smtp-in|filter-response|cc7627cc19ffc047|ehlo|proceed
report|0.4|1572482919.579862|smtp-in|link-identify|cc7627cc19ffc047|EHLO|mail-wm1-x32c.google.com
report|0.4|1572482919.579890|smtp-in|protocol-server|cc7627cc19ffc047|250-in.mailbrix.mx Hello mail-wm1-x32c.google.com [2a00:1450:4864:20::32c], pleased to meet you
report|0.4|1572482919.579894|smtp-in|protocol-server|cc7627cc19ffc047|250-8BITMIME
report|0.4|1572482919.579902|smtp-in|protocol-server|cc7627cc19ffc047|250-ENHANCEDSTATUSCODES
report|0.4|1572482919.579906|smtp-in|protocol-server|cc7627cc19ffc047|250-SIZE 36700160
report|0.4|1572482919.579909|smtp-in|protocol-server|cc7627cc19ffc047|250-DSN
report|0.4|1572482919.579913|smtp-in|protocol-server|cc7627cc19ffc047|250-STARTTLS
report|0.4|1572482919.579940|smtp-in|protocol-server|cc7627cc19ffc047|250 HELP
report|0.4|1572482919.591399|smtp-in|protocol-client|cc7627cc19ffc047|STARTTLS
report|0.4|1572482919.591604|smtp-in|filter-response|cc7627cc19ffc047|tls|proceed
report|0.4|1572482919.591623|smtp-in|protocol-server|cc7627cc19ffc047|220 2.0.0 Ready to start TLS
report|0.4|1572482919.641623|smtp-in|link-tls|cc7627cc19ffc047|TLSv1.2:ECDHE-RSA-AES256-GCM-SHA384:256
report|0.4|1572482919.653095|smtp-in|protocol-client|cc7627cc19ffc047|EHLO mail-wm1-x32c.google.com
report|0.4|1572482919.653392|smtp-in|filter-response|cc7627cc19ffc047|ehlo|proceed
report|0.4|1572482919.653398|smtp-in|link-identify|cc7627cc19ffc047|EHLO|mail-wm1-x32c.google.com
report|0.4|1572482919.653420|smtp-in|protocol-server|cc7627cc19ffc047|250-in.mailbrix.mx Hello mail-wm1-x32c.google.com [2a00:1450:4864:20::32c], pleased to meet you
report|0.4|1572482919.653427|smtp-in|protocol-server|cc7627cc19ffc047|250-8BITMIME
report|0.4|1572482919.653433|smtp-in|protocol-server|cc7627cc19ffc047|250-ENHANCEDSTATUSCODES
report|0.4|1572482919.653439|smtp-in|protocol-server|cc7627cc19ffc047|250-SIZE 36700160
report|0.4|1572482919.653442|smtp-in|protocol-server|cc7627cc19ffc047|250-DSN
report|0.4|1572482919.653446|smtp-in|protocol-server|cc7627cc19ffc047|250 HELP
report|0.4|1572482919.665304|smtp-in|protocol-client|cc7627cc19ffc047|MAIL FROM:<gilles.chehade@gmail.com> SIZE=2393
report|0.4|1572482919.665623|smtp-in|filter-response|cc7627cc19ffc047|mail-from|proceed
report|0.4|1572482919.667944|smtp-in|tx-begin|cc7627cc19ffc047|0cd3aa24
report|0.4|1572482919.667954|smtp-in|tx-mail|cc7627cc19ffc047|0cd3aa24|gilles.chehade@gmail.com|ok
report|0.4|1572482919.667964|smtp-in|protocol-server|cc7627cc19ffc047|250 2.0.0 Ok
report|0.4|1572482919.679517|smtp-in|protocol-client|cc7627cc19ffc047|RCPT TO:<gilles@poolp.org>
report|0.4|1572482919.679806|smtp-in|filter-response|cc7627cc19ffc047|rcpt-to|proceed
report|0.4|1572482919.682889|smtp-in|tx-envelope|cc7627cc19ffc047|0cd3aa24|0cd3aa24acdd1899
report|0.4|1572482919.682900|smtp-in|tx-rcpt|cc7627cc19ffc047|0cd3aa24|gilles@poolp.org|ok
report|0.4|1572482919.682907|smtp-in|protocol-server|cc7627cc19ffc047|250 2.1.5 Destination address valid: Recipient ok
report|0.4|1572482919.694237|smtp-in|protocol-client|cc7627cc19ffc047|DATA
report|0.4|1572482919.694461|smtp-in|filter-response|cc7627cc19ffc047|data|proceed
report|0.4|1572482919.695327|smtp-in|tx-data|cc7627cc19ffc047|0cd3aa24|ok
report|0.4|1572482919.695334|smtp-in|protocol-server|cc7627cc19ffc047|354 Enter mail, end with "." on a line by itself
report|0.4|1572482920.718062|smtp-in|protocol-client|cc7627cc19ffc047|.
report|0.4|1572482920.718353|smtp-in|filter-response|cc7627cc19ffc047|commit|proceed
report|0.4|1572482920.718469|smtp-in|tx-commit|cc7627cc19ffc047|0cd3aa24|2859
report|0.4|1572482920.718471|smtp-in|tx-reset|cc7627cc19ffc047|0cd3aa24
report|0.4|1572482920.721023|smtp-in|protocol-server|cc7627cc19ffc047|250 2.0.0 0cd3aa24 Message accepted for delivery
report|0.4|1572482920.733447|smtp-in|protocol-client|cc7627cc19ffc047|QUIT
report|0.4|1572482920.742012|smtp-in|filter-response|cc7627cc19ffc047|quit|proceed
report|0.4|1572482920.742032|smtp-in|protocol-server|cc7627cc19ffc047|221 2.0.0 Bye
report|0.4|1572482920.742189|smtp-in|link-disconnect|cc7627cc19ffc047
```

The format is very simple, it begins with a common prefix for all subsystems:

```
<message>|<version>|<timestamp>|<subsystem>
```

Followed by event messages with their specific optional paramters:

```
<message>|<version>|<timestamp>|<subsystem>|<event>|<param>
```

So taking one random line from above and dissecting it:
```
report|0.4|1572482919.667954|smtp-in|tx-mail|cc7627cc19ffc047|0cd3aa24|gilles.chehade@gmail.com|ok
```

We have a message of type "report",
respecting version 0.4 of the reporting protocol,
that was emitted at 1572482919.667954 (unix timestamp),
from the subsystem "smtp-in".
The message informs us that an event "tx-mail" occured,
providing us the session identifier for the client that issues the MAIL FROM,
the transaction identifier (message id) for the current transaction,
the e-mail address that was submitted by the session,
and the result for that sender.

As you can see,
the lines are very easy to parse,
very complete in terms of informations provided,
and tools can easily aggregate whatever information they want to do their work.
This is far more powerful than the parsing of user logs,
while not being any more complex.
A very interesting side-effect is that tools can parse this stream from a log file,
performing asynchronous processing,
or they can plug directly to the daemon's filtering API to receive these events in real-time,
performing trigger-based processing.

Describing all these events would be pages and pages of details,
I don't think it's necessary at this point to give that information in this book,
mostly because pretty much all events are straightforward to understand.
They will be part of the OpenSMTPD documentation,
future revisions of the book may revisit them if there's demand.

For the time being,
knowing that such a mechanism exists and lets you build tools around OpenSMTPD is enough to point you at the right direction.
From extracting IP addresses for firewall-blocking clients to generating advanced dashboards in a reporting tool,
anything that requires programmatic processing of logs is better served by the reporting mechanism.

<hr />

[<sup>1</sup>](#1) Commonly known as "where the f did this e-mail go" situations.

[<sup>2</sup>](#2) You have never seen an angry person until you tell someone that you lost their e-mails.

[<sup>3</sup>](#3) A LOOOOOOOOOOOT.

[<sup>4</sup>](#4) Including "sorry, I tripped on the cable" time.