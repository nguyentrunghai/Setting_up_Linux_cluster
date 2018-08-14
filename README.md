
This documents how I set up an Ubuntu linux cluster for our Computational Chemistry Lab at 
University of Illinois at Chicago.

The cluster includes one server and several clients. User login authentication is managed by OpenLDAP.
Users' home and application directories are exported from the server using NFS. 
The clients use Autofs to mount users' home and applications directories on demand from the server.

# Preparation
After installing Ubuntu (16.04) on the server and clients, we need to move the admin user out of `/home` direcrory 
to avoid overriding when `/home` is mounted.

We need to also rename `/home` directory to `/rhome` and create a new empty 
`/home` and mount `/rhome` to `/home`.
This will make things easiser when we want to move all the users' data to another disk partition.
This is done on the server only.

## Move the admin user out of `/home`
This is done on both server and clients.

Change user to root.

`~$ sudo -i`

Edit the `passwd` file.

`~# vim /etc/passwd`

Change the line near the end of the file.

`adminuser:x:1001:1001:Admin,,,:/home/adminuser:/bin/bash`

to

`adminuser:x:999:999:Admin,,,:/adminuser:/bin/bash`

Edit the `group` file.

`~# vim /etc/group`

Change the line near the end of the file.

`adminuser:x:1001:`

to

`adminuser:x:999:`

Move the admin home

`~# mv /home/adminuser /`

Change the ower

`~# chown -R adminuser:adminuser /adminuser`

`~# chown -R adminuser:adminuser /adminuser/.*`


Edit the file `login.defs`.

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

Add the line.

`/rhome   /home   none  bind`

`sudo mount -a`


# Setting up OpenLDAP Server

On the server, install:

`sudo apt-get update`

`sudo apt-get install slapd ldap-utils`

Give the password for the admin of the LDAP directory. LDAP should run after installed. Check the status by

`sudo /etc/init.d/slapd status`

## Post-Installation Configuration

`sudo dpkg-reconfigure slapd`

*Omit LDAP server configuration?* **No**

*DNS domain:* **mylab.xx.xx.edu**

*Organization name:* **MyLab**

*Administrator password:* Same as above

*Database backend to use:* **MDB**

*Do you want the database to be removed when slapd is purged?* **No**

*Move old databases?* **Yes**

*Allow LDAPv2 protocol?* **No**

## Populating LDAP Database

Create a file called `base.ldif`, with the following lines:

`dn: ou=People,dc=mylab,dc=xx,dc=xx,dc=edu`<br/>
`objectClass: organizationalUnit`<br/>
`ou: People`<br/>
`dn: ou=Group,dc=mylab,dc=xx,dc=xx,dc=edu`<br/>
`objectClass: organizationalUnit`<br/>
`ou: Group`<br/>

Add this file to the database.

`sudo ldapadd -x -D cn=admin,dc=mylab,dc=xx,dc=xx,dc=edu -W -f base.ldif`

To add the first LDAP user to the database, create a file called `ldapuser1.ldif` with the following lines

`dn: cn=ldapuser1,ou=Group,dc=mylab,dc=xx,dc=xx,dc=edu`<br/>
`objectClass: posixGroup`<br/>
`cn: ldapuser1`<br/>
`gidNumber: 10000`<br/>
`dn: uid=ldapuser1,ou=People,dc=mylab,dc=xx,dc=xx,dc=edu`<br/>
`objectClass: inetOrgPerson`<br/>
`objectClass: posixAccount`<br/>
`objectClass: shadowAccount`<br/>
`uid: ldapuser1`<br/>
`sn: ldapuser1`<br/>
`givenName: ldapuser1`<br/>
`cn: ldapuser1`<br/>
`displayName: ldapuser1`<br/>
`uidNumber: 10000`<br/>
`gidNumber: 10000`<br/>
`userPassword: somepasswd`<br/>
`gecos: ldapuser1`<br/>
`loginShell: /bin/bash`<br/>
`homeDirectory: /home/ldapuser1`<br/>

Add it  to the database.

`sudo ldapadd -x -D cn=admin,dc=mylab,dc=xx,dc=xx,dc=edu -W -f ldapuser1.ldif`
 
## Modifying slapd Configuration Database

Create a file called `uid_index.ldif` with the contents:

`dn: olcDatabase={1}mdb,cn=config`<br/>
`add: olcDbIndex`<br/>
`olcDbIndex: mail eq,sub`<br/>

Run the command:

`sudo ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f uid_index.ldif`

Create a file called `schema_convert.conf` with the contents:

`include /etc/ldap/schema/core.schema`<br/>
`include /etc/ldap/schema/collective.schema`<br/>
`include /etc/ldap/schema/corba.schema`<br/>
`include /etc/ldap/schema/cosine.schema`<br/>
`include /etc/ldap/schema/duaconf.schema`<br/>
`include /etc/ldap/schema/dyngroup.schema`<br/>
`include /etc/ldap/schema/inetorgperson.schema`<br/>
`include /etc/ldap/schema/java.schema`<br/>
`include /etc/ldap/schema/misc.schema`<br/>
`include /etc/ldap/schema/nis.schema`<br/>
`include /etc/ldap/schema/openldap.schema`<br/>
`include /etc/ldap/schema/ppolicy.schema`<br/>
`include /etc/ldap/schema/ldapns.schema`<br/>
`include /etc/ldap/schema/pmi.schema`<br/>

Create a directory:

`mkdir ldif_output`

Run the following commands:

`slapcat -f schema_convert.conf -F ldif_output -n 0 | grep corba,cn=schema`

`slapcat -f schema_convert.conf -F ldif_output -n0 -H ldap:///cn={2}corba,cn=schema,cn=config -l cn=corba.ldif`

Edit `cn=corba.ldif` to arrive at the following attributes:

`dn: cn=corba,cn=schema,cn=config`<br/>
`...`<br/>
`cn: corba`<br/>

And remove the following lines from the bottom:

`structuralObjectClass: olcSchemaConfig`<br/>
`entryUUID: 52109a02-66ab-1030-8be2-bbf166230478`<br/>
`creatorsName: cn=config`<br/>
`createTimestamp: 20110829165435Z`<br/>
`entryCSN: 20110829165435.935248Z#000000#000#000000`<br/>
`modifiersName: cn=config`<br/>
`modifyTimestamp: 20110829165435Z`<br/>

Run the command

`sudo ldapadd -Q -Y EXTERNAL -H ldapi:/// -f cn\=corba.ldif`


## Creating Self-Signed Certificate

`sudo apt-get install gnutls-bin ssl-cert`

`sudo sh -c "certtool --generate-privkey > /etc/ssl/private/cakey.pem"`

`sudo vim /etc/ssl/ca.info`

With the following template

`cn = My Lab`<br/>
`ca`<br/>
`cert_signing_key`<br/>

Run the commands

`sudo certtool --generate-self-signed --load-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/ca.info --outfile /etc/ssl/certs/cacert.pem`

`sudo certtool --generate-privkey --bits 1024 --outfile /etc/ssl/private/mylab_slapd_key.pem`

`sudo vim /etc/ssl/mylab.info`

Add the following lines

`organization = My Lab`<br/>
`cn = mylab.xx.xx.edu`<br/>
`tls_www_server`<br/>
`encryption_key`<br/>
`signing_key`<br/>
`expiration_days = 3650`<br/>

This crtificate is good for ten years. Then run

`sudo certtool --generate-certificate --load-privkey /etc/ssl/private/mylab_slapd_key.pem --load-ca-certificate /etc/ssl/certs/cacert.pem --load-ca-privkey /etc/ssl/private/cakey.pem --template /etc/ssl/mylab.info --outfile /etc/ssl/certs/mylab_slapd_cert.pem`

`sudo chgrp openldap /etc/ssl/private/mylab_slapd_key.pem`

`sudo chmod 0640 /etc/ssl/private/mylab_slapd_key.pem`

`sudo gpasswd -a openldap ssl-cert`

`sudo systemctl restart slapd.service`

Create the file called `certinfo.ldif` with the contents

`dn: cn=config`<br/>
`add: olcTLSCACertificateFile`<br/>
`olcTLSCACertificateFile: /etc/ssl/certs/cacert.pem`<br/>
`-`<br/>
`add: olcTLSCertificateFile`<br/>
`olcTLSCertificateFile: /etc/ssl/certs/mylab_slapd_cert.pem`<br/>
`-`<br/>
`add: olcTLSCertificateKeyFile`<br/>
`olcTLSCertificateKeyFile: /etc/ssl/private/mylab_slapd_key.pem`<br/>

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

`BASE   dc=mylab,dc=xx,dc=xx,dc=edu`<br/>
`URI    ldap://IP of the server`<br/>

On the server, edit

`sudo vim /etc/ldap/ldap.conf`

Change the parameters `BASE` and `URI` to

`BASE   dc=mylab,dc=xx,dc=xx,dc=edu`<br/>
`URI    ldap://localhost`<br/>

On both the server and client, install.

`sudo apt-get install libnss-ldapd libpam-ldapd nslcd`

It will ask which services to use, choose at least the three: `passwd`, `group` and `shadow`.

Check authentication:

`getent passwd`

`getent group`

`getent shadow`

We should see `ldapuser1` in the output.

On the server, run 

`sudo pam-auth-update`

In addition to the defaults, select `Automatically create user's home on first login`

On the server 

`su ldapser1`

This first login will create the directory `/home/ldapuser1`.


# Setting up NFS on Server

`sudo apt-get install nfs-kernel-server`

Edit the file `/etc/exports`

`sudo vim /etc/exports`

Add the following lines

`/rhome             nn.nn.nn.0/24(rw,sync)`<br/>
`/share             nn.nn.nn.0/24(rw,sync)`<br/>

`sudo mkdir -p /share/apps`

`sudo /etc/init.d/nfs-kernel-server restart`

# Setting up Autofs on Clients

`sudo apt-get install autofs`

Check directories shared by the server

`showmount -e server's IP`

Edit `/etc/ldap/ldap.conf`

`sudo vim /etc/ldap/ldap.conf`

and add

`/home   /etc/auto.home  --timeout=600`<br/>
`/share  /etc/auto.share --timeout=600`<br/>


Create two map files.

`sudo vim /etc/auto.home`

and add

`*   -fstype=nfs,rw,hard,intr,nodev,exec,nosuid,rsize=8192,wsize=8192   server's IP:/rhome/&`

`sudo vim /etc/auto.share`

and add

`apps    -fstype=nfs,rw,hard,intr,nodev,exec,nosuid,rsize=8192,wsize=8192  server's IP:/share/apps`

Restart Autofs service.

`sudo /etc/init.d/autofs restart`

Check if the user's home directory is mounted:

`cd /home/ldapuser1`


# Installing Modules

Do this on both server and clients.

Download Modules here: http://modules.sourceforge.net/

`tar -xzf modules-4.1.3.tar.gz`

`cd modules-4.1.3/`

`sudo mkdir /share/apps/modules`

`./configure --prefix=/usr/share/Modules --with-modulepath=/share/apps/modules`

`make`

`sudo make install`

`sudo ln -s /usr/share/Modules/init/profile.sh /etc/profile.d/modules.sh`

`sudo ln -s /usr/share/Modules/init/profile.csh /etc/profile.d/modules.csh`

# Installing the First Shared App
Do this on the server only.

Download VMD here: http://www.ks.uiuc.edu/Research/vmd/

`sudo mkdir /share/apps/vmd/1.9.4`

Follow the installation instruction to install vmd in `/share/apps/vmd/1.9.4`

Add a module file for vmd.

`sudo mkdir /share/apps/modules/vmd`

`sudo vim /share/apps/modules/vmd/1.9.4`

Put in the following contents

`#%Module1.0`

`proc ModulesHelp { } {`

`global dotversion`

`puts stderr "VMD v. 1.9.4 \n\thttp://www.ks.uiuc.edu/Research/vmd"`
`}`

`module-whatis "VMD Molecular Visualization Package"`

`prepend-path PATH /share/apps/vmd/1.9.4/bin`

`setenv VMD_PLUGIN_PATH /share/apps/vmd/1.9.4/lib/plugins/LINUXAMD64/molfile`

Check 

`module avail`

`module load vmd/1.9.4`


# References
[1] https://help.ubuntu.com/lts/serverguide/openldap-server.html

[2] https://help.ubuntu.com/community/LDAPClientAuthentication

[3] https://wiki.debian.org/LDAP/NSS

[4] https://help.ubuntu.com/community/AutofsLDAP

