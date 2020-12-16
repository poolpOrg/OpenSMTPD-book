# Getting started

    The secret to getting ahead is getting started.
    -- Mark Twain


## Preliminaries
Unlike other software that can put you in a world of stress,
OpenSMTPD tries to bring a little bit of peace and harmony in your life as a postmaster.
It is straighforward and this whole adventure will be as pleasing as a journey at the park.

Oh,
and as far as preliminaries are concerned,
there are none and your mail server will be working in a few minutes.


## OpenSMTPD on OpenBSD
If you're an OpenBSD user then you're in luck:
OpenSMTPD became available for testing in the base system with OpenBSD 4.6 in 2009,
then replaced the venerable Sendmail as the default MTA with OpenBSD 5.8 in 2015.

On a brand new OpenBSD install without any former setup,
the daemon will start at boot time and only allow local connections,
delivery to local mailboxes and relaying from local users to remote recipients.

If you're running a machine that doesn't need to operate as a mail exchanger accepting mail from remote hosts,
then you're all good and I hope you enjoyed this book because this is were our roads split.

Otherwise,
you can just skip to the next chapter.


## OpenSMTPD on other systems
Throughout the years,
OpenSMTPD was ported to other systems including FreeBSD,
NetBSD,
DragonFlyBSD,
Linux,
macOS and Solaris.

The project provides a code base that can be built and that runs on these systems,
however there's no official package that can be installed with their respective package managers.
Package maintainers from these communities are encouraged to create them if they want to provide a clean integration to their system,
the OpenSMTPD developers often help them to ease packaging.

The best option is to first check if your system ships with OpenSMTPD in its base system.
If that's the case and it is the latest stable version, then...
you're all set and you can skip to the next chapter, chop chop.

Otherwise, you can check if your system has a pre-built package available for you to use.
This is the case on many systems were installing the latest stable OpenSMTPD is just a command away.
If that's the case for you, then again you're all set and you can skip to the next chapter.

Unfortunately,
several Linux distributions as well as macOS and Solaris do not have OpenSMTPD packaged.
The only remaining option is to build the project from sources and install manually,
the old school way.


## Building from source

### On OpenBSD
OpenBSD always ships with the latest stable OpenSMTPD in its base system,
there's absolutely no point in building from source unless you're a developer willing to contribute,
or if you're running an older OpenBSD and want to run the latest stable OpenSMTPD.

The process to build from source is very easy.
All you need is a fresh CVS checkout of the source tree somewhere on your system,
usually `/usr/src`,
then run the build from `usr.sbin/smtpd` (assuming you have proper permissions):

```sh
$ cd /usr/src/usr.sbin/smtpd
$ make
$ doas make install
```

This builds the code and installs the resulting executables.


### On other systems

If you need to build OpenSMTPD on systems other than OpenBSD,
the recommended method is to fetch the latest stable release from the official website.
A tarball is always linked from the front page. 

[https://www.OpenSMTPD.org](https//www.OpenSMTPD.org)



#### Building from the portable tarball

The portable tarball consists of the same code as the OpenBSD repository.
In addition it also contains a compatibility layer that allows the project to build on systems with behaviors differing from OpenBSD,
or that are missing some of its programming interfaces.

OpenSMTPD makes use of a minimal set of dependencies that are part of the OpenBSD system,
the portable version expects the same dependencies to be installed on target systems:

  - bison, to build the configuration parser
  - libevent, to abstract the asynchronous event loop.
  - libssl, preferably through the LibreSSL project, to provide TLS support.

Optionally,
the following dependencies will allow the building of conditional features:

  - libz, to allow mail queue compression
  - libdb, or any Berkeley DB compatible interface, to provide database files.


Once these dependencies are installed,
OpenSMTPD can be built and installed as follow:

```
$ tar -zxf opensmtpd-6.8.0p1.tar.gz
$ cd opensmtpd-6.8.0p1
$ ./configure --prefix=/usr/local
$ make
$ sudo make install
```

Note that depending on your system the dependencies may be installed in various places.
OpenSMTPD's configure script attempts some autodetection but this doesn't always succeed and you may need to pass additional options to specify paths:

```
$ ./configure --prefix=/usr/local --with-libevent=/opt --with-libssl=/opt
```

The portable tarball comes with a README.md file which documents expected configure options for some known systems.


## Creating the daemon users
To be able to run,
OpenSMTPD requires that you provide it with two dedicated system accounts that are used to improve the security of the daemon.
The first account allows the daemon to revoke root privileges in child processes so that no process manipulates user input with privileges.
The second one is used to run the queue process as a distinct unprivileged user,
preventing other processes from being able to alter or delete its content should they be compromised.

On OpenBSD,
the _\_smtpd_ and _\_smtpq_ unprivileged accounts are part of the base system and no additional work is required.
On systems that provide a package,
these users were most likely created automatically at install possibly with a different naming convention.
When installing by hand... they must be created by hand.

While no naming convention is imposed by OpenSMTPD and these accounts can be named freely,
the code expects these particular names unless overridden with configure flags at build time.
The accounts do not need a real home directory or shell,
their only purpose is to allow dropping privileges.

The commands below are what would be required to create them on an OpenBSD system,
you should refer to your operating system's documentation to adapt them accordingly:

```
# groupadd _smtpd
# groupadd _smtpq
# useradd -s /sbin/nologin -d /var/empty -g=_smtpd _smtpd
# useradd -s /sbin/nologin -d /var/empty -g=_smtpq _smtpq
```


## About the mailwrapper
Once upon a time, the only MTA software easily available was Sendmail.
As a result, most MUA back then were written with hardcoded pathname to the `sendmail` executable,
which was the one true way of enqueuing mail locally.

As new MTA solutions became available,
it was easier for them to adopt a sendmail-compatible interface rather than to expect MUA developers to make changes and support multiple MTA interfaces.
The sendmail interface became a de-factor standart and MUA would simply call the `sendmail` executable without worrying about what MTA was hiding behind it.

The fact that the `sendmail` executable had now become a de-facto interface and no longer a Sendmail specific executable led to ambiguities and issues when multiple MTA were installed on the same machine.
This is where the mailwrapper came into play.

My MUA expects /usr/sbin/sendmail to be used for sending mail.
But if we take a closer look, it is no longer an executable installed by my MTA but really a symbolic link to /usr/sbin/mailwrapper:

```
$ ls -l /usr/sbin/sendmail                 
lrwxr-xr-x  1 root  wheel  21 Mar  5 19:43 /usr/sbin/sendmail -> /usr/sbin/mailwrapper
$
```

When my MUA executes `/usr/sbin/sendmail`,
what it really does is that it executes `/usr/sbin/mailwrapper` which then opens its configuration file and finds the following:

```
$ grep ^sendmail /etc/mailer.conf
sendmail        /usr/sbin/smtpctl
$
```

When executed,
the mailwrapper looks at the name it was invoked with and searches inside the _/etc/mailer.conf_ what command is associated to that name.
This allows multiple MTA to be installed on the same system in different places,
not colliding one with another,
and giving the postmaster a way to configure which MTA is the active one answering to the sendmail-interface.

To ensure that OpenSMTPD responds correctly to the Sendmail interface used by most MUA,
the _mailer.conf_ configuration must be adapted to support the following commands:

```
$ cat /etc/mailer.conf
sendmail    /usr/sbin/smtpctl 
send-mail   /usr/sbin/smtpctl 
mailq       /usr/sbin/smtpctl 
makemap     /usr/sbin/smtpctl 
newaliases  /usr/sbin/smtpctl
$
```

Note that the path to smtpctl may be different depending on the prefix you used when you configured your build.
Also,
not all systems rely on the mailwrapper and some may use different mechanisms or none at all,
you may need to adapt this according to your operating system's documentation as I'm not familiar with all mechanisms out there.


## We're all set, let's give it a run !
With the daemon installed and user accounts created,
you're ready to start the daemon.

The manual way of starting it is by running the `smtpd` command as _root_.
Since you're obviously not logged in as _root_ because it is bad practice,
this is usually done with the `sudo` command (or `doas` on OpenBSD).

Throughout this book,
whenever I use the `doas` command in an example,
you can substitute it with `sudo` if that's what your system uses:

```
$ doas /usr/local/sbin/smtpd
$
```
  
Depending on your system,
startup scripts may be provided and you should refer to your operating system's documentation.
On OpenBSD,
the `rcctl` utility may be used to start OpenSMTPD as follows:

```
$ sudo rcctl start smtpd
smtpd(ok)
$
```

If no error message is printed to the console,
then OpenSMTPD is probably running in the background and ready to serve local requests:
```
$ ps ax |grep smtpd
85854 ??  Ssp     0:00.02 smtpd
52840 ??  Sp      0:00.10 smtpd: klondike (smtpd)
38508 ??  Sp      0:00.05 smtpd: control (smtpd)
67581 ??  Sp      0:00.17 smtpd: pony express (smtpd)
23615 ??  Sp      0:00.04 smtpd: lookup (smtpd)
  374 ??  Sp      0:00.07 smtpd: queue (smtpd)
36517 ??  Sp      0:00.03 smtpd: scheduler (smtpd)
$ tail -1 /var/log/maillog
Aug 18 20:26:57 ams-1 smtpd[2731]: info: OpenSMTPD 6.7.0 starting
$ nc localhost 25
220 localhost ESMTP OpenSMTPD
^C
$
```

If it isn't started,
the reason should be displayed on the console or available in your system's log file.
The most common causes of startup failure are a misplaced configuration file,
missing unprivileged accounts or even another MTA already registered and/or running on the system.

Once your done...
Congratulations, you can move to the next chapter !
