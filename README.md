<!-- [![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]
[![LinkedIn][linkedin-shield]][linkedin-url]
 -->


<!-- PROJECT LOGO -->
<br />
<p align="center">
  <a href="https://www.openldap.org/">
    <img src="https://www.openldap.org/images/headers/LDAPworm.gif" alt="Logo" width="240" height="240">
  </a>
  <br>
  <a href="http://www.postfix.org">
    <img src="http://www.postfix.org/mysza.gif" alt="Logo" width="240" height="240">
  </a>
  <a href="https://www.dovecot.org">
    <img src="https://www.dovecot.org/typo3conf/ext/dvc_content/Resources/Public/Images/dovecot_logo_vector.svg" alt="Logo" width="240" height="240">
  </a>

  <h3 align="center">LDAP Postfix and Dovecot Auth on Ubuntu 20.04</h3>
</p>



<!-- TABLE OF CONTENTS -->
## Table of Contents

- [Table of Contents](#table-of-contents)
- [About The Project](#about-the-project)
  - [Built With](#built-with)
- [Getting Started](#getting-started)
  - [LDAP](#ldap)
    - [LDAP Installation](#ldap-installation)
    - [Verify the LDAP Search](#verify-the-ldap-search)
    - [Custom Directory Creation and Directory Modification](#custom-directory-creation-and-directory-modification)
    - [Load an Additional Schemas](#load-an-additional-schemas)
    - [Adding Users by LDIF File](#adding-users-by-ldif-file)
    - [LDAP Configuration for Test](#ldap-configuration-for-test)
  - [Postfix](#postfix)
    - [Postfix Installation](#postfix-installation)
    - [Postfix Configuration](#postfix-configuration)
    - [Starting Postfix](#starting-postfix)
    - [SMTP Connection Test](#smtp-connection-test)
    - [LDAP Lookup Test](#ldap-lookup-test)
  - [Dovecot](#dovecot)
    - [Dovecot Installation](#dovecot-installation)
    - [Dovecot Configuration](#dovecot-configuration)
    - [Starting Dovecot](#starting-dovecot)
    - [IMAP Connection & Authentication Test](#imap-connection--authentication-test)
    - [Postfix Integration](#postfix-integration)
    - [Dovecot Deliver Test](#dovecot-deliver-test)
    - [Log Viewing](#log-viewing)



<!-- ABOUT THE PROJECT -->
## About The Project

This project made for Authentication of `Lightweight Directory Access Protocol (LDAP)`, `Postfix` and `Dovecot`. **Basically installation of a mailserver**, in order for them to be connected to each other, and help with address lookup and aliases.
<br>
* `Ubuntu 20.02`<br>
* `Users don’t have system accounts on the Unix machine`<br>
* `User accounts are virtual accounts stored in an LDAP`


### Built With

* [OpenLDAP](https://www.openldap.org/)
* [Postfix](http://www.postfix.org/)
* [Dovecot](https://www.dovecot.org/)



<!-- GETTING STARTED -->
## Getting Started

### LDAP 

This is the directory service that stores the (virtual) user accounts for the mail server.
The LDAP directory needs to keep several information needed by Postfix and Dovecot to work.

- Username & Password for authentication
- __UID__ and __GID__ for managing permissions of the user's __maildir__
- Location of the __maildir__
- List of mail aliases for a given user to allow one user to have several mail adresses.

#### LDAP Installation

```sh
apt-get install slapd ldap-utils
```
After installation, you can see running LDAP port on 389:
```sh
lsof -Pni :389
``` 
<br>

The configuration files and what are they used for, in __/etc/ldap__.
<br>
| LDAP Folder                 | Explanation                                                     |
| --------------------------- | --------------------------------------------------------------- |
| sasl2/                      | Used for SASL authentication. Initially empty and unconfigured. |
| schema/                     | Contains the included schema and ldif files.                    |
| slapd.d/                    | The LDAP server’s configuration storage.                        |
| slapd.d/cn=config           | Contains the server configuration and directory databases.      |
| slapd.d/cn=config/cn=schema | Contains the currently loaded schemas.                          |
| ldap.conf                   | Used to define system-wide defaults for LDAP clients.           |


The actual database, that is automatically built from this configuration, is stored in /var/lib/ldap.


#### Verify the LDAP Search
To verify the ldap server is configured successfully, you can use the below command and verify that the domain entry is present.

```sh
ldapsearch -x -W -D cn=admin,dc=example,dc=com -b dc=example,dc=com -LLL
```
#### Custom Directory Creation and Directory Modification

You can set up your own directory by (writing and importing LDIF files) or with the Debian Packet Manager (dpkg). This should only be used for a first-time setup.

```sh
dpkg-reconfigure slapd
```
<br>

| Config Question                       | Answer      | Explanation                                                                          |
| ------------------------------------- | ----------- | ------------------------------------------------------------------------------------ |
| Omit OpenLDAP server configuration?   | No          | This will start the configuration wizard.                                            |
| DNS domain name:                      | example.com | Name of your directory (this will result in a BaseDN of the form dc=example,dc=com). |
| Organization name:                    | example     | Name of your organization.                                                           |
| Administrator password:               | secret      | New password for the LDAP Administrator (cn=admin,dc=example,dc=com).                |
| Database backend to use:              | HDB         | Based on Oracle Berkeley Database (BDB) but more effective.                          |
| Remove database when slapd is purged? | No          | Keep the database if OpenLDAP is uninstalled.                                        |
| Move old database?                    | Yes         | Remove the old database so that it does not interfere with the new configuration.    |

The dc=nodomain has now been replaced with dc=example,dc=com.

You can modify directory with: 
1. Using the ldap-utils (ldapadd, ldapdelete, ldapmodify, …)
2. Using a graphical user interface (i.e. Apache Directory Studio)

#### Load an Additional Schemas
For sure, to use e-mail address on accounts, we need to provide the types:
- attributetype mailacceptinggeneralid to list typical aliases for the local part of the address
- attributetype maildrop for the final mail box destination used by Postfix
- objectclass postfixUser to identify users who receive mail by Postfix

> You can find the ldif file on [here](src/postfix.ldif). <br> &ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;&ensp;*Thanks to [Credativ](https://github.com/credativ/postfix-ldap-schema)*.

Use this line to add custom ldif file:
```sh
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/ldap/postfix.ldif
```
The new attributes are avaiable from now on.
If not, try a restart (__/etc/init.d slapd restart__).

> To load a schema with ldapadd, it has to be in LDIF format, so it must be converted first. 
> - The conversion can be done with ```slapcat```. You’ll need a config file and an output directory:<br>
 ```cd /etc/ldap/schema```
  ```mkdir ldif_output```
  ```touch schema_convert.conf``` <br>
> - The schema_convert.conf file contains the schema to be converted (and any dependencies):<br>
  ```include /etc/ldap/schema/core.schema```<br>
  ```include /etc/ldap/schema/cosine.schema```<br>
  ```include /etc/ldap/schema/nis.schema```<br>
  ```include /etc/ldap/schema/inetorgperson.schema```<br>
  ```include /etc/ldap/schema/postfix.schema``` <br>
> - Start the conversion: <br>
```slapcat -f schema_convert.conf -F ./ldif_output/ -n0``` <br>
```cp /etc/ldap/schema/ldif_output/cn\=config/cn\=schema/cn\=\{4\}postfix.ldif /etc/ldap/schema/postfix.ldif```
> - Finally, in the postfix-book.ldif, the following changes need to be made: <br>* dn: cn=postfix,cn=schema,cn=config<br>* cn: postfix<br>* Remove the lines starting from __structuralObjectClass__

>**Note:**You can find postfix.schema on [here](https://geek.co.il/articles/postfix/postfix.schema). Also other contributed schemas available on [here](https://github.com/credativ/postfix-ldap-schema)


#### Adding Users by LDIF File
Create a userimport.ldif with the following content to add a single user (the value for userPassword can be generated with slappasswd):
```sh
# Some User
dn: uniqueIdentifier=test1,ou=people,dc=example,dc=com
objectClass: organizationalPerson
objectClass: person
objectClass: top
objectClass: PostfixBookMailAccount
objectClass: extensibleObject
cn: Test1 User
givenName: Test1
mail: test1@example.com
mailEnabled: TRUE
mailGidNumber: 5000
mailHomeDirectory: /srv/vmail/test1@example.com
mailQuota: 10240
mailStorageDirectory: maildir:/srv/vmail/test1@example.com/Maildir
mailUidNumber: 5000
sn: Test1
uniqueIdentifier: test1
userPassword: {SSHA}Ifz0oceGr1wwBP1BtBduPLTVbo6A2Qkd
```

Then add it using this command:
```sh
ldapadd -W -D "cn=admin,dc=example,dc=com" -f userimport.ldif
```
>**Note**:You can get SSHA decrypted password by using `passwd`.

#### LDAP Configuration for Test

The BaseDN for this howto is “dc=example,dc=com” and it contains two organizationalUnits:

  - “ou=people,dc=example,dc=com” for user accounts (RDN = uniqueIdentifier)
  - “ou=services,dc=example,dc=com” for service accounts (RDN = uid)
  - You’ll need accounts for postfix and dovecot

LDAP structure should be like:
<br>
<img src="images/mailserver_ldap_config_01.png" width="100%" height="100%"> &ensp; &ensp;<img src="images/mailserver_ldap_config_02.png" width="100%" height="100%">

### Postfix 
Postfix is the default Mail Transfer Agent (MTA) in Ubuntu. It attempts to be fast and secure, with flexibility in administration. It is compatible with the MTA sendmail.

#### Postfix Installation
To install Postfix run the following command and during installation select the “No Configuration” option:
```sh
apt-get install postfix postfix-pcre postfix-ldap
```
Optionally, you can install extra tools for managing mailboxes (mutt) and sending testmails (swaks):

```sh
apt-get install mutt swaks
```

> __Note:__ If you have some access problems, set ```/etc/postfix/``` owner to ```postfix:postfix``` and its mode to ```0100```. 
#### Postfix Configuration

Before starting Postfix, some configuration files need to be created and edited in __/etc/postfix__. For an example, check the files:

* [main.cf](src/main.cf) -> This is the main configuration file for Postfix
* [virtual_domains](src/virtual_domains.cf) –> Contains the domains the server takes mails for
* [ldap_virtual_recipients.cf](src/ldap_virtual_recipients.cf) –> LDAP query for recipient validation
* [ldap_virtual_aliases.cf](src/ldap_virtual_aliases.cf) –> LDAP query to get aliases and forwarding addresses 
* [identitycheck.pcre](src/identitycheck.pcre) –> Regular expression to block clients that use your hostname
* [drop.cidr](src/drop.cidr) –> Contains blacklisted IP addresses

> __Note:__ You can use also ```dpkg-reconfigure postfix``` as a start. <br>
> __Note: If any changes done in Postfix files, especially in main.cf, need to restart Postfix service!!!__

#### Starting Postfix

Start Postfix with the following command:  ```sh service postfix start ```

Check if it is running:  ```sh lsof -Pni :25 ```

Have a look at __/var/log/mail.log__ to see if there are any errors. It's helps very nice to see error that you can't see.

#### SMTP Connection Test
Use Telnet to connect to the SMTP server:  ```sh telnet 127.0.0.1 25 ```. When you are connected, send an ```sh EHLO client```. You should get the following response:
>Trying 127.0.0.1... <br>
Connected to 127.0.0.1.<br>
Escape character is '^]'.<br>
220 mail.example.com ESMTP Postfix (Ubuntu)<br>
EHLO client<br>
250-mail.example.com<br>
250-PIPELINING<br>
250-SIZE 10240000<br>
250-ETRN<br>
250-AUTH DIGEST-MD5 NTLM CRAM-MD5 LOGIN PLAIN<br>
250-AUTH=DIGEST-MD5 NTLM CRAM-MD5 LOGIN PLAIN<br>
250-ENHANCEDSTATUSCODES<br>
250-8BITMIME<br>
250 DSN<br>
QUIT<br>
221 2.0.0 Bye<br>

#### LDAP Lookup Test

You can test the mapping with the ```postmap``` command
```sh
postmap -q alice@example.com ldap:/etc/postfix/ldap_virtual_recipients.cf
postmap -q postmaster@example.com ldap:/etc/postfix/ldap_virtual_aliases.cf
```
Both of the above commands should return “alice@example.com”.

### Dovecot
Dovecot as LDA (Local Delivery Agent). This is the POP3 and IMAP server. It accepts incoming mail from Postfix and stores it in virtual mailboxes. It is connected to the LDAP for user authentication and lookups.

#### Dovecot Installation
Install Dovecot and the necessary extensions. During installation, you will be asked if a certificate should be created. If you skip this, you can add your own certificate later.
```
apt-get install dovecot-core dovecot-imapd dovecot-pop3d dovecot-lmtpd dovecot-ldap
```
#### Dovecot Configuration 
The configuration is spread over several files in __/etc/dovecot/__ and __/etc/dovecot/conf.d__ and contains some basic configuration blocks, which can be included by uncommenting them. ```doveconf -n``` will print the final configuraton build up out of all snippets. <br>

Disable unwanted protocols (IMAPS and POP3S) by setting the ports to 0 in __/etc/dovecot/conf.d/10-master.conf__. Also, set the permissions, user and group for the authentication-userdb:
```
inet_listener imaps {
    port = 0
    #port = 993
    #ssl = yes
}
inet_listener pop3s {
    port = 0
    #port = 995
    #ssl = yes
}
unix_listener auth-userdb {
    mode = 0600
    user = vmail
    group = vmail
}
```
Dovecot can use LDAP as a password database for authentication as well as a user database for information like the maildir location, and the UID and GID of the user. To activate LDAP as a password and user database, enable it in __/etc/dovecot/conf.d/10-auth.conf__ and disable __auth-system.conf.ext__.
```
auth_mechanisms = plain login
#!include auth-system.conf.ext
!include auth-ldap.conf.ext
```
__/etc/dovecot/conf.d/auth-ldap.conf.ext__ should contain the declaration for __passdb__ and __userdb__. Set the LDAP-related parameters in __/etc/dovecot/dovecot-ldap.conf.ext__:
```
hosts = 127.0.0.1
dn = uid=dovecot,ou=services,dc=example,dc=com
dnpass = secret
ldap_version = 3
base = ou=people,dc=example,dc=com
user_attrs = mailHomeDirectory=home,mailUidNumber=uid,mailGidNumber=gid,mailStorageDirectory=mail
user_filter = (&(objectClass=PostfixBookMailAccount)(uniqueIdentifier=%n))
pass_attrs = uniqueIdentifier=user,userPassword=password
pass_filter = (&(objectClass=PostfixBookMailAccount)(uniqueIdentifier=%n))
default_pass_scheme = CRYPT

passdb {
  driver = ldap

  # Path for LDAP configuration file, see example-config/dovecot-ldap.conf.ext
  args = /etc/dovecot/dovecot-ldap.conf.ext
}
userdb {
  driver = ldap
  args = /etc/dovecot/dovecot-ldap.conf.ext
  
  # Default fields can be used to specify defaults that LDAP may override
  #default_fields = home=/home/virtual/%u
}
```
Activate logging in __/etc/dovecot/conf.d/10-logging.conf__:
```
log_path = syslog
syslog_facility = mail
auth_debug = yes
```
The location of the Maildir is set in __/etc/dovecot/conf.d/10-mail.conf__. The user's homedirectory will be read from LDAP, and '/mailbox' will be appended with the following configuration:
```
maildir:~/mailbox
```

Add a system user and group named vmail with uid and gid 5000:
```
addgroup --system --gid 5000 vmail
adduser --system --home /srv/vmail --uid 5000 --gid 5000 --disabled-password --disabled-login vmail
```
Make sure that __/srv/vmail__ has been created.


#### Starting Dovecot
Start Dovecot with ```service dovecot start``` and check if it is running with  ```lsof -Pni — port 110``` and __143__ should be open. If it isn’t running, check the logfiles (mail.log and syslog). If there are no errors in the log, start Dovecot in foreground mode to have the errors printed to the console:  dovecot -F.

#### IMAP Connection & Authentication Test
Use Telnet to connect to Dovecot’s IMAP Server: ```telnet 127.0.0.1 143```. Then send a login request ( ```1 login alice@example.com secret```) to see if authentication (plain) is working:

```Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
* OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE STARTTLS AUTH=PLAIN AUTH=LOGIN] Dovecot (Ubuntu) ready.
1 login alice@example.com secret
1 OK [CAPABILITY IMAP4rev1 LITERAL+ SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS SPECIAL-USE BINARY MOVE] Logged in
2 logout
* BYE Logging out 
```
#### Postfix Integration
Activate Dovecot Deliver in Postfix by adding these lines to __/etc/postfix/master.cf__:
```
dovecot   unix  -       n       n       -       -       pipe
        flags=ODRhu user=vmail:vmail argv=/usr/lib/dovecot/deliver -e -f ${sender} -d ${recipient}
```
Set the postmaster address in __/etc/dovecot/conf.d/15-lda.conf__:
```
postmaster_address = postmaster@example.com
```
Restart Dovecot and Postfix:
```
service dovecot restart
service postfix restart
```
#### Dovecot Deliver Test
Send a testmail to alice@example.com using one of the following methods:

```
echo Testmail | sendmail -f bob@example.com alice@example.com 
```
```
swaks --from bob@example.com --to alice@example.com --server 127.0.0.1:25
```

```
telnet 127.0.0.1 25

Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
220 mail.example.com ESMTP Postfix (Ubuntu)
EHLO test.local
250-mail.example.com
250-PIPELINING
250-SIZE 10240000
250-ETRN
250-AUTH DIGEST-MD5 NTLM CRAM-MD5 LOGIN PLAIN
250-AUTH=DIGEST-MD5 NTLM CRAM-MD5 LOGIN PLAIN
250-ENHANCEDSTATUSCODES
250-8BITMIME
250 DSN
MAIL FROM:<bob@example.com>
250 2.1.0 Ok
RCPT TO:<alice@example.com>
250 2.1.5 Ok
DATA
354 End data with <CR><LF>.<CR><LF>
Testmail
.
250 2.0.0 Ok: queued as 51586A1146
QUIT
221 2.0.0 Bye
```
Check the log /var/log/mail.log and Alice’s mailbox:  ```mutt -f /srv/vmail/alice@example.com/Maildir/ ```

#### Log Viewing
Postfix sends all log messages to __/var/log/mail.log__. However, error and warning messages can sometimes get lost in the normal log output so they are also logged to __/var/log/mail__.err and __/var/log/mail.warn__ respectively.

**Note that** 
Files and folders containing passwords:
- /etc/postfix/ldap_virtual_aliases.cf <br>
- /etc/postfix/ldap_virtual_recipients.cf<br>
- /etc/dovecot/dovecot-ldap.conf.ext<br>
- /etc/saslauthd.conf<br>
- /var/www/html/roundcube/config/config.inc.php

Folders containing private keys:
- /etc/postfix/certs<br>
- /etc/dovecot/private<br>
- /etc/ssl/private (possibly for your Apache key)

>For some reason, I need to change ```/etc/dovecot/dovecot.conf``` file to get rid of errors:
>```
>service stats {
>    unix_listener stats-reader {
>        user = vmail
>        group = vmail
>        mode = 0666
>    }
>
>    unix_listener stats-writer {
>        user = vmail
>        group = vmail
>        mode = 0666
>    }
>}
>```





Source: <br>
- Mastering CentOS 7 Linux Server<br>