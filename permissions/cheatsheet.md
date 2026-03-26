# Linux Permission and User Management Cheat Sheet



## chmod

chmod 644 file ->owner read/write permissions, group and others read only

chmod 755 file ->owner full permissions, group and others read/execute permissions

chmod 600 file -> owner read/write permissions only, used  for private keys

chmod 700 dir -> owner full permissions only, used for private directories

chmod +t dir -> add sticky bits

chmod u+s file -> add SetUID

chmod g+s dir -> add SetGID

chmod -R 755 dir ->  recursive, verify path before running

chmod u+x file -> adds execute permissions for owner

chmod o-r file -> remove read permissions from others

chmod a+x file -> add execute permissions for everyone





## chown

sudo chown user file or directory -> change owner

sudo chown user:group file or directory ->  change the owner and the group

sudo chown :group file or directory -> change group only

sudo chown -R user:group directory -> recursive



The owner can always change permissions as the kernel checks ownership for chmod and not the permission bits themselves



## chgrp

sudo chgrp group file or directory -> change group only



## ACL commands

setfacl -m u:user:rw file -> gives user read/write permissions

setfacl -x u:user file -> remove user entry

setfacl -b file -> remove all ACLs

getfacl file -> view all ACL entries



Session refresh is required after group changes:
su - username or newgrp groupname



## sudo

Temporary privilege escalation - it is logged, audited and safer than root login.



Two methods to grant sudo access on Ubuntu:

1)sudo group (standard): sudo usermod -aG sudo username

2)sudo visudo use this for fine grained control: sudo visudo and add username ALL=(ALL:ALL)ALL



Syntax breakdown:

user  host=(run-as-user: run-as-group) commands



Always use visudo as it checks syntax before saving because a broken sudoers file with no syntax check locks me of sudo entirely.



## User Management

sudo useradd -m -s /bin/bash user -> create a user with a home directory using the flag -m and -s set shell to bash the default is /bin/sh

sudo passwd user -> set password

sudo usermod -aG group user -> add to group, always use -aG

sudo userdel -r user -> delete user and home directory



Key files:

/etc/passwd -> account info, world readable, no hashes

/etc/shadow -> password hashes, root readable only

/etc/group -> group definitions and membership

/etc/skel -> template copied into every new directory



Password hash prefix:

'$y$' = yescrypt (modern Ubuntu default, memory hard)

'$6$' = SHA-512 (older systems, still common in production)



## Finding Dangerous Permissions

find / -perm -o+w 2>/dev/null -> world writable files

find /usr/bin -perm -4000 -> SetUID binaries

find /usr/bin -perm -2000 -> SetGID binaries

find / -nouser 2>/dev/null -> orphaned files



2>/dev/null suppresses permission denied noise



## Filesystem Attributes

The immutable flag blocks modification even by root



sudo chattr +i file  -> make immutable

sudo chattr -i file -> remove immutable

lsattr file -> check attributes



Used to protect critical config files on hardened servers.

