Building Postfix on RHEL / CentOS from Source 74
This entry was posted in LinuxReferenceTechnology and tagged binariesbuildCentOSCentOS 5CentOS 5.5CentOS 5.xCentOS 6compilePostfixPostfix 2.3.3Postfix 2.6.6Postfix 2.8Postfix 2.9RedhatRHELrpmsource on January 17, 2011 by Steve Jenkins (updated 2184 days ago)
UPDATE 7/11/15: The following procedure will work on RHEL/CentOS 7, 6, and 5 systems to upgrading from those systems’ default versions to Postfix 3.02.

CentOS (a non-commercial clone of RedHat’s RHEL) is my server operating system of choice because it’s extremely stable and widely-used. One of the reasons it’s so stable is because the built-in software packages (and the software updates from the standard repositories) are generally (although there are always exceptions) older, more widely tested, patched, and secure versions of the most popular server daemons, such as the Apache web server, MySQL database server, and Postfix mail server.

This stability, however, comes with a price: if you want features from updated versions of a popular daemon, such as Postfix, you can’t just type “yum update postfix” to get the latest version.

As of this blog post, the version of Postfix that installs via yum on CentOS 7, 6, and 5 systems (using the standard repositories) is lower than the latest stable version. But there have been a lot of great improvements to Postfix since the earlier releases, and the now-current version is Postfix 3.02. Or perhaps you want to compile your own version of Postfix to enable MariaDB / MySQL support.  To take advantage of all the features from the latest version of Postfix, you can easily build and install your own binaries from the Postfix source code and install the latest Postfix on RHEL / CentOS.

This tutorial assumes:

You are running RHEL or CentOS 7, 6, or 5 (I have also verified this exact procedure also works on Fedora since version 12)
You have a working Postfix install (most likely Postfix 2.3.3 on RHEL/CentOS 5, Postfix 2.6.6 on RHEL/CentOS 6, or Postfix 2.10.1 on RHEL / CentOS 7)
You are using bash shell
If you have a different environment than noted above, then you’ll have to make whatever adjustments are necessary to make these instructions apply to you.

As always, backup first, follow these instructions at your own risk, do your best to understand what I’m recommending (rather than just following the steps blindly), and YMMV (your mileage may vary). 🙂

CentOS 5 Only: Check the Previous Build Options
My overall goal when building Postfix from source was to build it with most of the same options as the 2.3.3 version that is installed by default on CentOS 5. So my first step was to find out what options were used when the default version was compiled by looking in /etc/postfix/makedefs.out. Mine looked like this:

# Do not edit -- this file documents how Postfix was built for your machine.
SYSTYPE = LINUX2
AR      = ar
ARFL    = rv
RANLIB  = ranlib
SYSLIBS = -L/usr/lib -lldap -llber -lpcre -L/usr/lib/sasl2 -lsasl2 -L/usr/kerberos/lib -lssl -lcrypto -ldl -lz  -pie -Wl,-z,relro -ldb -lnsl -lresolv
CC      = gcc $(WARN) -fPIC -DHAS_LDAP -DLDAP_DEPRECATED=1 -DHAS_PCRE -I/usr/include/pcre -DUSE_SASL_AUTH -DUSE_CYRUS_SASL -I/usr/include/sasl -DUSE_TLS -I/usr/kerberos/include
OPT     = -O
DEBUG   = -g
AWK     = awk
STRCASE =
EXPORT  = AUXLIBS=' -L/usr/lib -lldap -llber -lpcre -L/usr/lib/sasl2 -lsasl2 -L/usr/kerberos/lib -lssl -lcrypto -ldl -lz  -pie -Wl,-z,relro' CCARGS='-fPIC -DHAS_LDAP -DLDAP_DEPRECATED=1 -DHAS_PCRE -I/usr/include/pcre -DUSE_SASL_AUTH -DUSE_CYRUS_SASL -I/usr/include/sasl -DUSE_TLS -I/usr/kerberos/include ' OPT='-O' DEBUG='-g'
WARN    = -Wall -Wno-comment -Wformat -Wimplicit -Wmissing-prototypes \
        -Wparentheses -Wstrict-prototypes -Wswitch -Wuninitialized \
        -Wunused
Where we want to pay attention is the EXPORT argument on line 12. This contains two important sections: AUXLIBS (the auxiliary libraries that were used when this version of Postfix was compiled) and CCARGS (the optional arguments passed to the C compiler).

The above output shows that the default version of Postfix on CentOS includes support for LDAP, PCRE, SASL, and TLS. It also used the -fPIC flag when compiled, which (according to Google but which I don’t totally understand) enables “position independent code.” Google also informed me that the -pie -Wl,z,relro options in CCARGS are used to “harden” compiled source code to make it less vulnerable to attack. Again, as a non-programmer, I don’t completely understand how this works, but I welcome further explanation from anyone in the comments!

Download and Unpack the Source Code
The source code for Postfix (including beta and past versions) can be downloaded from the Postfix Download Page. Choose a location close to you, then pick the release you want. For this example, I’m using the Postfix 3.0.2 stable release, which was released on July 19, 2015.

Download the source code to your server and save it in the /usr/local/src directory. Unpack the file (which will make its own subdirectory) with:

tar zxvf postfix-3.0.2.tar.gz
Root vs. User
Technically, the creator of Postfix recommends that Postfix be compiled as an unprivileged (regular) Linux user then installed as root. However, I have to admit that I did all these steps as root, which is more dangerous, but it also meant I didn’t have to deal with directory permissions issues. Again, the official recommendation is to only install as root, so proceed at your own risk if you do it otherwise.

Install the Berkeley DB Development Files
Before compiling Postfix, you’ll need to install the Berkeley DB development files, which you can do with:

yum install db4-devel
Choose Your Install Options
Depending on your specific needs, you can include more or fewer options when you compile Postfix yourself. Check section 4.3 of the Postfix Install readme for more information about which options are available and how to select them.

Again, my goal was to install Postfix with essentially the same options as the pre-compiled 2.3.3 that installs via yum, so these steps will reflect that.

Use the cd command to go into the directory where you unpacked the downloaded Postfix tarfile:

cd postfix-3.0.2
In that directory, you’ll see a file called Makefile. This file includes all the options the the compiler will evaluate when you compile the new binaries. Don’t edit this file directly. Instead, you should use the make makefiles command to rebuild the Makefile with the options you want.

I recommend using a text editor to write a short script that includes all your desired make options first, then running that script to execute the make makefiles command.

First, find out whether you have a 32-bit or a 64-bit OS with:

uname -i
Then use your favorite text editor to copy and paste the appropriate example script below into a file called make_postfix.sh (put it in the same directory as all the Postfix source files).

For a 32-bit (i386 or i686) OS:

#/bin/sh

make makefiles CCARGS='-fPIC -DUSE_TLS -DUSE_SSL -DUSE_SASL_AUTH -DUSE_CYRUS_SASL -DPREFIX=\\"/usr\\" -DHAS_LDAP -DLDAP_DEPRECATED=1  -DHAS_PCRE -I/usr/include/openssl -I/usr/include/sasl  -I/usr/include' AUXLIBS='-L/usr/lib -L/usr/lib/openssl -lssl -lcrypto -L/usr/lib/sasl2 -lsasl2 -lpcre -lz -lm -lldap -llber -Wl,-rpath,/usr/lib/openssl -pie -Wl,-z,relro' OPT='-O' DEBUG='-g'
view rawmake_postfix_32.sh hosted with ❤ by GitHub
For a 64-bit (x86_64) OS:

#/bin/sh

make makefiles CCARGS='-fPIC -DUSE_TLS -DUSE_SSL -DUSE_SASL_AUTH -DUSE_CYRUS_SASL -DPREFIX=\\"/usr\\" -DHAS_LDAP -DLDAP_DEPRECATED=1 -DHAS_PCRE -I/usr/include/openssl -I/usr/include/sasl  -I/usr/include' AUXLIBS='-L/usr/lib64 -L/usr/lib64/openssl -lssl -lcrypto -L/usr/lib64/sasl2 -lsasl2 -lpcre -lz -lm -lldap -llber -Wl,-rpath,/usr/lib64/openssl -pie -Wl,-z,relro' OPT='-O' DEBUG='-g'
view rawmake_postfix_64.sh hosted with ❤ by GitHub
Notice that only difference between the above scripts is that the 32-bit version uses /usr/lib and the 64-bit uses /usr/lib64.

When run, the example scripts above will create a Makefile with instructions to compile Postfix with TLS, SSL, Cyrus-based SASL_AUTH, LDAP, and PCRE support. I also kept the original fPIC and hardening options that were used in the original binaries. Postfix will compile and run just fine without them, but again – my goal was to make an updated set of binaries that were as close as possible to the original ones, so I kept them in there.

Adding Additional Support (such as MySQL)
If you need to compile Postfix with support for any additional options, then you’ll need to add support for them in your Makefile. For example, to add MySQL support, add a few necessary additional arguments to your make makefiles command so that it looks like this:

For a 32-bit OS with MySQL support:

#/bin/sh

make makefiles CCARGS='-fPIC -DUSE_TLS -DUSE_SSL -DUSE_SASL_AUTH -DUSE_CYRUS_SASL -DPREFIX=\\"/usr\\" -DHAS_LDAP -DLDAP_DEPRECATED=1 -DHAS_PCRE -I/usr/include/openssl -DHAS_MYSQL -I/usr/include/mysql -I/usr/include/sasl  -I/usr/include' AUXLIBS='-L/usr/lib -L/usr/lib/openssl -lssl -lcrypto -L/usr/lib/mysql -lmysqlclient -L/usr/lib/sasl2 -lsasl2 -lpcre -lz -lm -lldap -llber -Wl,-rpath,/usr/lib/openssl -pie -Wl,-z,relro' OPT='-O' DEBUG='-g'
view rawmake_postfix_32_myqsl.sh hosted with ❤ by GitHub
For a 64-bit OS with MySQL support:

#/bin/sh

make makefiles CCARGS='-fPIC -DUSE_TLS -DUSE_SSL -DUSE_SASL_AUTH -DUSE_CYRUS_SASL -DPREFIX=\\"/usr\\" -DHAS_LDAP -DLDAP_DEPRECATED=1 -DHAS_PCRE -I/usr/include/openssl -DHAS_MYSQL -I/usr/include/mysql -I/usr/include/sasl  -I/usr/include' AUXLIBS='-L/usr/lib64 -L/usr/lib64/openssl -lssl -lcrypto -L/usr/lib64/mysql -lmysqlclient -L/usr/lib64/sasl2 -lsasl2 -lpcre -lz -lm -lldap -llber -Wl,-rpath,/usr/lib64/openssl -pie -Wl,-z,relro' OPT='-O' DEBUG='-g'
view rawmake_postfix_64_mysql.sh hosted with ❤ by GitHub
Build Your Makefile
Once you decide which of the example scripts to use to build your make file, just copy and paste the appropriate example into a text file called make_postfix.sh (make sure you create it in the same directory as the Postfix source files you just unpacked), or use any of them as a starting point to edit your own to customize for what you need. Once you create your make_postfix.sh script, you’ll need to make it executable and run it:

chmod 755 make_postfix.sh
./make_postfix
As the script runs, you’ll see a few lines of output as it processes, the last of which will include:

(cat conf/makedefs.out Makefile.in) > Makefile
Install Any Required Libraries
Depending on the CCARGS and AUXLIBS options you chose when updating your Makefile, you may need to install some development libraries before Postfix will compile properly and use them. For example, if you copied my options above, then you’ll need to install the Cyrus, OpenSSL, PCRE, and OpenLDAP libraries (along with their development files), which you can do with the following yum command (if you already have one or more installed, yum will skip them):

yum install cyrus-sasl cyrus-sasl-devel openssl openssl-devel pcre pcre-devel openldap openldap-devel
If you selected other compile options, then you’ll need to install the appropriate libraries for your case. For example, you’ll also need the mysql-devel library installed if you’re compiling with MySQL support.

Make the Postfix Binaries
Once you’ve updated your Makefile with the desired options (and installed any required libraries), you’re ready to build the Postfix binaries. Type:

make
Depending on the speed of your server, it may take a while for the compile to finish. You’ll see lots of output scroll by. If the final lines of the output contain any messages that include the word “Error,” or something else doesn’t look right, then you did something wrong. Don’t worry, just type:

make tidy
to clean up the environment and follow the steps again. Check for libraries that should have been installed, or verify that you included the backslashes properly in your make makefiles command. If the make tidy command doesn’t work for some reason, just copy Makefile.init over the top of Makefile and run make tidy again. If things ever get really hosed, you can always just delete the newly created directory, unpack the Postfix package file and start over.

If everything seemed to compile correctly, you can move on to installing the new binaries.

Install the New Binaries
If you already have Postfix running, then you’ve already got an /etc/postfix/main.cf file you like, you’ve already got your /etc/aliases file configured, and you probably have a number of other existing files that you don’t want to replace. Good news! The following command will just upgrade the Postfix binaries without messing with your configuration files:

make upgrade
Depending on the version of Postfix from which you’re upgrading, you may get any number of informational messages at the end of your upgrade process. These messages will inform you of configuration changes and/or files that exist on your system that are no longer part of Postfix, like this:

 Note: the following files or directories still exist but are no
    longer part of Postfix:

     /etc/postfix/postfix-files
     /etc/postfix/postfix-script /etc/postfix/post-install
     /usr/share/doc/postfix-2.3.3/README_FILES/QMQP_README
That means that those files were part of the older Postfix installation that are either no longer needed by the updated version of Postfix, or that they are now stored somewhere else. You can go ahead and delete them with:

rm /etc/postfix/postfix-files
rm /etc/postfix/post-install
rm /etc/postfix/postfix-script
rm /usr/share/doc/postfix-2.3.3/README_FILES/QMQP_README
As of Postfix 2.9, IPV4 support needs to be explicitly stated in the Postfix main.cf file. So after the upgrade, you’ll also see the following:

COMPATIBILITY: editing main.cf, setting inet_protocols=ipv4.
Specify inet_protocols explicitly if you want to enable IPv6.
In a future release IPv6 will be enabled by default.
which is just telling you that the upgrade process automatically made the necessary changes to your main.cf file.

As of Postfix 2.10, the upgrade process will automatically insert an smtpd_relay_restrictions option in your main.cf file and inform you with this message:

COMPATIBILITY: editing /etc/postfix/main.cf, overriding
smtpd_relay_restrictions to prevent inbound mail from unexpectedly
bouncing.Specify an empty smtpd_relay_restrictions value to keep
using smtpd_recipient_restrictions as before.
If you’re upgrading from Postfix 2.6.6 (which you probably are if you’ve done a clean install of CentOS 6), then expect to see these lines at the end of the upgrade, which are merely informing you of a few edits the upgrade process took care of for you:

Editing /etc/postfix/master.cf, adding missing entry for postscreen TCP service
Editing /etc/postfix/master.cf, adding missing entry for smtpd unix-domain service
Editing /etc/postfix/master.cf, adding missing entry for dnsblog unix-domain service
Editing /etc/postfix/master.cf, adding missing entry for tlsproxy unix-domain service
If you’re upgrading from any Postfix 2.x to Postfix 3.x, you’ll also see this notice:

postfix: Postfix is running with backwards-compatible default settings
postfix: See http://www.postfix.org/COMPATIBILITY_README.html for details
postfix: To disable backwards compatibility use "postconf compatibility_level=2" and "postfix reload"

    Note: the following files or directories still exist but are no
    longer part of Postfix:

     /usr/libexec/postfix/main.cf /usr/libexec/postfix/master.cf
This same warning will appear in your maillog every time Postfix starts.

Post-Upgrade Configuration Changes
I suspect that with the later versions of Postfix, the make upgrade command you just ran probably took care of this next step, but it won’t hurt to do this next step manually just to be sure. Run the built-in script that verifies that your configuration files are good to work with the newer version of Postfix you just installed with:

postfix upgrade-configuration
Chances are you probably won’t see any output from the above command except for the backward compatibility warning  mentioned above, which is fine.

However, one post-upgrade configuration change does need to be made manually. As of Postfix 2.11, the default method of the default master.cf now uses “unix” instead of “fifo” for the pickup and qmgr services in order to avoid periodic disk drive spin-up. Things will still work fine without changing, but depending on your system, making the following configuration change to the new default could speed things up for you. Edit /etc/postfix/master.cf and locate the lines that read:

pickup    fifo  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      fifo  n       -       n       300     1       qmgr
and change them to read:

pickup    unix  n       -       n       60      1       pickup
cleanup   unix  n       -       n       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
Restart Postfix
Finally, you’re ready to restart (a “reload” isn’t sufficient in this case) the updated version of Postfix with:

service postfix restart
and your new version of Postfix should be humming along in place of your old one! You can verify this by doing:

postconf -d | grep mail_version
The updated Postfix version will be displayed.

Congratulations! You’re running a current version of Postfix on RHEL / CentOS!

Recommended: Prevent yum from “updating” Postfix
If you want to protect your manually compiled Postfix installation from being accidentally overwritten by a future yum update (for example, the upgrade from CentOS 5.5 to CentOS 5.6 will revert Postfix to a patched version of 2.3.3), you can instruct yum to exclude Postfix from future upgrades by adding the following line to your /etc/yum.conf file:

exclude=postfix*
If you ever change your mind and want to allow yum to attempt an upgrade, you can comment out or remove that line and then attempt your update.

Special Thanks to Jerrale Gayle for helping me with my original Makefile.
