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
  <a href="https://github.com/dorukalioglu/ldap">
    <img src="https://www.openldap.org/images/headers/LDAPworm.gif" alt="Logo" width="240" height="240">
    <img src="http://www.postfix.org/mysza.gif" alt="Logo" width="240" height="240">
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
  - [Installation](#installation)
    - [Custom Directory Creation](#custom-directory-creation)
    - [A First Test](#a-first-test)
    - [Directory Modification](#directory-modification)
    - [Load an Additional Schemas](#load-an-additional-schemas)
- [Usage](#usage)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)
- [Acknowledgements](#acknowledgements)



<!-- ABOUT THE PROJECT -->
## About The Project

This project made for Authentication of `Lightweight Directory Access Protocol (LDAP)`, `Postfix` and `Dovecot`. **Basically installation of a mailserver**, in order for them to be connected to
each other, and help with address lookup and aliases.
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

To get a local copy up and running follow these simple steps.

### Installation

This is an example of how to list things you need to use the software and how to install them.
* LDAP
```sh
apt-get install slapd ldap-utils
```
After installation, you can see running LDAP port on 389:
```sh
lsof -Pni :389
``` 
<br>

The configuration can be found in __/etc/ldap__ . Here’s a short explanation of the existing files/folders:
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
#### Custom Directory Creation
Now that slapd is running, you can set up your own directory. This can be done by hand (writing and importing LDIF files) or – on Ubuntu – with the Debian Packet Manager (dpkg). This however should only be used for a first-time setup.
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

#### A First Test
This query should return two DNs: your directory (dc=example,dc=com) and your admin account (cn=admin,dc=example,dc=com).

```sh
ldapsearch -x -W -D cn=admin,dc=example,dc=com -b dc=example,dc=com -LLL
```

#### Directory Modification
1. Modifying the config in /etc/ldap/slapd.d directly (not recommended!)
2. Using the ldap-utils (ldapadd, ldapdelete, ldapmodify, …)
3. Using a graphical user interface (i.e. Apache Directory Studio)

#### Load an Additional Schemas
To load a schema with ldapadd, it has to be in LDIF format, so it must be converted first. The conversion can be done with slapcat. You’ll need a config file and an output directory:
```sh
cd /etc/ldap/schema
mkdir ldif_output
touch schema_convert.conf
```
<br>
The schema_convert.conf file contains the schema to be converted (and any dependencies):
<br>

```sh
include /etc/ldap/schema/core.schema
include /etc/ldap/schema/cosine.schema
include /etc/ldap/schema/nis.schema
include /etc/ldap/schema/inetorgperson.schema
include /etc/ldap/schema/postfix-book.schema
include /etc/ldap/schema/postfix.schema
```
<br>

>**Note:**You can find postfix.schema on [here](https://geek.co.il/articles/postfix/postfix.schema).

Start the conversion:

```sh
slapcat -f schema_convert.conf -F ./ldif_output/ -n0
```

Copy the corresponding output file to /etc/ldap/schema:
```sh
cp /etc/ldap/schema/ldif_output/cn\=config/cn\=schema/cn\=\{4\}postfix-book.ldif /etc/ldap/schema/postfix-book.ldif
```
Finally, in the postfix-book.ldif, the following changes need to be made:
  * dn: cn=postfix-book,cn=schema,cn=config
  * cn: postfix-book
  * Remove the metadata starting from structuralObjectClass

Then add it to the directory as follows:
```sh
sudo ldapadd -Y EXTERNAL -H ldapi:/// -f postfix-book.ldif
```

The new attributes of PostfixBookMailAccount are available from now on.
If not, try a restart (/etc/init.d slapd restart).

<!-- USAGE EXAMPLES -->
## Usage

Use this space to show useful examples of how a project can be used. Additional screenshots, code examples and demos work well in this space. You may also link to more resources.

_For more examples, please refer to the [Documentation](https://example.com)_



<!-- ROADMAP -->
## Roadmap

See the [open issues](https://github.com/github_username/repo_name/issues) for a list of proposed features (and known issues).



<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to be learn, inspire, and create. Any contributions you make are **greatly appreciated**.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request



<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE` for more information.



<!-- CONTACT -->
## Contact

Your Name - [@twitter_handle](https://twitter.com/twitter_handle) - email

Project Link: [https://github.com/github_username/repo_name](https://github.com/github_username/repo_name)



<!-- ACKNOWLEDGEMENTS -->
## Acknowledgements

* []()
* []()
* []()





<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/github_username/repo.svg?style=flat-square
[contributors-url]: https://github.com/github_username/repo/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/github_username/repo.svg?style=flat-square
[forks-url]: https://github.com/github_username/repo/network/members
[stars-shield]: https://img.shields.io/github/stars/github_username/repo.svg?style=flat-square
[stars-url]: https://github.com/github_username/repo/stargazers
[issues-shield]: https://img.shields.io/github/issues/github_username/repo.svg?style=flat-square
[issues-url]: https://github.com/github_username/repo/issues
[license-shield]: https://img.shields.io/github/license/github_username/repo.svg?style=flat-square
[license-url]: https://github.com/github_username/repo/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/in/github_username
[product-screenshot]: images/screenshot.png
