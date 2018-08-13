
This is a documentation of how I set up an Ubuntu linux cluster for our Computational Chemistry Lab at 
University of Illinois at Chicago.

The cluster includes one server and several clients. User login authentication is managed by OpenLDAP.
Users' home and application directories are exported from the server using NFS. 
The clients use Autofs to mount users' home and applications directories on demand.

# Preparation
After installing Ubuntu on the server and clients, we need to move the admin user out of the `/home` direcrory 
to avoid overiding when we mount `/home` from the server to clients.

We need to also rename the `/home` directory to `/rhome` and create a new empty 
`/home` and mount `/rhome` to `/home`.
This will make things easiser when we want to move all the users' data to another disk partition.
This is done on the server only.

## Move the admin user out of `/home`
This is done on both server and clients.

Change user to root

`~$ sudo -i`

Edit the `passwd` file

`~# vim /etc/passwd`

Change the line near the end of the file:

`adminuser:x:1001:1001:Admin,,,:/home/adminuser:/bin/bash`

to

`adminuser:x:999:999:Admin,,,:/adminuser:/bin/bash`

Edit the `group` file

`~# vim /etc/group`

Change the line near the end of the file:

`adminuser:x:1001:`

to

`adminuser:x:999:`

Move the admin home

`mv /home/adminuser /`

Change the ower

`chown -R adminuser:adminuser /adminuser`

`chown -R adminuser:adminuser /adminuser/.*`


Edit the file `login.defs`

`~# vim /etc/login.defs`

Change the line 

`UID_MIN                   1000`

to 

`UID_MIN                   999`

Change the line

`GID_MIN                   1000`

to

`GID_MIN                   999`

Reboot the computer.


## Move `/home` to `/rhome` 

`sudo mv /home /rhome`

Edit the file `fstab`

`sudo vim /etc/fsatb`

Add the line 

`sudo /rhome   /home   none  bind`

`sudo mount -a`


# Setting up OpenLDAP server

On the server, install:

`sudo apt-get update`

`sudo apt-get install slapd ldap-utils`

Give the password for the admin of the LDAP directory. 

LDAP should run after installed. Check the status by

sudo /etc/init.d/slapd status

## Post-Installation Configuration

`sudo dpkg-reconfigure slapd`

*Omit LDAP server configuration?* **No**

*DNS domain:* **mylab.xx.xx.edu**

*Organization name:* **MyLab**

*Administrator password:* **Same as above**

*Database backend to use:* **MDB**

*Do you want the database to be removed when slapd is purged?* **No**

*Move old databases?* **Yes**

*Allow LDAPv2 protocol?* **No**

## Populating LDAP database

Create a file called `base.ldif`, with the following lines:

`dn: ou=People,dc=mylab,dc=xx,dc=xx,dc=edu`

`objectClass: organizationalUnit`

`ou: People`

`dn: ou=Group,dc=mylab,dc=xx,dc=xx,dc=edu`

`objectClass: organizationalUnit`

`ou: Group`

Add this file to the database

`sudo ldapadd -x -D cn=admin,dc=mylab,dc=xx,dc=xx,dc=edu -W -f base.ldif`

To add the first LDAP user to the database, create a file called `ldapuser1.ldif` with the following lines

`dn: cn=ldapuser1,ou=Group,dc=mylab,dc=xx,dc=xx,dc=edu`

`objectClass: posixGroup`

`cn: ldapuser1`

`gidNumber: 10000`

`dn: uid=ldapuser1,ou=People,dc=mylab,dc=xx,dc=xx,dc=edu`

`objectClass: inetOrgPerson`

`objectClass: posixAccount`

`objectClass: shadowAccount`

`uid: ldapuser1`

`sn: ldapuser1`

`givenName: ldapuser1`

`cn: ldapuser1`

`displayName: ldapuser1`

`uidNumber: 10000`

`gidNumber: 10000`

`userPassword: somepasswd`

`gecos: ldapuser1`

`loginShell: /bin/bash`

`homeDirectory: /home/ldapuser1`

Add it  to the database

`sudo ldapadd -x -D cn=admin,dc=mylab,dc=xx,dc=xx,dc=edu -W -f ldapuser1.ldif`
 
## Modifying the slapd Configuration Database

Create a file called `uid_index.ldif` with the contents:

`dn: olcDatabase={1}mdb,cn=config`

`add: olcDbIndex`

`olcDbIndex: mail eq,sub`

Run the command:

`sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f uid_index.ldif`

Create a file called `schema_convert.conf` with the contents:

`include /etc/ldap/schema/core.schema
include /etc/ldap/schema/collective.schema
include /etc/ldap/schema/corba.schema
include /etc/ldap/schema/cosine.schema
include /etc/ldap/schema/duaconf.schema
include /etc/ldap/schema/dyngroup.schema
include /etc/ldap/schema/inetorgperson.schema
include /etc/ldap/schema/java.schema
include /etc/ldap/schema/misc.schema
include /etc/ldap/schema/nis.schema
include /etc/ldap/schema/openldap.schema
include /etc/ldap/schema/ppolicy.schema
include /etc/ldap/schema/ldapns.schema
include /etc/ldap/schema/pmi.schema`


#
Since we will allow users to login into the server directly, 
we need to configure LDAP client on the server


