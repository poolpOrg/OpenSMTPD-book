# Documentation, resources and community

    Lack of documentation is becoming a problem for acceptance.
    -- Wietse Venema


## Documents any OpenSMTPD user should know about
Now that you have the daemon running,
maybe it's the right time to discuss the various resources that are available for you to get familiar with the project.

To become a true guru, you should try to keep around the following documents as references.
As printing hundreds of pages is the modern equivalent of kicking nature in the groin,
it is preferable that you memorize them.
The RFC is only 90-ish pages long.


### The Holy RFC
OpenSMTPD implements RFC 5321, the _Simple<sup>[1](#1)</sup> Mail Transfer Protocol_.
Any question regarding the expected behavior and interactions with other MTA are answered in that document.
These are 95 pages of pure joy<sup>[2](#2)</sup>.

A major goal of the project is to comply with the RFC as much as possible in order to provide a sane SMTP implementation.
Because the RFC is not the real-world,
following it too strictly may result in interoperability issues with other implementations.
OpenSMTPD takes a more pragmatic approach:
the RFC is a Holy document and, as every other Holy document,
it must be sometimes interpreted or ignored.


### The man pages
As is the habit with OpenBSD projects,
the man pages are just wonderful.
They are up-to-date,
they document about everything while providing descriptions and examples of use.
Mistakes and undocumented features are considered as bugs and are fixed as soon as possible to keep the documentation as great as possible.

When about to ask support to developers,
always make sure that you have searched the man pages for your answer.
Most of the time,
developers simply reply with a copy of a paragraph answering the questions and coming directly from the man pages.

In a parallel universe where this book doesn't exist,
you should be able to setup your mail exchanger using only the man pages.
The man page for the configuration file ends with complete examples that could be copy-pasted and used to get a daemon started.
Luckily for you, in this universe this book exists and you should most definitely buy multiple copies to keep as reference.


## Sites and resources
As an OpenSMTPD user,
you will also want to know about the online spots where the cool kids hang out.
These are useful if you want to find informations, help or news about the project,
or if you just want to spend some quality time on the Internet.


### The official website
Home, sweet home.
This is where you go to fetch a release tarball,
the address of our mailing list,
details regarding the goals of the project and a short blurb about its history.

[https://www.OpenSMTPD.org](https://www.OpenSMTPD.org)


### The Undeadly news site
Undeadly.org is an OpenBSD-specific news site.
It covers OpenSMTPD stories when developers attends hackathons and describe their work there,
or when there are major changes going into OpenSMTPD.
Even if you are not an OpenBSD user and only care about OpenSMTPD stories,
it is a valuable source of information to understand the design and programming interface changes taking place in the project due to research done by OpenBSD hackers.

[https://www.undeadly.org](https://www.undeadly.org)


### poolp.org
This is where the project started before joining the OpenBSD universe and where I continue to experiment and test bleeding-edge code on a small community of guinea pigs.

The website contains a blog where,
among other unrelated things,
I write technical posts about various aspects of the project ranging from code explanation to rationale behind major refactoring or works in progress.

[https://poolp.org](https://poolp.org)


### The OpenSMTPD mailing list
misc@opensmtpd.org is a general purpose low-volume list that can be used to ask questions about configuration,
discuss about the project or related topics,
submit diffs or share interesting links with a community of hundreds of users running OpenSMTPD on various systems.

[https://www.opensmtpd.org/list.html](https://www.opensmtpd.org/list.html)


### Twitter
The project is also active on twitter where I maintain the @OpenSMTPD account.
Tweets that are related to the projects are posted with the #OpenSMTPD hashtag.

[https://twitter.com/OpenSMTPD](https://twitter.com/OpenSMTPD)


### IRC
Last but not least,
developers and users hang around the #OpenSMTPD channel on the Freenode network.
The channel is moderately active with around a hundred members at the time of this writing,
holding technical discussions and assisting people with the problem they meet using OpenSMTPD.


## This book
OH.
I ALMOST FORGOT.
This book is going to be your new reference.
It will be updated and should become the new OpenSMTPD Holy book as far as I'm concerned.

<hr />

[<sup>1</sup>](#1) OOOOH, THE IRONY.

[<sup>2</sup>](#2) No wait. OOOOH, THE SARCASM.
