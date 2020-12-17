# The roles of OpenSMTPD

    Cat: Where are you going?
    Alice: Which way should I go?
    Cat: That depends on where you are going.
    Alice: I don’t know.
    Cat: Then it doesn’t matter which way you go.
    -- Lewis Carroll, Alice in Wonderland


## Network server and client
First of all, OpenSMTPD is a network server and client.
It operates as a server when accepting connections from remote clients and mail exchangers,
and it operates as a client when connecting to remote mail exchangers.
When a mail exchangers operates as a backup MX,
for example,
a mail will enter the system through the server part of the MTA,
then be relayed to the primary MX through the client part of the MTA.

In its "secure by default" configuration,
OpenSMTPD does not accept to receive messages from the network,
limiting its server capabilities,
but accepts sending to the network,
making use of its client capabilities.
Either way,
incoming or outgoing,
it supports IPv4,
IPv6,
and encapsulating SMTP sessions in TLS either by using the SMTPS protocol,
or by using the STARTTLS extension to the SMTP protocol.


## Local enqueuer
In addition to these network capabilities,
OpenSMTPD provides a local enqueuing mechanism.
That mechanism relies on the use of the _sendmail_ interface,
which all MTA adhere to,
to inject messages in the daemon through a Unix socket.
It is technically possible to setup OpenSMTPD as a local MTA only allowing mail to be exchanged between local users,
and not making use of any networking capabilities.

Unlike with the network server which requires the daemon to be running to accept connections,
the local enqueuing capability allows local users to submit messages even if the daemon is not running.
Messages submitted while the daemon is down are written to an "offline" queue which is processed upon restart.
This ensures local applications and users don't have to deal with errors due to the MTA being down due to maintainance.
As far as they know,
the mail was accepted even though recipients may tell them that it is taking time to be delivered.

For a local user,
there are no major differences between using the local enqueuer or the network submission to localhost.
Both methods rely on the same code,
no performances benefits are expected from using one over the other,
however if your MUA supports local enqueuing,
and most do,
this is the sane thing to do really.


## Decision engine
Whatever the method used to inject a mail for processing,
a major role for the MTA it is to determine if it knows what to do with the mail and if it accepts to do it.
Many MTA use configuration options to take the decision,
OpenSMTPD relies on a ruleset which is processed sequentially.

No matter how this is handled,
the idea remains the same:
given an envelope coming from a particular sender and heading to a particular recipient,
is it ok for this MTA to accept taking care of the next step for the envelope ?

The decision engine is a critical part of the MTA as misconfigurations result in catastrophic situations,
as you will lose mail if legitimate envelopes are rejected,
and you will open yourself to abuse if illegitimate envelopes are accepted.
Spammers actively scan mail exchangers in order to find misconfigured ones that will accept to send mail to others on their behalf.

The decision can only result in two situations:
an envelope is accepted because the MTA knows what to do with it or it is refused.
When accepted,
it means that the MTA is operating as the destination for that envelope and will deliver the message to a local user,
or it is operating as a relay to another mail exchanger and will forward it there.

Either way,
if an envelope is accepted and the SMTP transaction isn't aborted,
a copy of the envelope and message are written to the queue on disk and waiting for delivery to take place.


## Queueing
The SMTP protocol mandates that mail exchangers do whatever they have to do to ensure that envelopes and messages they are responsible for won't vanish.
This means that if someone tripped on the power cable of a mail exchanger right after it had accepted to handle a transaction,
it must be able to process the transaction upon restart as if nothing had happened.
It also implies that unless envelopes and messages are written to a persistent storage right before accepting the transaction,
usually the disk,
then mails may be lost by the mail exchanger.

Not only the envelopes and messages must be written to the disk,
but they must be written in a way that shutting the mail exchanger down at any time during the writing doesn't cause a corrupted state.
In addition,
disk accesses are not cheap so the trickier the technique to ensure this sane state,
the bigger the hit on performances for each and every transaction.
If you keep in mind that all of this is to ensure that your mail exchanger,
with an uptime of hundreds of days,
doesn't lose a mail in the unlikely case that it would crash right at the wrong time,
then you can really grasp what it means to take responsability of a message.

The queue within OpenSMTPD holds all mails pending delivery,
it provides an interface to read and write to the persistent storage and nothing more than that.
A scheduler is in charge of determining for which mails within the queue a delivery should be attempted at a given time.
Mails exit the queue when postmaster explicitely requests deletion,
when the mail has been sucessfully delivered, permanently rejected or simply when it expires.

OpenSMTPD provides a very reliable queue which has persistence guarantees as a top priority while providing the best performances under that constraint.
The details of how this is done are only relevant to curious developers and will be explained in a later chapter.


## Scheduling
Storing messages in a passive queue would not be very interesting if OpenSMTPD did not actually try to do something with them.
And this is where the scheduler comes into play.

It is a simple component which will scan the queue once at startup and create a very light memory view of all known envelopes.
That memory view is structured in a way that lets the scheduler efficiently find envelopes based on their transaction identifier,
but also to know at a given time which ones have reached expiry and which ones should be scheduled next.
During the lifetime of the daemon,
whenever a new envelope is inserted or removed from the queue,
the scheduler will update its state in order to always have an exact representation of what envelopes exist.

Since the scheduler always knows of all the envelopes it has to manage,
it is fully idle when there are no envelopes,
and it can always sleep and wake up right in time for the next envelope to schedule.
This makes it a very light process in terms of resources consumption.

When envelopes are schedulable,
it notifies the proper subsystem so an attempt is made to deliver to a local user or to relay to a remote mail exchanger.
If an envelope reaches expiry without a successful delivery,
it notifies the queue so the envelope is removed and updates its internal state to reflect that.


## Mail Delivery and Mail Transfer Agents
A mail that is sitting in the queue was either accepted for local delivery or for relying,
so when the scheduler makes it schedulable the mail must be passed to the proper subsystem.

Local deliveries happen through mail delivery agents.
These are programs which follow a convention of reading content from their standard input,
delivering them wherever they should,
and reporting success with their exit status.
OpenSMTPD ships with three delivery methods by default:
mbox which delivers to an mbox-format mailbox,
maildir which delivers to a maildir-format mailbox
and lmtp which connects to an LMTP server and forwards the mail.

Relayed deliveries happen through the mail transfer agent which manages clients connecting to other mail exchangers.
The agent will do MX lookups, establish connections, play an SMTP session and try to submit a transaction for a mail.

Either way,
the status of the delivery is notified to the queue and scheduler so they can properly update their state.


## Putting it all together
With all of the above,
we can provide a relatively precise description of what OpenSMTPD does.

Its role is to accept a mail from a local enqueuer or through a network submission,
ensure that it knows and wants to do something with it,
write it to a safe storage to make sure it doesn't lose it,
determine when and how are the next appropriate time and methods to attempt delivering,
request the proper subsystem to make an attempt at delivering locally or remotely,
and finally update its state based on the success or failure of this attempt.

Of course each and every task has its details and could be explained for pages,
but this description is enough to know we're on the same page as to what to expect for the time being.

