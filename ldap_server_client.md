
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

`include /etc/ldap/schema/core.schema`

`include /etc/ldap/schema/collective.schema`

`include /etc/ldap/schema/corba.schema`

`include /etc/ldap/schema/cosine.schema`

`include /etc/ldap/schema/duaconf.schema`

`include /etc/ldap/schema/dyngroup.schema`

`include /etc/ldap/schema/inetorgperson.schema`

`include /etc/ldap/schema/java.schema`

`include /etc/ldap/schema/misc.schema`

`include /etc/ldap/schema/nis.schema`

`include /etc/ldap/schema/openldap.schema`

`include /etc/ldap/schema/ppolicy.schema`

`include /etc/ldap/schema/ldapns.schema`

`include /etc/ldap/schema/pmi.schema`

Create a directory:

`mkdir ldif_output`

Run the following commands

`slapcat -f schema_convert.conf -F ldif_output -n 0 | grep corba,cn=schema`

`slapcat -f schema_convert.conf -F ldif_output -n0 -H ldap:///cn={2}corba,cn=schema,cn=config -l cn=corba.ldif`

Edit `cn=corba.ldif` to arrive at the following attributes:

`dn: cn=corba,cn=schema,cn=config`

`...`

`cn: corba`

And remove the following lines from the bottom:

`structuralObjectClass: olcSchemaConfig`

`entryUUID: 52109a02-66ab-1030-8be2-bbf166230478`

`creatorsName: cn=config`

`createTimestamp: 20110829165435Z`

`entryCSN: 20110829165435.935248Z#000000#000#000000`

`modifiersName: cn=config`

`modifyTimestamp: 20110829165435Z`

Run the command

`sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f cn\=corba.ldif`


## Create self-signed Certificate

`sudo apt-get install gnutls-bin ssl-cert`

`sudo sh -c "certtool --generate-privkey > /etc/ssl/private/cakey.pem"`

`sudo vim /etc/ssl/ca.info`

With the following template

`cn = My Lab`

`ca`

`cert_signing_key`

Run the commands

`sudo certtool --generate-self-signed --load-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/ca.info --outfile /etc/ssl/certs/cacert.pem`

`sudo certtool --generate-privkey --bits 1024 --outfile /etc/ssl/private/mylab_slapd_key.pem`

`sudo vim /etc/ssl/mylab.info`

Add the following lines

`organization = My Lab`

`cn = mylab.xx.xx.edu`

`tls_www_server`

`encryption_key`

`signing_key`

`expiration_days = 3650`

Then run

`sudo certtool --generate-certificate --load-privkey /etc/ssl/private/mylab_slapd_key.pem --load-ca-certificate /etc/ssl/certs/cacert.pem --load-ca-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/mylab.info --outfile /etc/ssl/certs/mylab_slapd_cert.pem`

`sudo chgrp openldap /etc/ssl/private/mylab_slapd_key.pem`

`sudo chmod 0640 /etc/ssl/private/mylab_slapd_key.pem`

`sudo gpasswd -a openldap ssl-cert`

`sudo systemctl restart slapd.service`

Create the file called `certinfo.ldif` with the contents

`dn: cn=config`

`add: olcTLSCACertificateFile`

`olcTLSCACertificateFile: /etc/ssl/certs/cacert.pem`

`-`

`add: olcTLSCertificateFile`

`olcTLSCertificateFile: /etc/ssl/certs/mylab_slapd_cert.pem`

`-`

`add: olcTLSCertificateKeyFile`

`olcTLSCertificateKeyFile: /etc/ssl/private/mylab_slapd_key.pem`

Then run

`sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f certinfo.ldif`

# LDAP Authentication

Since we will allow users to login into the server directly, 
we need to configure LDAP client on the server also.

On the client, install

`sudo apt-get install ldap-utils`

On the client, edit 

`sudo vim /etc/ldap/ldap.conf`

Change the parameters `BASE` and `URI` to

`BASE   dc=mylab,dc=xx,dc=xx,dc=edu`

`URI    ldap://IP of the server`

On the server, edit

`sudo vim /etc/ldap/ldap.conf`

Change the parameters `BASE` and `URI` to

`BASE   dc=mylab,dc=xx,dc=xx,dc=edu`

`URI    ldap://localhost`

On both the server and client, install 

`sudo apt-get install libnss-ldapd libpam-ldapd nslcd`

It will ask which services to use, choose at least the three: `passwd`, `group` and `shadow`.

Check authentication:

`getent passwd`

`getent group`

`getent shadow`

we should see `ldapuser1` in the output.

On the server, run 

`sudo pam-auth-update`

In addition to the defaults, select `Automatically create user's home on first login`

On the server 

`su ldapser1`

This first login will create the directory `/home/ldapuser1`.


# Setting up NFS on the server

`sudo apt-get install nfs-kernel-server`

Edit the file `/etc/exports`

`sudo vim /etc/exports`

Add the following lines

/rhome             nn.nn.nn.0/24(rw,sync)
/share             nn.nn.nn.0/24(rw,sync)

`sudo mkdir -p /share/apps`

`sudo /etc/init.d/nfs-kernel-server restart`

# Setting up Autofs on clients

`sudo apt-get install autofs`

Check directories shared by the server

`showmount -e server's IP`

Edit `/etc/ldap/ldap.conf`

`sudo vim /etc/ldap/ldap.conf`

and add

`/home   /etc/auto.home  --timeout=600`

`/share  /etc/auto.share --timeout=600`


Create two map files

`sudo vim /etc/auto.home`

and add

`*    -nfsvers=4 -fstype=auto  server's IP:/rhome/&`

`sudo vim /etc/auto.share`

and add

`apps    -nfsvers=4 -fstype=auto  server's IP:/rhome/apps`

