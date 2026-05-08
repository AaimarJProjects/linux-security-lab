# Linux User Management and Permissions



Linux permissions determine what resources an authenticated user is allowed to access, like what files and directories can the user read, edit, remove, and traverse through, and it also determines what commands a user can carry out. This is important because it determines what each user can access and do limiting how much damage they can do to the system if there account is compromised by an attacker.



## Standard Permissions Model 

There are three standard access levels: the owner (the person who owns the file), the group (the group that is assigned to the file), and others (everyone else on the system). Also there are three permission levels: read, write, and execute. Each permission level means something different whether it is applied to a file or a directory.


A file: read permissions (allows you to view or see the contents in a file), write permissions (allows you to make changes or modify the contents in a file), execute permissions (allows you to run the file as a program or script).

Where as in a directory, the kernel checks read permissions (to determine if you can use "ls" to list all the files in the directory), write permissions (determines if you can create or delete files in the directory), and execute permissions (determines if you have permission to traverse or enter a directory).Read and execute are completely independent capabilities and the kernel checks them separately. Read lets you lists what's there, and execute lets you move through and resolve paths inside the directory.


Each permission level has a numeric value: read = 4, write = 2, and execute = 1 and these numeric values can be added to each other to give each access level a different set of permissions. For example if the owner has a permission numeric value of 7 that means the owner has: read(4) permissions + write(2) permissions + execute(1) permissions = 7. Another example is if the group has a permission numeric value of 5 that means the group has read(4) + execute(1) permissions = 5.

These permissions are added up and commonly expressed like 644. The first number is the owner permissions (read(4) + write(2) = 6), the second number is the group permissions (read(4)= 4), and the last number is others permissions (read(4)= 4), this is commonly used for config files or documents as it allows the owner to view the contents and make changes to the file, but only allows the group and others to view what is in the file. 



### Other Commonly Expressed Permissions:
| Permissions | When they are used |
| --- | --- |
| 777 | Every access level has maximum permissions, never use this, it is really bad and is an instant red flag for files and directories, because it allows anyone with a user account to make changes, and even delete files in a directory even files that don't belong to them. | 
| 600 | Only the owner has read, and write permissions, this is used for secrets like passwords, and private ssh keys where only the owner should be able to view, and make changes to the contents in the file.|
| 700 | Only the owner has read, write, and execute permissions, this is used for private directories. | 
| 755 | This is used in scripts and programs. |



## ACLs
ACLs stand for access control lists and it is used when you need fine grained or specific permissions to be applied to individual users or groups since standard permissions can only be applied to three levels which are the owner, the group and others. With ACLs, I can give each user their own specific permissions on top of the standard permissions, and there is a mask. The mask changes to what ever is the highest configured permissions that you apply to a user or group in the ACL. If you set a mask to cap the effective permissions of all the ACL entries and you add a new ACL entry it silently recalculates the mask, and can undo a manual restriction. A file or directory that has an ACL applied ends with + in the permission string.

### Verification
``` bash
# Creates a file called testmask3.
touch testmask3 

# Adds an ACL entry for the file giving it an access control list.
setfacl -m u:<username>:rw- testmask3

# Views the ACL for the file. Saw the line with user:<username>:rw-.
getfacl testmask3

# Change the mask to cap Permissions to read for all ACL entries. 
setfacl -m m::r-- testmask3

# Viewed the ACL for the file and saw the permissions capped at read for all the added ACL entries #effective: r--.
getfacl testmask3

# Added another user with rw- permissions
setfacl -m u:<another_username>:rw- testmask3

# The mask silently recalculates back to rwx to match the highest configured entry, removing the r-- cap entirely.
getfacl testmask3
```



## Special Permission Bits

### Sticky Bits
Sticky Bits are set on shared directories and prevent you from deleting files in the shared directory that don't belong to you even if you have write permissions unless you are the owner of the directory. Directories that have Sticky Bits special permissions applied to them start with 1000 and end with t or T at the end of the permission string, taking the place where the execute(x) would be for others permissions. When others have execute permission the t is common and when others don't then the T is capital. Example of a directory has sticky bits special permissions is /tmp. /tmp is a world writable file so any user can create and process temporary files, ands sticky bits prevents users from deleting another user's files.

### Verification
```bash
# Creates a directory that is owned by root.
sudo mkdir stickydir

# Makes the directory world writable. 
chmod o+w stickydir 

# Adds sticky bits. 
chmod +t stickydir

# Changes into the directory, stickydir. 
cd stickydir
 
# Creates a file as the current user. 
touch <new_file>

# Creates another file as another user. 
sudo -u testuser1 touch testfile  

# Tried to remove the other file, testfile as the current user, asked me if I wanted to remove write protected regular file I type y for yes and then I got a cannot remove 'testfile': Operation not permitted message.
rm testfile
```
#### The reason behind the cannot remove 'testfile': Operation not permitted message
This is because rm doesn't stop me from deleting the file but it sent the verification message first because I didn't have write permissions to file I was trying to delete, and because the directory has sticky bits applied to it the kernel, the one actually deleting the user checked if user had write permissions for the directory, and it did as the directory was world writable and if the first is true, then it checks if the file is being deleted by the file or directory owner, and if neither is true then the kernel refuses to delete the file.



### SetGID
SetGID is set on directories so that each new file created in the directory has the same group so every file belongs to the project group by default. Directories that have SetGID permissions start with 2000 or have an s or S in the permission string where the group execute(x) would be. The common s means the group has execute permissions and the capital S means the group does not have execute permissions. SetGID permissions are different for files, instead of running a program with your permissions the program runs with the files group's permissions instead. To apply SetGid you must be a member of the directory's currently assigned group to apply the bit. Without SetGID files inherit the creating user's primary group. With SetGid active they inherit the directory's group. Existing file are unaffected when the directory group changes and only files created after the change inherit the updated group.

### Verification
```bash
# Creates a directory. 
mkdir setgid_dir

# Makes the directory world writable 
chmod o+w setgid_dir

# Changes directory's group 
sudo chgrp testgroup setgid_dir

# Tried to apply SetGID, and because I was not a member of testgroup SetGID bits weren't applied. This was because the kernel checked and saw I was not a member of the group and refused to apply the permission bits.
chmod g+s setgid_dir

# Adds the user to testgroup 
sudo usermod -aG testgroup <username>

# Logged out and back in to apply group changes.
su - username 

# Now I was able to apply the SetGID bits  
chmod g+s setgid_dir 

#Changed into the directory 
cd setgid_dir 

# Created files in the directory with other users.
sudo -u <username> touch <filename>

# Verified they all inherited the directories group by default.
ls -l 
```

### SetUID
SetUID is a special permission that allows a file to be executed with the permissions of the owner and not the user who ran it. When a file has SetUID permissions it will start with 4000 or have an s or S by the where execute is located in the owners permission string. A common s means the owner has execute permissions and a capital S means the owner does not have execute permissions. Example of a file with SetUID permissions is /usr/bin/passwd as this file is owned by root and normal users like myself can change my password, because passwd temporarily runs as root to write to /etc/shadow where hashed password are stored.

### Verification
```bash
# Lists the permisions for /usr/bin/passwd file and saw -rws in the owners permissions bits showing the file had SetUID permissions
ls -l /usr/bin/passwd
```



## Filesystem Attributes (chattr)
Filesystem attributes operate below the standard permission system which means that even root cannot modify a file if an immutable flag is set using chattr. This is useful as it helps protect critical config files on a hardened server because even if an attacker gets root access they still can not make changes or delete a file that has an immutable flag set without removing it first.

### Verification
``` bash
# Creates a file. 
touch immutablefile 

# Adds the immutable flag. 
sudo chattr +i immutablefile 

# Attempted to delete the file with temporary root permissions the kernel returned cannot remove 'immutablefile': Operation not permitted.
sudo rm immutablefile

# Writes to the file to try and edit the file with temporary root permissions. Returned Operation not permitted.
sudo echo "I am trying to edit" >> immutableflag 
```



## What is the purpose of /etc/passwd and /etc/shadow 
/etc/passwd was first created to store all user account details like their username, uid, gid, password hash, login shell, and home directory. It is world readable, so anyone or any service can look at it to see user account details to verify what user is doing what, etc.., because of this the password hash is exposed as it could be ready by anyone, and saved in a file allowing a user to copy all the password hashes of user accounts and try and crack them offline. That's why /etc/shadow a separate file was created. /etc/shadow stores password hashes for each user account, and can only be read and written to by root and read by the shadow group preventing regular user from being able to access password hashes unless they had root permissions or were apart of shadow group. Before each hash there is a prefix like $y$ for yescrypt, $6$ for SHA-512 both are secure but watch out for MD5 as this is older, weak and not secure. Now only a placeholder like x is shown for passwords in /etc/passwd. The exact permissions for /etc/shadow is rw-r----- 1 root shadow.



## Umask
Umask controls the default permissions that are given to files and directories. On my Ubuntu virtual machine my umask is 0002 so when I create a new file the default permissions are 664 (0666 - 0002(the umask) = 664) so the owner and the group has read(r) and write(w) permissions, and others have read(r) permissions by default . When I create a directory my default permissions are 775 (0777 - 0002(the umask) = 775) so the owner and the group has read(r), write(w), and execute(x) permissions and others have read(r) and execute(x) permissions. Files start at 0666 and not 0777 because execute permissions should require a deliberate decision and because no file should be runnable as a program unless someone explicitly chose to make it so.



## Sudo
Sudo gives you temporary escalated privileges which is safer than root because each time you use sudo it is logged in auth.log so I am able to see in the logs who did what. Two methods to grant sudo access on Ubuntu are adding a user to the sudo group using ```sudo usermod -aG sudo <username>``` and using visudo which checks syntax before saving preventing you from being locked out of sudo entirely because the sudoers file was saved with a syntax error. For fine grained control using ```sudo visudo``` and adding: username ALL=(ALL:ALL) ALL this breaks down as which user, on which host, can run as which user and group, and which commands so all means no restrictions.



## User Management
When creating a user I need to use -m and -s flags with useradd else the user does not get a home directory and a bash shell which means the user can not really do anything useful. Also I learned that I must use the -aG flag when adding a user to a group because the -G flag alone removes the user from all their other groups which could break their access to things immediately.



## Finding Dangerous Permissions
World writable files are dangerous as any user on the system can modify them. SetUID and SetGID binaries are always worth checking because they run with elevated permissions and if one gets compromised an attacker could gain root access. Also orphaned files which are files with no owner, which are a sign that a user account was deleted but the files were never cleaned up which is a security and housekeeping issue.



## chmod
Chmod is used to change permissions on a file. The kernel checks ownership when using chmod which means my ability to change permissions depends on if I am the owner, and not whether I have permissions to the file or directory. The two ways to change permissions using chmod are by either using the numeric way or the symbolic way. An example using the numeric way is me creating a config file by default my file has 664 permissions but I need it to be 644 because it is a config file so by using ```chmod 644 <filename>``` it changes the group permissions to read only. When using the symbolic way it is a bit different as there are three letters the first is for who like u(for the owner), g(for the group) ,and o(for others),the second letter is for the operator like +(add permission), -(take away permission), =(set exact permissions) and the third letter is the permission r(read), w(write), and x(execute). For example using the symbolic way you would set write permissions to the group using ```chmod g+w <filename>```.



## Ownership

### Chown
Chown is used change the owner of a file or directory. 

#### Verification
```bash
# Change the owner from user1 to user2.
sudo chown user2 <filename or directory_name>

# Change the group to a new group with the owner.
sudo chown user2:newgroup <filename or directory_name>

# Change the group only
sudo chown :newgroup <filename or directory_name>

# Change all the owners or groups in a directory using -R 
sudo chown -R <owner>/<group dir>
```
Be extremely careful when using -R as you wouldn't want to accidentally change the owner or groups of files that you didn't want to.

### Chgrp
Chgrp is used to change the group assigned to a file or directory using ```sudo chgrp newgroup <filename or directory_name>```.

