
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


