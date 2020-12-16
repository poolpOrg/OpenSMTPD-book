# What you need to know about SMTP

    Risk comes from not knowing what you're doing.
    -- Warren Buffett

    We are all born ignorant, but one must work hard to remain stupid.
    -- Benjamin Franklin


## What is SMTP ?
SMTP is the Simple Mail Transfer Protocol.
It is a protocol standardized in RFC5321 and used by mail systems to communicate one with another.
The SMTP protocol does not handle the entire mail chain but has a very specific objective:
transfer mail reliably from point A to point B.
It is the electronic analogy of a mailman,
more reliable though...

## The SMTP protocol
SMTP is a plaintext protocol that relies on a line-based human-readable dialogue between a client and a server.
A human,
and that's including me,
can issue a connection to an SMTP server and play out an SMTP session manually:

```
$ nc localhost 25 
S: 220 poolp.org ESMTP OpenSMTPD
C: HELO localhost
S: 250 poolp.org Hello localhost [127.0.0.1], pleased to meet you
C: MAIL FROM:<gilles>
S: 250 2.0.0: Ok
C: RCPT TO:<gilles>
S: 250 2.1.5 Destination address valid: Recipient ok
C: DATA
S: 354 Enter mail, end with "." on a line by itself
C: Subject: proving a point
C: 
C: point made
C: .
S: 250 2.0.0: 832a2432 Message accepted for delivery
C: QUIT
S: 221 2.0.0: Bye
```

Note that this doesn't mean that a transfer is necessarily readable in the wire.
The plaintext dialogue may very well take place within an encrypted transport layer, such as TLS, making it look gibberish on the outside.
However,
no matter how it looks on the outside,
it is beautiful on the inside.


### The SMTP session
The SMTP session begins when an SMTP client connects to an SMTP server and ends when the client disconnects or is disconnected.
A session can be described as "everything that happened during a single connection".

An SMTP session always starts with an introduction phase between the server and the client:

```
S: 220 poolp.org ESMTP OpenSMTPD
C: HELO localhost
S: 250 poolp.org Hello localhost [127.0.0.1], pleased to meet you
```

Depending on how the client introduced itself,
with the verb HELO or the verb EHLO, an extended HELO,
the server may advertise informations regarding its features, extensions it supports and its limitations.

```
S: 220 poolp.org ESMTP OpenSMTPD
C: HELO localhost
S: 250 poolp.org Hello localhost [127.0.0.1], pleased to meet you

S: 220 poolp.org ESMTP OpenSMTPD
C: EHLO localhost
S: 250-poolp.org Hello localhost [127.0.0.1], pleased to meet you
S: 250-8BITMIME
S: 250-ENHANCEDSTATUSCODES
S: 250-SIZE 36700160
S: 250-DSN
S: 250 HELP
```

At this point,
the client may begin a transaction to transfer a message:

```
C: MAIL FROM:<gilles>
S: 250 2.0.0: Ok
C: RCPT TO:<gilles>
S: 250 2.1.5 Destination address valid: Recipient ok
C: DATA
S: 354 Enter mail, end with "." on a line by itself
C: Subject: proving a point
C: 
C: point made
C: .
S: 250 2.0.0: 832a2432 Message accepted for delivery
```

The session ends either due to a disconnect from either side or with an explicit request to QUIT by the client:
```
C: QUIT
S: 221 2.0.0: Bye
```

Congratulations,
you now know how to speak SMTP and play sessions by hand.
Read the next subsection to become a guru who understands the true essence of SMTP: transactions.


### The SMTP transaction
During an SMTP session, a client can send multiple messages.
Each message is handled within its own transaction,
containing a single sender, a single message and one or many recipients.

The transaction begins when client submits a new sender with the MAIL FROM command:

```
C: MAIL FROM:<gilles>
S: 250 2.0.0: Ok
```

Client then submits one or many recipients with the RCPT TO command:

```
C: RCPT TO:<gilles>
S: 250 2.1.5 Destination address valid: Recipient ok
```

Finally, client submits message with the DATA command and ends it with a single dot, '.', on a line by itself.

```
C: DATA
S: 354 Enter mail, end with "." on a line by itself
C: Subject: proving a point
C: 
C: point made
C: .
S: 250 2.0.0: 832a2432 Message accepted for delivery
```

The single dot on a line by itself is a request to commit the transaction.
The client basically asks for the server to acknowledge that it will become responsible for that message being delivered to all recipients.
As soon as the server answers positively,
the client assumes that the message will not silently vanish.
The server will attempt to deliver the message to all recipients that were accepted in the transaction AND notify the sender if any error occurs.


### The SMTP network
The SMTP protocol operates on a network consisting of nodes commonly known as Mail eXchangers,
or MX for short,
that all speak SMTP.
The goal of these nodes is to bring a message one-step closer towards destination,
usually by looking up in the DNS what is the next MX in charge of destination.
In the network,
nodes operate either as a relay which will accept a message and send it to another node,
or as a destination which will accept a message and deliver it to the recipient.
From a node perspective,
the next node is always the destination,
otherwise it would be better to skip it and talk to the destination directly.
Nodes require an envelope consisting of a sender and a recipient,
and possibly some other details,
to determine if they should operate as a relay or a destination.
In some cases,
a node may be a relay for some domains and a destination for others.

The concept of relay and destination will be met again frequently in this book.
For now,
what matters is to understand that from a user point of view the SMTP protocol is only about sending mail,
not retrieving it in any way.
The retrieving has to be handled by direct access to the mailbox,
or by the use of a different protocol such as POP or IMAP.

The role of the SMTP server is particularly critical.
If the server is down,
the domains that it handles no longer receive mail.
Users assume their mail systems to be reliable and they usually do not tolerate loss.
Since downtimes are unavoidable, the SMTP protocol provides mechanisms and imposes various constraints to mitigate their impact.
