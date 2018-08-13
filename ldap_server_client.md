
This is a documentation of how I set up an Ubuntu linux cluster for our Computational Chemistry Lab at 
University of Illinois at Chicago.

The cluster includes one server and several clients. User login authentication is managed by OpenLDAP.
Users' home and application directories are exported from the server using NFS. 
The clients use Autofs to mount users's home and applications directories on demand.

# Preparation
After install Ubuntu on the server and clients, we need to move the admin user out of the /home direcrory 
to avoid overiding when we mount /home from the server to clients.

We need to also rename the /home directory to /rhome and create a new empty /home and mount /rhome to /home.
This will make things easiser when we want to move all the users' data to another disk partition.
This is done on the server only.

## Move the admin user out of /home
This is done on both server and clients.

Change user to root
`~$ sudo -i`

Edit the passwd file
~# vi /etc/passwd

