# Preface

I considered myself an experienced sysadmin when,
in 2007,
a user of mine asked if I could make a trivial change to his mail configuration.
It was nothing new to me.
I had been running mail services for about ten years,
hosting thousands of users over the course of time,
and been contracted multiple times to setup mail infrastructures for others.
I was familiar with Sendmail and Postfix,
mail stuff was easy work that I could do while sipping a cup of coffee pondering about the meaning of life.

That user...
I don't recall precisely his request,
I'm still in denial,
but it was of the kind that I would accept and deal with right away.

So I started editing Sendmail's "configuration file" and I could not convince myself that what I had done was correct.
The more I read the changes, the more they confused me.
I had the intuition, a subconscious warning, that something was going to break for some weird reason.
After a while, I decided to do what any sysadmin would do in that situation: take a leap of faith[<sup>1</sup>](#1).

I deployed, broke mail and reverted back to previous configuration.
I was devastated,
ridiculed by the SMTP Daemon,
my old trusted friend had just slapped me in the face and I could see him laughing.
At me, not with me.

To change my mind,
I called a friend for a "beer and hacking session" at a pub nearby and wrote the first lines of a _simple_ SMTP server.
A few hours later,
slightly intoxicated,
I had the _poolp.org_ SMTP server delivering its first mails to my mailbox,
violating every single RFC requirement out there.

Years later and with the contribution of many people,
OpenSMTPD has evolved a lot from the initial daemon I showed pyr@, chl@, henning@ and reyk@.
It is dead simple,
provides a wide range of features,
has very clean code and can be extended to achieve impressive setups.

I have no doubts someday it will rule the world.

-- Gilles Chehade

<hr />

[<sup>1</sup>](#1) Don't do that.
