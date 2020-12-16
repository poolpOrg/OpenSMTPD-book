# Setting up your own first MX

## A few words before we dive in
This chapter assumes that you have a relatively good understanding of how SMTP and DNS work,
as well as basic understanding of how the configuration file works.
If you jumped right here,
impatient grasshopper,
I highly recommend that you take the time to read the first part of the book,
you will not go very far in this journey with a "test \& try" strategy.
I'll wait right here, don't worry.

Good.

I knew you'd be back.

This chapter will go through the process of setting up an MX interoperating with other MX in the SMTP network.
It will start with a very basic MX and progressively move to more advanced setups.
All use-cases cannot realistically be covered,
but my goal is to cover a wide set of use-cases ranging from a small personal server to a corporate high-volume mail server,
and provide you with an understanding of how you can deal with your own specific use-cases by yourself.

In this chapter,
we will be setting up an MX for the domain "hypno.cat".


## Requirements for a working mail exchanger
There are very few requirements to run a basic MX and,
as long as they are met,
you will be able to exchange mails from your domain to the rest of the world in just a few minutes.

First of all,
you need a machine that will operate as the MX.
For the time being,
let's not worry about CPU, RAM or disk performance concerns,
any machine that you can keep powered-on will do.
You'll want a machine that's reliable though:
until we have discussed how to set up secondary MX,
any downtime of that machine might cause loss of e-mails.

With that machine,
you need an Internet access that does not filter incoming or outgoing SMTP trafic:
in other words, you should be able to emit or receive connections to TCP port 25 (noted 25/tcp from now on).
To prevent home computers infected with malware from spamming the world,
many ISP have disallowed emitting or receiving SMTP trafic from a regular home connection.
More recently,
to avoid poorly-maintained servers from being compromised and used as spamming proxies,
many hosting companies have started applying the same restrictions to the servers they rent.
YES, there are hacks and workarounds but you want a reliable setup and this does not begin with a hack.
Note that some providers filter 25/tcp by default but can remove the filter on request or through a control panel or by contacting their support.

Finally,
you need to have control over the DNS zone for the domain you intend to emit and receive e-mail for.
We will be making changes to the zone so the other MX can find yours but also for some of the more advanced use-cases.
SMTP is VERY tightly coupled with DNS, you will not be able to do much unless you have control over the DNS zone.

That's enough _technically_ to setup a working MX.

In practice,
if you are serious about running your own MX,
you will have few additional requirements which will be covered in the next chapter.


## Setting up the DNS
This book not being about DNS,
I'll keep this section as short as possible but I really, really, REALLY recommend that you to get familiar with the protocol and your DNS server software.

Any DNS server software will do the job just fine,
I will simply use NSD in my examples since that's what's available on OpenBSD base system.
Some registrars will allow you to manage your domains through their own interface,
as a result you don't even need to run your own DNS server,
all you need is to be able to configure records for your domain.


### The hypno.cat zone

We will start with a very simple zone that's not SMTP-capable and build from there:

```
$ORIGIN hypno.cat.
$TTL 6h
@       IN      SOA     hypno.cat. root.hypno.cat. (
                        20190705084300  ; serial
                        1h              ; refresh
                        30m             ; retry
                        7d              ; expiration
                        1h )            ; minimum
                NS      ns-master.poolp.org.
                NS      ns-backup.poolp.org.
@               A       82.65.169.200
www             A       82.65.169.200
```

$ORIGIN allows us to set the domain to which host names are relative.
Note the trailing dot which differentiates absolute names from relative names.
The $ORIGIN can be expressed as @ in records so the following records are equivalent:

```
@               A       82.65.169.200
hypno.cat.      A       82.65.169.200
```

By setting it to "hypno.cat.",
we can declare a record "www" without having to specify the full name "www.hypno.cat.".
The following records are also equivalent:

```
www             A       82.65.169.200
www.hypno.cat.  A       82.65.169.200
```


$TTL allows providing a default time-to-live value for records that don't have one.
The value you set will have a huge impact on how often other DNS servers query yours,
you will want to investigate what value is right for you.

The SOA record,
short for Start Of Authority,
contains administrative informations regarding the zone.
The details are not really in the scope of this section so I'll just point my fingers at the values in parenthesis:
the serial is simply a value incremented when you make changes to the zone so secondary servers pick up the change,
the refresh, retry, expiration and minimum values are TTL values that inform other DNS servers about how often they should check for changes in your zone.
When resolving a name in your zone,
most DNS resolvers will cache the result to avoid having to look it up again through your DNS if another query came up right away.
The TTL values in your SOA record inform these resolvers how often they should expire their cache:
low values mean your zone changes get propagated faster at the cost of more pressure put on your DNS server,
higher values mean your DNS server is under less pressure but zone changes take longer to propagate.
A good approach is to use low TTL values during the setup of your infrastructure,
then increase the values when your zone is stable and rarely changes.

The NS records,
short for Name Server,
point to the nameservers holding the zone for the domain.
Here, "ns-master.poolp.org." and "ns-backup.poolp.org." are authoritative for the "hypno.cat." domain,
these are the ones who hold the truth regarding that zone.

Finally, the A records map an IPv4 address to a name.
So the following record states that both hypno.cat and www.hypno.cat will resolve to 82.65.169.200:
```
@               A       82.65.169.200
www             A       82.65.169.200
````

These are not the only records available in DNS,
far from it,
but we have to start somewhere so there we are.
The following subsections will introduce the A,
PTR and MX records which are the most basic ones that we need to be reachable.
Other records used in more advanced setups will be discussed later.


### Configuring PTR, A and MX records

Now that we know the basics about how to setup a DNS zone,
we need to do the proper plumbing for our MX.

#### Configuring the PTR record
We'll start with the PTR record because it is a bit special:
the zone I described above is used for "forward DNS" lookups,
but "reverse DNS" is handled in a somewhat special zone.

This could easily grow into a full chapter so let's keep it simple:
unless you own the address range of your IP address, you don't have control over that zone.
If you own the address range you should already know about arpa zones or rush to buy a DNS book.

Your ISP or server hosting company leasing you the IP address are the ones in charge of that zone.
In most cases they'll just provide you with a form so you can declare what hostname the IP address should resolve to,
translating it to a proper PTR record in their zone.
The IP address of my MX is 82.65.169.200 and I want it to be named "mx.hypno.cat.",
so using the form provided by my server hosting company I declare the name which translates to the following record:

```
200.169.65.82.in-addr.arpa.    PTR mx.hypno.cat.
```

This can be verified using the "nslookup" command:
```
$ nslookup 82.65.169.200
Server:         62.210.16.6
Address:        62.210.16.6#53
Non-authoritative answer:
200.169.65.82.in-addr.arpa     name = mx.hypno.cat.
Authoritative answers can be found from:
129.83.212.in-addr.arpa nameserver = nsb.online.net.
129.83.212.in-addr.arpa nameserver = nsa.online.net.
$
```

#### Configuring the A record}
We have declared the reverse DNS record for "mx.hypno.cat." so that looking up the IP address can gives us the name but this is not enough.
We also need to declare in our zone the forward DNS record so that looking up the name can give us the IP address.
This is as simple as adding an A record in the hypno.cat. zone and bumping (increasing the value for) the serial in the SOA record:

```
$ORIGIN hypno.cat.
$TTL 6h
@       IN      SOA     hypno.cat. root.hypno.cat. (
                        20190705094400  ; serial
                        1h              ; refresh
                        30m             ; retry
                        7d              ; expiration
                        1h )            ; minimum
                NS      ns-master.poolp.org.
                NS      ns-backup.poolp.org.
@               A       82.65.169.200
www             A       82.65.169.200
mx              A       82.65.169.200
```

With this new record,
it is now possible to resolve "mx.hypno.cat." into 82.65.169.200,
as we'll verify using "nslookup":

```
$ nslookup mx.hypno.cat
Server:         62.210.16.6
Address:        62.210.16.6#53
Non-authoritative answer:
Name:   mx.hypno.cat
Address: 82.65.169.200
$
```

Having a name resolve to an IP address and the IP address resolve to the same name back is called FCrDNS,
short for Forward-Confirmed Reverse DNS,
and is an essential requirement to being able to deliver mail successfully to most MX:
a mismatching FCrDNS will often result in e-mail being blocked, delayed or assumed to be from a sender with a poor reputation.


#### Configuring the MX record
At this point,
the "mx.hypno.cat." name can be resolved successfully but other MX do not know that it is an MX as it is no different from the "www.hypno.cat." record.
Here comes the MX record,
which is how a zone declares which machines are in charge of handling mail exchanges.

There's a lot to say about MX records and we'll have the zone evolve in this book as we tackle more advanced use-cases,
but for now let's keep it simple and add just one MX record:

```
$ORIGIN hypno.cat.
$TTL 6h
@       IN      SOA     hypno.cat. root.hypno.cat. (
                        20190705094400  ; serial
                        1h              ; refresh
                        30m             ; retry
                        7d              ; expiration
                        1h )            ; minimum
                NS      ns-master.poolp.org.
                NS      ns-backup.poolp.org.
@               A       82.65.169.200
www             A       82.65.169.200
mx              A       82.65.169.200
```

The MX records states that mail for "hypno.cat." is to be handled by "mx.hypno.cat.".
The number "0" is the selection preference,
this is irrelevant for our setup right now as we only have one MX,
we'll look into it again when we add secondary MX for redundancy.

With the record,
whenever someone sends a mail to someone with an e-mail address ending with @hypno.cat,
the DNS resolution will result in the e-mail being transferred to "mx.hypno.cat.".
We can check that this MX resolution is properly done using the "dig" utility,
the output is a bit bloated but the interesting part is the ANSWER SECTION:

```
$ dig -t MX hypno.cat
; <<>> DiG 9.4.2-P2 <<>> -t MX hypno.cat
;; global options:  printcmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39566
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 1
;; QUESTION SECTION:
;hypno.cat.                     IN      MX
;; ANSWER SECTION:
hypno.cat.              21600   IN      MX      0 mx.hypno.cat.
;; AUTHORITY SECTION:
hypno.cat.              21600   IN      NS      ns-master.poolp.org.
hypno.cat.              21600   IN      NS      ns-backup.poolp.org.
;; ADDITIONAL SECTION:
mx.hypno.cat.           21600   IN      A       82.65.169.200
;; Query time: 26 msec
;; SERVER: 62.210.16.6#53(62.210.16.6)
;; WHEN: Fri Jul  5 09:59:50 2019
;; MSG SIZE  rcvd: 109
$
```

With the zone configured as such,
our machine is now reachable by other MX machines that want to transfer e-mails to our domain.


## Setting up the MX
Having a properly configured DNS is of no use if no MX is listening for e-mail transfers,
so this section will now focus on getting one ready for handling e-mails for our domain.

### The most basic MX
The most basic MX is one that will simply allow receiving e-mail for our domain and putting it into local users mailboxes,
as well as allowing them to send e-mail to our and other domains.
Luckily for us,
this is almost the default configuration in OpenSMTPD,
the only differences being that the default configuration doesn't know about our domain and only accepts connections from the local machine.

The default configuration is this one:
```
#	$OpenBSD: smtpd.conf,v 1.11 2018/06/04 21:10:58 jmc Exp $

# This is the smtpd server system-wide configuration file.
# See smtpd.conf(5) for more information.
table aliases file:/etc/mail/aliases

# To accept external mail, replace with: listen on all
#
listen on lo0
action "local" mbox alias <aliases>
action "relay" relay

# Uncomment the following to accept external mail for domain "example.org"
#
# match from any for domain "example.org" action "local"
match for local action "local"
match for any action "relay"
```

now after reading the comments,
let's remove them and adapt the configuration to accept e-mail for "hypno.cat" from the world without aliases for the time being:

```
listen on all
    
action "local" mbox
action "relay" relay
    
match from any for domain "hypno.cat" action "local"
match for any action "relay"
```

The main difference being the change on the "listen" line so the server listens to connections on all interfaces not just local ones,
as well as the change of a "match" rule so the server matches envelopes coming from any source for the domain "hypno.cat".
With only these changes,
people can now send an e-mail to a user of the server by using a @hypno.cat e-mail address,
OpenSMTPD will deliver the e-mail to an mbox.
Local users may send mail to @hypno.cat which will also deliver locally,
or send mail to any other domain which will be relayed outside.

### Making sure everything works
Now that DNS and MX are properly set,
we need to check that our MX can actually exchange e-mails with the world.

Once started,
the first thing you will want to check is if a machine outside your network can reach your MX through 25/tcp.
Sending yourself a mail from another e-mail address is the quickest way to ensure things work right,
as if you receive the e-mail it means your MX is reachable.
You'll want to reply to that e-mail to make sure that the exchanges work both ways.

Now what if this doesn't work ?

You have many ways of troubleshooting the issue.
First, make sure that you don't see connections in your log because it could be a mistake,
you could very well have received the e-mail but be looking at the wrong place.
If you see nothing in logs,
you should try connecting to your MX on 25/tcp using telnet or netcat,
an OpenSMTPD banner similar to this should be displayed:

```
$ nc mx.hypno.cat 25
220 mx.hypno.cat ESMTP OpenSMTPD
^C
$
```

If you don't see a banner,
it means that your MX is not reachable from the outside.
Either you have a firewall preventing connections,
your ISP/Host is blocking trafic,
your configuration is incorrect or maybe you haven't restarted the daemon to use your updated configuration.

If you see the banner but you can't reach yourself from another e-mail address,
the issue is most certainly in DNS.
Either the DNS configuration is incorrect,
or maybe the DNS resolver on the sending end had something incorrect in cache,
maybe even your zone is not being used at all.
You will have to investigate in the DNS perimeter and this is where you'll need a DNS book because it's out of my scope ;-)