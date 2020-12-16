# Introduction

    No, this is not the beginning of a new chapter in my life;
    this is the beginning of a new book!
    That first book is already closed, ended, and tossed into the seas;
    this new book is newly opened, has just begun!
    Look, it is the first page!
    And it is a beautiful one!
    -- C. JoyBell C.


## Once upon a time...
In the early 70s,
Ray Tomlinson exchanged the first message between two machines distant by just a few meters.
He unknowingly invented what would later become one of the most popular component of today's Internet:
<strike>spam</strike> e-mail.

Very fast-forward...

The SMTP protocol,
used to transfer mail from a mail exchanger to another,
was standardized in the late 70s.
It started gaining traction during the 80s as more and more machines became permanently connected,
progressively deprecating UUCP as the protocol of choice for exchanging e-mail.
In the 90s,
with internet service providers and mail hosting companies handing addresses to everyone with an internet access,
the SMTP protocol became so widespread that it became the media of choice to exchange messages worldwide.
Not everyone shared the same instant messenger but everyone with an internet access had an e-mail address that could be reached by anyone else.

Estimates claim that today +350 billions emails are exchanged around the world daily between +3.1 billions mailboxes owned by +2 billion people<sup>[1](#1)</sup>.
That's roughly one person out of three for the entire population of our planet (Earth at the time of this writing).
Not only e-mail addresses are used as identifiers to log into various websites and are often mandatory for administrative tasks in many countries,
but you can also ask anyone at random for an e-mail address and they will provide at least one:
this is how popular SMTP has become.

E-mail is simple and inexpensive to use,
it uniquely identifies a recipient,
it is free to obtain so users may have as many as needed,
it is fast to deliver,
it allows efficiently sending the same message to many people at once,
it allows attaching all kinds of files to a message,
and it does not require sender and recipients to be in front of their computers at the same time.
If we recall that this came to the public at a time where permanent connection for end-users was a luxury and were the alternative was to send a letter through the post office and wait for days or weeks,
no wonder it became so widely used.

Today, with affordable dedicated servers and cheap permanent internet access,
many individuals and companies prefer to operate their own mail services rather than trusting a third-party private company with their messages.
The popularity of open-source systems like Linux and *BSD makes it possible to setup mail exchangers,
ranging from a simple laptop-powered relay to the most complex configurations handling billions of messages for thousands of users...
for virtually no cost but some machine(s), spare time and a good reference book such as the one you're reading right now ;-)

SMTP has shortcomings in the modern world.
Many things could be changed to improve security,
efficiency,
performances or even resistance to spam.
A new protocol could be devised to improve drastically the state of mailing but... don't hold your breath.
SMTP works just the right amount of fine and is so widespread that it became too big to fail.
The burst of use by social networks using mail to identify users and sending them notifications made it even stronger than it was,
and attempts to deprecate it have never succeeded anything beyond entering the history of failed attempts<sup>[2](#2)</sup>.

Luckily for us,
SMTP allows extending the protocol through extensions so we are not stuck in the 70s.
Sure it's not as sexy as a brand new protocol,
but it allows fixing the shortcomings through an evolutive process rather than by a complete revolution,
trashing something that works in the way.

When I first wrote this chapter,
many years ago,
I concluded this section by thanking Ray Tomlinson for all the spam he cursed us with when he submitted that first message between two machines.
Sadly, he passed away in 2016 so I will give him the tribute he deserves and thank him for his huge contribution to the world.
As a developer working with people in different timezones and as an individual having friends and family spread in different countries,
his initial work and how it later evolved in this huge communication media had a huge impact in my life. RIP.


## What is OpenSMTPD ?
It is a FREE implementation of the SMTP protocol,
as defined by RFC 5321,
with the addition of some commonly used extensions.
It is a software which allows ordinary machines to be turned into mail exchangers capable of communicating with clients and other mail exchangers,
using the SMTP protocol,
in order to route mails to their destination.

The software is distributed for free under the ISC license.
It imposes no restrictions for both private and commercial uses.
The license grants permission to use,
copy,
modify and distribute the software for any purpose,
with or without fee,
as long as the copyright and the permission notices appear in all copies.
The code is offered to the community,
you are allowed to sell it and/or make a derivative work without giving back anything,
all you have to remember is that karma is a bitch.

The ISC license is reproduced below:

    Permission to use, copy, modify, and distribute this software for any
    purpose with or without fee is hereby granted, provided that the above
    copyright notice and this permission notice appear in all copies.
    
    THE SOFTWARE IS PROVIDED "AS IS" AND THE AUTHOR DISCLAIMS ALL WARRANTIES
    WITH REGARD TO THIS SOFTWARE INCLUDING ALL IMPLIED WARRANTIES OF
    MERCHANTABILITY AND FITNESS. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR
    ANY SPECIAL, DIRECT, INDIRECT, OR CONSEQUENTIAL DAMAGES OR ANY DAMAGES
    WHATSOEVER RESULTING FROM LOSS OF USE, DATA OR PROFITS, WHETHER IN AN
    ACTION OF CONTRACT, NEGLIGENCE OR OTHER TORTIOUS ACTION, ARISING OUT OF
    OR IN CONNECTION WITH THE USE OR PERFORMANCE OF THIS SOFTWARE.


## Why is OpenSMTPD so simple ?
It is heavily influenced by the development mindset shared between OpenBSD hackers and follows a few rules from that universe:
- be sane by default, do not have surprising default behaviors
- keep the configuration easy and simple
- fight and remove useless knobs and buttons: users are going to push them
- make things "just work"

It provides sane defaults by ensuring that any behaviour not explicitely set by users will be safe and will respect the principle of least surprise.
As a result,
the most common setup is expressed with the smallest configuration file,
and one must very explicitely request to shoot own foot as the software will not help to do that by itself.

The configuration file is probably one of the simplest out there.
Lots of efforts have been poured into making it as readable as possible<sup>[3](#3)</sup>.
Most setups can be expressed in less than 10 lines of human-readable rules,
and the grammar for these rules describe the intention so reading them aloud is often enough to spot obvious configuration bugs.

No unnecessary knobs are provided because if the default behaviour should be tweaked,
then maybe it's not the best default behaviour after all and it must be re-assessed by developers.
Of course sometimes this isn't true and buttons must be provided,
but every time a new button is requested by a user,
an investigation takes place to understand the real need behind that button,
and if it can be solved transparently for everyone without adding a new keyword in the grammar.
For some buttons, the value is so optimal that any attempt to tweak them outside of very specific use-cases,
by people who know exactly what they are doing,
will actually degrade the performances of the software.

Finally, it just works.
The default setup works right away,
without any surprise,
and does exactly what you can expect from a default setup.
To be honest, it's almost boring.


## Is OpenSMTPD good for me ?
The only way to know is for you to actually try and figure by yourself if it has the features you need.
The focus on keeping it simple and its relatively young age makes it less featureful than alternatives,
however it does cover a very wide range of common use-cases and a majority of people will find it fulfills their needs.

In addition,
the configuration file is considerably simpler than ones found in alternatives.
People setting up their first mail exchangers will find it much easier with OpenSMTPD and will suffer excruciating pain dealing with anything else later on.
This is how we trap people.

As far as reliability is concerned,
it's been in use for years and has gone through very intensive stress-tests of all code paths.
It's been used in production to deal with hundreds of concurrent sessions,
hundreds of outgoing mails per-second and even multi-million messages queues in some situations.
It is rock-solid,
the performances are sensibly identical to other SMTP servers on the incoming path and very efficient on the outgoing path.

It even went through an audit which resulted in reliability and security fixes,
as well as refactors of some areas to make similar classes of attacks a thing of the past.
There's virtually no reason NOT to try it unless you know for sure it lacks a feature you need.


## Some terminology
Besides the light tone,
this is still a technical book and you will be encountering some terms and acronyms here and there.
Unless you know what they mean,
you are going to have a difficult time understanding things,
not to mention discussing with other postmasters.

The good news is that I don't like using cryptic terms and acronyms,
my use of the terminology is not too dense and you will be able to work around with just a few of them.

- postmaster: you, the mail system administrator.
- users: them, the people who receive mail.
- MX: or Mail eXchanger, a machine running an SMTP server and exchanging mails.
- MUA: or Mail User Agent, a program used to submit and read mail (i.e: thunderbird, mutt, ...).
- MDA: or Mail Delivery Agent, a program in charge of delivering mail to a user mailbox.
- MTA: or Mail Transfer Agent, a program in charge of transfering mail from an MX to another, often used to refer to the SMTP server as a whole.

This is most definitely not an exhaustive list and I could throw many many other acronyms to make this book look smarter.
This should be enough to get you somewhere with the next chapters though.


<hr />

[<sup>1</sup>](#1) Source: statistics found on the internet, which as we all know, never lies.

[<sup>2</sup>](#2) A minute of silence for Google Wave, please.

[<sup>3</sup>](#3) A Ph.D in Compilers Theory is no longer a prerequisite to running a mail server.