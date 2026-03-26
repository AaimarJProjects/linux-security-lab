# Linux User Management and Permissions



## What This Week Covered

This week I moved from SSH configuration into Linux permission and user management. I learned how permissions work at the file and directory level, how special permission bits extend the standard model, and how ACLs allow fine grained access beyond the standard three levels explained below. I also learned how to audit a system for dangerous permissions and how filesystem attributes can protect files even from root.



## Permissions

\-There are three standard access levels that I learned are: the owner (the person who owns the file), the group (the group that is assigned to the file), and others (everyone else on the system).



\-I also learned that there are three permission levels: read, write, and execute. Each permission level means something different whether it is applied to a file or a directory.



\-I will start with a file: read permissions (this allows me to view or see the contents in a file), write permissions (this allows me to make changes or modify the contents in a file), execute permissions (this allows me to run the file as a program or script).



\-In a directory, read permissions (allows me to use "ls" to list all the files in the directory), write permissions (allow me to create or delete files in the directory), and execute permissions (determines if I have permission to traverse or enter a directory).



\-Each permission has a numeric value: read = 4, write = 2, and execute = 1. Also each permission can be added to each other to give each access level a different set of permissions. For example if the owner has a permission numeric value of 7 that means the owner has: read(4) permissions + write(2) permissions + execute(1) permissions = 7. Another example is if the group has a permission numeric value of 5 that means the group has read(4) + execute(1) permissions = 5.



\-These permissions are added up and commonly expressed like 644. The first number is the owner permissions (read(4) + write(2) = 6), the second number is the group permissions (read(4)= 4), and the last number is others permissions(read(4)= 4), this is commonly used for config files or documents as it allows the owner to view the contents and make changes to the file but only allows the group and others to view what is in the file.  Below are a few others:



600: owner = 6(read(4) + write(2)) group = 0 and others = 0 so they both have no permissions. I learned that this is used for secrets like passwords and private ssh keys where only the owner should be able to view and make changes to the contents in the file.



777: owner, the group, and others all have read(4) + write(2) + execute(1) permissions. I would never use this as this is really bad and is an instant red flag for files ,and directories as it allows anyone with a user account to make changes and even delete files in a directory even files that don't belong to them.



700:owner = 7 (read(4) + write(2) + execute(1)), group = 0 and others = 0 so they have no permissions. I learned that this is used for private directories.



755: owner = 7 (read(4) + write(2) + execute(1)), group = 5(read(4) + execute(1)), and others = 5(read = 4 + execute (1).This is used in scripts and programs.



## Special Permission Bits

\-Sticky Bits: are set on shared directories and prevents me from deleting files in the shared directory that don't belong to me even if I have write permissions. Directories that have Sticky Bits special permissions applied to them start with 1000 and end with t or T at the end of the permission string, taking the place where the execute(x) would be for others permissions. I learned that when others have execute permission the t is common and when others don't then the T is capital. Example of a directory has sticky bits special permissions is /tmp. /tmp is a world writable file so any user can create and process temporary files and sticky bits prevents a user from deleting another user's files.



\-SetGID: set on directories so that each new file created in the directory has the same group so that every file belongs to the project group by default. Directories that have SetGID permissions start with 2000 or have an s or S in the permission string where the group execute(x) would be. The common s means the group has execute permissions and the capital S means the group does not have execute permissions. SetGID permissions are different for files though as instead of running a program with my group permissions the program runs with the file group permissions instead.



\-SetUID: is a special permission that allows a file to be executed with the permissions of the owner and not the user who ran it. I learned that when a file has SetUID permissions it will start with 4000 or have an s or S by the where execute is located in the owners permission string. A common s means the owner has execute permissions and a capital S means the owner does not have execute permissions. Example of a file with SetUID permissions is /usr/bin/passwd as this file is owned by root and normal users like myself can change my password because passwd temporarily runs as root and to write to /etc/shadow where hashed password are stored.



&#x20;

## Permission Check Order

\-I learned that Linux permission checks are carried out in a specific order and stops at the first match.

\-First Linux checks if I am the file owner and if I am then I have the permissions that were given to the file owner but if I am not then Linux checks if I am a member of the group assigned to the file or directory and if I am then I have the permissions that were assigned to the group but if I am not then I fall into others, and have whatever permissions that was given to others.



## umask

\-Umask controls the default permissions that are given to files and directories. On my Ubuntu virtual machine my umask is 0002 so when I create a new file the default permissions are 664 (0666 - 0002(the umask) = 664) so the owner and the group has read(r) and write(w) permissions, and others have read(r) permissions by default . When I create a directory my default permissions are 775 (0777 - 0002(the umask) = 775) so the owner and the group has read(r), write(w), and execute(x) permissions and others have read(r) and execute(x) permissions.



## chmod

\-Chmod is used to change permissions on a file. The kernel checks ownership for chmod which means my ability to change permissions depends on if I am the owner and not whether I have permissions to the file or directory.

\-The two ways I learned to change permissions using chmod are by either using the numeric way or the symbolic way. An example of me using the numeric way is me creating a config file by default my file has 664 permissions but I need it to be 644 because it is a config file and I don't want the group being able to edit it so by using "chmod 644 filename" I just change group permissions to read only. When using the symbolic way it is a bit different as there are three letters the first is for who like u(for the owner), g(for the group) ,and o(for others),the second letter is for the operator like +(add permission), -(take away permission), =(set exact permissions) and the third letter is the permission r(read), w(write), x(execute).

If I wanted to do the same example using symbolic chmod I would use "chmod g=r filename" to just allow the group to have read permissions.



## Ownership

\-chown is used change the owner of a file or directory. For example to change the owner from user1 to user2 I would use "sudo chown user2 (filename or directoryname)". Also if I wanted to change the group to a new group with owner I could do so by using "sudo chown user2:newgroup (filename or directoryname)" or if I just wanted to change the group I could do so using "sudo chown :newgroup (filename or directoryname)". There is also the option to change all the owners or groups in a directory using "sudo chown -R owner/group dir/" -R stands for recursive as it applies the changes to all the files in the directory. I would be extremely careful when using -R as I wouldn't want to accidentally change the owner or groups of files that I didn't want to.

\-chgrp is used to change the group assigned to a file or directory using "sudo chgrp newgroup (filename or directoryname)".



## ACLs

\-ACLs stand for access control lists and it allows fine grained or specific permissions to be applied to individual users or groups since standard permissions can only be applied to three levels which are the owner, the group and others. With ACLs, I can give each user their own specific permissions ontop of the standard permissions. Also I saw that with ACLs there is a mask which caps the maximum permissions that I can give a user and this caught me off guard because when I set permissions higher than the mask my effective permissions ended up lower than what I configured. Lastly I saw that when a file or directory has an ACL it ends with with an A in the permission string.



## sudo

\-I learned that sudo gives me temporary escalated privileges and that it is safer than root because each time I use sudo it is logged so I am able to see in the logs who did what. Two methods that I learned to grant sudo access on Ubuntu are adding a user to the sudo group using "sudo usermod -aG sudo username" and using visudo which checks syntax before saving preventing me from being locked out of sudo entirely because the sudoers file was saved with a syntax error. For fine grained control using "sudo visudo" and adding: username ALL=(ALL:ALL) ALL this breaks down as which user, on which host, can run as which user and group, and which commands so all means no restrictions.

## User Management

\-I learned that when creating a user I need to use -m and -s flags with useradd else the user does not get a  home directory and a bash shell which means the user can not really do anything useful. Also I learned that I must use the -aG flag when adding a user to a group because the -G flag alone removes the user from all their other groups which could break their access to things immediately.



## Filesystem Attributes

\-I learned that filesystem attributes operate below the standard permission system which means that even root cannot modify a file if an immutable flag is set using chattr. This is useful as it helps protect critical config files on a hardened server because even if an attacker gets root access they still can not make changes or delete a file that has an immutable flag set without removing it first.



## Finding Dangerous Permissions

\-I learned that world writable files are dangerous as any user on the system can modify them. SetUID and SetGID binaries are always worth checking because they run with elevated permissions and if one gets compromised an attacker could gain root access. Also I learned that orphaned files which are files with no owner are a sign that a user account was deleted but the files were never cleaned up which is a security and housekeeping issue.



