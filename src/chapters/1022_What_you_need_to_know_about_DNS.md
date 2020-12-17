# What you need to know about DNS

    Letitia!
    What a name.
    Halfway between a salad and a sneeze.
    -- Terry Pratchett


## The DNS protocol
DNS, the Domain Name Service, is an essential component of modern networks.
Without it,
we would be required to remember things like 88.190.237.114,
2a01:e0b:1000:23:be30:5bff:fed7:9af or even 127.0.0.1

DNS is not a very complex service but it provides considerably more than simply resolving names to addresses.
You can manage to survive with basic knowledge of how it works but the SMTP network relies greatly on the DNS protocol for many purposes and you won't be comfortable with SMTP until you are relatively comfortable with DNS.

As a matter of fact,
most SMTP failures result from problems taking place at the DNS level and a broken DNS setup will invariably result in broken SMTP exchanges.

Let's have a look at how DNS works and how SMTP interacts with it without worrying (yet) about more advanced topics like DKIM, SPF, DMARC or DANE which make use of the DNS system to "enhance" the SMTP protocol.
I will also pretend that IPv6 doesn't exist for now,
mainly because it will allow me to keep the examples simple.


### DNS zones
DNS servers store the informations regarding the domains they handle in _zones_.
The _ns-master.poolp.org_ nameserver is in charge of the domain _opensmtpd.org_ and has a zone file describing the details regarding this domain.
The zone for _opensmtpd.org_ contains the following (irrelevant parts truncated):

```
                NS      ns-master.poolp.org.
                NS      ns-backup.poolp.org.
                A       82.65.169.200
www             A       82.65.169.200
opensmtpd.org.  MX 0    mx-in.poolp.org.
opensmtpd.org.  MX 50   mx-backup.poolp.org.
```

The _opensmtpd.org_ zone declares two nameservers,
_ns-master.poolp.org_ and _ns-backup.poolp.org_.
It then declares that _opensmtpd.org_ and _www.opensmtpd.org_ are resolved as the IP address _82.65.169.200_,
and more interestingly to us postmasters,
it also declares that _opensmtpd.org_ has two mail exchanger records:
_mx-in.poolp.org_ with a preference of 0 and _mx-backup.poolp.org_ with a preference of 50.
This is done through the use of MX records.

Note that records may have relative labels,
as is the case with _www_ which is relative to the zone domain _opensmtpd.org_,
or absolute as is the case with any label ending with a dot.


### MX records
The MX records are a particular kind of record which lists the Mail eXchangers that handle mail for a particular domain or subdomain.
To be able to handle mail correctly,
a domain must list at least one MX record but may list several for a specific domain or subdomains.
The example above shows how different mail exchangers can be configured to take care of specific subdomains or domains:

```
opensmtpd.org.          MX 0    mx-in.poolp.org.
opensmtpd.org.          MX 10   mx-backup.poolp.org.
subnet1.opensmtpd.org.  MX 0    mx1.example.org.
subnet2.opensmtpd.org.  MX 0    mx1.example.com.
```

The mail exchangers _mx-in.poolp.org_ and _mx-backup.poolp.org_ both handle the domain _opensmtpd.org_,
while _mx1.example.org_ handles _subnet1.opensmtpd.org_ and _mx1.example.com_ handles _subnet2.opensmtpd.org_.
Dead simple.


### MX lookup
SMTP servers make use of DNS for many purposes,
like looking up the hostname matching the IP address of a client for example,
but the most important use upon which the entire SMTP protocol is built is _MX lookup_.
If you read this far,
you probably understood that this boils down to finding which MX are responsible for the destination domain of a message.

Let's pretend that Eric Faurot and I are exchanging mails.
I (gilles@poolp.org) want to send an email to Eric Faurot (eric@faurot.net) so I enter his e-mail address in my MUA (mutt) and send it.

Behind the scene,
my MUA _somehow_ submits the message to a configured SMTP server.
The SMTP server accepts to take responsability for the mail,
queues it for relaying and returns a transaction identifier to my MUA with an OK status for the message.
My MUA doesn't care much about this and doesn't tell me anything since this is a normal situation and in a sane Unix world you only become chatty when things go wrong.

Then,
the magic happens on the server side:
my SMTP server performs an _MX lookup_.
It requests its configured nameserver to perform the lookup of MX records for domain _faurot.net_.
The nameserver will check if it has the information in cache and otherwise will locate which nameservers are in charge of _faurot.net_.

```
$ dig +nocomments -t NS faurot.net |grep NS 
; <<>> DiG 9.4.2-P2 <<>> +nocomments -t NS faurot.net
;faurot.net.                    IN      NS
faurot.net.             10757   IN      NS      c.dns.gandi.net.
faurot.net.             10757   IN      NS      a.dns.gandi.net.
faurot.net.             10757   IN      NS      b.dns.gandi.net.
$
```

It can then request either one to return the MX entries for the domain _faurot.net_:

```
$ dig +nocomments -t MX faurot.net |grep MX 
; <<>> DiG 9.4.2-P2 <<>> +nocomments -t MX faurot.net
;faurot.net.                    IN      MX
faurot.net.             10496   IN      MX      50 fb.mail.gandi.net.
faurot.net.             10496   IN      MX      10 spool.mail.gandi.net.
$ 
```

At this point,
it provides my SMTP server with the MX records it found:
_fb.mail.gandi.net_ with a preference of 50 and _spool.mail.gandi.net_ with a preference of 10.
My SMTP server reorders the mail exchangers so that the ones with the lowest preference numbers have the highest priority.
In this case,
it puts _spool.mail.gandi.net_ in first position and _fb.mail.gandi.net_ in second position.

My SMTP server then performs a DNS _A lookup_ on the first mail exchanger to resolve the name to an address,
attempts to connect to it and submit the message there.

Usually,
that will be the end of it as far as my MX is concerned,
it had committed to deliver the message to its destination and that's what it did.


### Backup MX
Due to the quantic nature of our universe,
an MX may be in two overlapping states:
it may be willing to accept a message but it may also be unable to accept it at the same time.

The MX may have a dying drive failing disk writes at random,
it may be lacking disk space all of a sudden due to a surge of messages with large attachements,
it may even work strangely due to a broken DNS setup<sup>[1](#1)</sup>.
A variety of reasons can prevent it from committing to deliver a message.

That's if its even reachable at all.

The MX could suffer a power outage,
a network outage,
someone tripping on the cable,
or even an exhausted sysadmin accidentally typing `halt` in the wrong terminal<sup>[2](#2)</sup>.

This is where preferences come into play.
Each MX server for a domain has a preference value associated to its MX record in the zone.
The preference is a numerical value which determines in what order MX servers should be attempted by a remote MX when trying to transfer messages for the domain.
The lower the preference value is, the highest priority the MX gets.
This mechanism can be use to distribute load over pools of machines,
to prioritize faster servers,
or to create... backup MX.

```
faurot.net.   10496   IN      MX      10 spool.mail.gandi.net.
faurot.net.   10496   IN      MX      50 fb.mail.gandi.net.
```

From my MX point of view,
there is no difference between these two MX servers except for the requirement to attempt one before the other.
It considers _spool.mail.gandi.net_ as closer to the destination than _fb.mail.gandi.net_,
though from a purely technical perspective it could send to either one and be discharged of its responsability with regard to the mail as they both are declared as destination MX for the domain in the zone.

Any of the MX for a domain may accept messages for that domain,
regardless of their preference value,
but since an MX commits to bring a message closer to its destination,
it implies that an MX that's not the destination will only attempt to forward a message to another MX with a lower preference value.
These MX servers are called _backup MX_ because they provide a backup service to the _primary MX_ servers in case they are unreachable,
and commit to either attempt passing them the messages until they succeed or acknowledge the original sender that an error occurred.

Note that the preference value is only here to create a priority-sorted list of MX servers to contact.
There is no requirement that a priority 10 exists,
there is no rule preventing a zone from using preferences ranging from 100 to 200 with 100 being its _primary MX_.
Basically,
nothing prevent postmasters to give the preference they want to the MX they want and the sender MX should not think in terms of primary or backup MX,
but in terms of what's the MX with the lowest value that it has not tried yet.

Let's get back to my mail exchange and pretend that _spool.mail.gandi.net_ is temporarily unavailable.
My MX fails to connect and looks at the next entry in the list which is _fb.mail.gandi.net_.
It resolves the name to an address,
then attempts to connect and passe over the message.

At this point there are three possible outcomes:
- I'm being unlucky, the backup MX is also temporarily unable to handle the mail.
    Since the zone does not provide a third MX to contact,
    my MX is stuck with the mail and will have to reattempt a transfer later.

- The MX has explicitely rejected the mail permanently.
  There is no need to try again later in hope that another MX would accept it,
  a backup MX is not allowed to reject a message that could be accepted by a primary MX,
  so all destination MX can be considered as authoritative in their rejections.

- The MX has accepted the message.
  As far as my MX is concerned,
  another MX is now responsible for not letting the message vanish,
  work is over.


<hr />

[<sup>1</sup>](#1) By now you know this is the first thing to rule out.

[<sup>2</sup>](#2) This is a work of fiction. Names, characters, businesses, places, events, locales, and incidents are either the products of the author's imagination or used in a fictitious manner. Any resemblance to actual persons, living or dead, or actual events is purely coincidental.
