# What you need to know about the Internet Message Format

    Information is a source of learning. But unless it is organized, processed, and available to the right people in a format for decision making, it is a burden, not a benefit.
    -- William Pollard


## The Internet Message Format
The Internet Message Format,
IMF for short,
is the format used for exchanging messages between mail exchangers.
It is standardized in RFC 5322 and describes what and how things should be written in the ``DATA'' phase of an SMTP transaction:

```
C: DATA
S: 354 Enter mail, end with "." on a line by itself
C: Subject: proving a point
C: 
C: point made
C: .
S: 250 2.0.0: 832a2432 Message accepted for delivery
```

It is very important to understand that while the IMF is used by mail exchangers,
the real consumers are the MUA software which depend on the structure to properly display the message for the end-user.
As far as a mail exchanger is concerned,
an IMF could be badly crafted for the most part without having any impact whatsoever on the transfer of messages.
The use of the IMF is a convention for interoperability.


### The basic structure
In its most basic structure the IMF consists of two parts,
the headers part and the body part,
separated by an empty line:

```
some_header: foo
another_header: bar

this is a very nice body,
spanning on multiple lines,
including empty lines,
it is the first empty line that separates headers from body.
```

### The headers part
The headers are metadata that help make sense out of a message.
They contain fields which help the MUA display the message in the most readable way,
fields which help the sender provide details like a subject, name or date,
and fields which are added by mail exchangers to help postmasters troubleshoot transfer.

Each header consists of a header name and a header value,
separated by a colon and a space ``: '',
and may span on multiple lines as long as continuation lines are escaped by prepending with whitespaces:

```
header_name: header_value
long_header_name: value
  continues here
```

Restrictions exists with regard to the names of headers and their value length,
but also with regard to the structure of the headers part.
Some headers are required,
some are optional,
some may appear only once or multiple times,
while others are custom and only makes sense for some applications.

The details beyond all these requirements and restrictions are out of the scope of this book,
we only need to have very basic knowledge of the IMF,
so I'll cover the relevant details when we need them.


### The body part
The body part contains... the body of the message.
This is what the initial sender wanted the recipient to read, metadata set aside.

Restrictions also exist with regard to the length of lines and such,
again this is irrelevant to this book,
read the RFC if you're interested in these details.

In its most simple form,
the IMF will contain a header part with a few headers and a body part with a human-readable message:

```
Date: Wed, 16 Dec 2020 10:41:51 +0200
From: Gilles Chehade <gilles@poolp.org>
To: Eric Faurot <eric@faurot.net>
Subject: quick question for you

helo eric,
I was just wondering...
Any chance you could review my diff before I die of old age ?
```

In this example,
the body consists of simple ASCII message which will be displayed as is by the MUA.
The headers could also provide hints to display the message in a different format or encoding,
in HTML and UTF-8 for example,
leaving it up to the MUA to properly deal with that information:

```
Date: Wed, 16 Dec 2020 10:41:51 +0200
From: Gilles Chehade <gilles@poolp.org>
To: Eric Faurot <eric@faurot.net>
Subject: quick question for you
Content-Type: text/html; charset=utf-8

<html>
  <body>
    <h1>helo eric,</h1>
    <p>
      I was just wondering...
      Any chance you could review my diff before I die of old age ?
    </p>
  </body>
</html>
```
  


Note that the headers provide ``From'' and ``To'' headers.
It is important to understand that they do not come from the ``MAIL FROM'' and ``RCPT TO'' SMTP phases.
They are metadata written by the original sender to display in the MUA,
but serves no purpose in terms of transfer.

We have to distinguish the ``SMTP envelope'' from the ``DATA envelope''.
The former is what happens at the protocol level before the DATA is even received,
the latter is what will be displayed in the MUA no matter how the message was transfered.
The ``SMTP envelope'' contains only the e-mail addresses,
while the ``DATA envelope'' may contain the first and last name to read more user-friendly.
Both may be aligned,
in the sense that they have the same addresses between the SMTP and DATA envelopes,
but this is not a requirement and you will often receive mail where the SMTP sender differs from the DATA ``From'' header.


### Multi-part messages
Nowadays messages contain more than just a simple ASCII message.
Some messages contain attachements while others provide both an HTML and a plain text version of the same body,
leaving to the MUA the decision to display the most appropriate version.

We have only looked at a very simple IMF structure,
but thanks to the magic of Multipurpose Internet Mail Extensions,
also known as MIME,
it is possible to structure a message so that it embeds multiple messages or components in a single message.
With appropriate hints,
the MUA is then capable of splitting the body into multiple distinct parts and decide what to do with them.
For mail-exchangers,
only the top level structure is relevant and the notion of nested structures is of absolutely no concern.

We don't need to dive into this but I will still provide a simple example just so you grasp how they work,
then you can forget about multi-part for the remaining of this book:

```
Date: Wed, 16 Dec 2020 10:41:51 +0200
From: Gilles Chehade <gilles@poolp.org>
To: Eric Faurot <eric@faurot.net>
Subject: quick question for you
MIME-Version: 1.0
Content-Type: multipart/alternative;
  boundary="--==_mimepart_5b784aee4aca8_643f3fbdb76d45c04373fd";
  charset=utf-8

--==_mimepart_5b784aee4aca8_643f3fbdb76d45c04373fd
Content-Type: text/plain; charset=utf-8
    
helo eric,
I was just wondering...
Any chance you could review my diff before I die of old age ?
--==_mimepart_5b784aee4aca8_643f3fbdb76d45c04373fd
Content-Type: text/html; charset=utf-8

<html>
  <body>
    <h1>helo eric,</h1>
    <p>
      I was just wondering...
      Any chance you could review my diff before I die of old age ?
    </p>
  </body>
</html>  
--==_mimepart_5b784aee4aca8_643f3fbdb76d45c04373fd--
```

Basically,
the top structure declares that the content type of the body is a multipart message and provides a boundary delimiter between these parts.
The body then contains multiple sections separated by the delimiter,
each one being a new IMF structure with their own headers and body,
the headers declaring the content type of the body.

In this example,
the IMF contains an HTML and a plain text version of the same message,
and the content type of the top structure hints the MUA about which one to pick first.
When you're in a deep rage because of a mail displaying as HTML in your console client,
now you know that either it wasn't multipart and didn't provide a plain text version,
or the Content-Type header was not providing the proper hint for your MUA to do its work correctly.

A multipart message may contain many parts with different goals.
Here the two parts were intended to provide different alternative for an end-user to read the message,
but other parts may serve other purposes like adding attachements to a message or even encoding images that can be referenced in the HTML version of a message.
How parts are consumed is a matter of what headers are present in the parts and the capabilities of the MUA.
