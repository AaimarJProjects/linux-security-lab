# Linux Permission and User Management Cheat Sheet



## chmod
``` bash
# Owner read/write permissions, group and others read only used for config files.
chmod 644 <filename> 

# Owner full permissions, group and others read/execute permissions.
chmod 755 <directory_name>

# Owner read/write permissions only, used for private keys.
chmod 600 <filename> 

# Owner full permissions only, used for private directories.
chmod 700 <directory_name> 

# Adds sticky bits
chmod +t <directory_name> 

# Adds SetUID
chmod u+s <filename> 

# Add SetGID
chmod g+s <directory_name or filename>

# Recursive, verify path before running
chmod -R 755 <directory_name>

# Adds execute permissions for owner
chmod u+x <filename> 

# Remove read permissions from others
chmod o-r <filename or directory_name> 

# Add execute permissions for everyone
chmod a+x <filename or directory_name> 

```



## chown
```bash
# Changes owner
sudo chown user <filename or directory_name> 

# Change the owner and the group
sudo chown user:group <filename or directory_name>

# Changes group only
sudo chown :group <filename or directory_name>

# Recursive
sudo chown -R user:group <directory_name> 
```


The owner can always change permissions as the kernel checks ownership for chmod and not the permission bits themselves



## chgrp
```bash
# Changes group only
sudo chgrp group <filename or directory_name>
```


## ACL commands
``` bash
# ACL entry that gives user read/write permissions 
setfacl -m u:user:rw <directory_name>

# Remove user entry
setfacl -x u:user <directory_name> 

# Remove all ACLs
setfacl -b <directory_name> 

# View all ACL entries
getfacl <filename>
```


Session refresh is required after group changes:
```su - <username or newgrp groupname>```



## sudo

Temporary privilege escalation - it is logged, audited and safer than root login.



Two methods to grant sudo access on Ubuntu:

1) sudo group (standard): ```sudo usermod -aG sudo <username>```

2) sudo visudo use this for fine grained control: sudo visudo and add username ALL=(ALL:ALL)ALL



Syntax breakdown:

user  host=(run-as-user: run-as-group) commands



Always use visudo as it checks syntax before saving because a broken sudoers file with no syntax check locks me of sudo entirely.



## User Management
```bash
#  Create a user with a home directory using the flag -m and -s set shell to bash the default is /bin/sh
sudo useradd -m -s /bin/bash <username>

# Set password
sudo passwd <username> 

# Add to group, always use -aG
sudo usermod -aG group <username> 

# Deletes user and home directory
sudo userdel -r <username> 
```


### Key files:

- /etc/passwd - account info, world readable, no hashes.

- /etc/shadow - password hashes, root and shadow group readable only.

- /etc/group - group definitions and membership.

- /etc/skel - template copied into every new directory.



### Password hash prefix:

```$y$``` = yescrypt (modern Ubuntu default, memory hard)

```$6$``` = SHA-512 (older systems, still common in production)



## Finding Dangerous Permissions
``` bash
# World writable files
find / -perm -o+w 2>/dev/null 

# SetUID binaries
find /usr/bin -perm -4000 

# SetGID binaries
find /usr/bin -perm -2000 

# Orphaned files
find / -nouser 2>/dev/null 
```


```2>/dev/null``` suppresses permission denied noise



## Filesystem Attributes

The immutable flag blocks modification even by root


``` bash
# Make immutable
sudo chattr +i <filename>  

# Remove immutable
sudo chattr -i <filename> 

# Check attributes
lsattr <filename> 
```


Used to protect critical config files on hardened servers.

