<p align="right">06-Nov-2014</p>
<H1>Using NFSv4 ACL Inheritance <br />to Share Documents Between Two Groups of Users</H1>

### Objective
This is a example of how to create a top level directory named SharedDocs that will give full access to the members of group1 and group2.  
### Prerequisites
- Microsoft Active Directory Configured for Linux Authentication
- AD Groups group1 and group2 
- AD Users in each of the two groups
- AD User in the Administrators Group to Do the Setup

### Order of ACEs is Important 
The order of the ACEs in the ACL is very important because the ACL is searched from the top until a match is found for the user that is access the file and the search stops there.  For example, if a user who is not the owner of the file nor a member of the POSIX owning group and the EVERYONE@ ACE occurs before before an ACE that is more specific to that user, then the EVERYONE@ ACE will be used to grant permission to the file.

### Administrative Account
Start by logging onto a Linux NFSv4 client with an account that is a member of the Active Directory Administrators group.  The example will use the account dadmin@enas.net.
Next, check that Authentication with Active Directory is setup properly on the NFSv4 client by running the following commands:
	
	$ ssh dadmin@enas.net@kvm00-053
	Warning: Permanently added 'kvm00-053,10.33.0.53' (RSA) to the list of known hosts.
	dadmin@enas.net@kvm00-053's password: 
	Last login: Fri Oct 17 09:59:20 2014 from 10.10.253.18
	[dadmin@kvm00-053 ~]$ id
	uid=10011(dadmin) gid=10544(administrators) groups=10544(administrators),10001(remote desktop users),10004(group1),100081(group4)
You can see id command shows that the damin@enas.net account has a primary group membership of administrators so it has access to a newly exported filesystem an create a new top level directory.

### Command Examples
Start by setting your umask and creating a new directory to share

	umask 002
	rm -rf SharedDocs
	mkdir SharedDocs

Set the POSIX permissions on the directory so the the owner and group have full permissions. This will limit the access to only members of the two groups.

	chmod 0770 SharedDocs

Verify the permissions and ACLs on this new directory.

	ls -ld SharedDocs/
	drwxrwx--- 2 dadmin administrators 8192 Nov  5 18:20 SharedDocs/
	
	nfs4_getfacl SharedDocs/
	A::OWNER@:rwaDxtTcCy
	A::GROUP@:rwaDxtcy
	A::EVERYONE@:tcy

Modify the OWNER@ and GROUP@ ACEs using the -m option on the SharedDocs directory so that they are full control.

	nfs4_setfacl -m A::OWNER@:rwaDxtTcCy A::OWNER@:rwaDxtTnNcCoy SharedDocs
	nfs4_setfacl -m A::GROUP@:rwaDxtcy A:g:GROUP@:rwaDxtTnNcCoy SharedDocs

Add ACES so that group1 and group2 have full access to the directory SharedDocs using the -a option and index of 3 and 4 to control there location in the list. 

	nfs4_setfacl -a A:g:group1@enas.net:rwaDdxtTnNcCy 3 SharedDocs
	nfs4_setfacl -a A:g:group2@enas.net:rwaDdxtTnNcCy 4 SharedDocs

The previous command moved the @EVERYONE ACE to index 5 so that the group1 and group2 ACEs will be used before the EVERONE@ ACE. Look at have the ACL is now using nfs4_getfacl.

	nfs4_getfacl SharedDocs/
	A::OWNER@:rwaDxtTcCy
	A::GROUP@:rwaDxtcy
	A:g:group1@enas.net:rwaDxtcy
	A:g:group2@enas.net:rwaDxtcy
	A::EVERYONE@:tcy

Add inheritable ACEs after the permission ACEs for the OWNER@ and GROUP@ for any files or directories created in SharedDocs directory at index 6 and 7. These permissions will also show up as the POSIX permissions rwx for owner and group on new files and directories.
 
	nfs4_setfacl -a A:fdi:OWNER@:rwaDxtTnNcCoy 6 SharedDocs
	nfs4_setfacl -a A:fdgi:GROUP@:rwaDxtTnNcCoy 7 SharedDocs

Add full control inheritable ACEs for group1 and group2 for all files created in this directory and sub-directories at index 7 and 8.

	nfs4_setfacl -a A:fdgi:group1@enas.net:rwaDdxtTnNcCy 7 SharedDocs
	nfs4_setfacl -a A:fdgi:group2@enas.net:rwaDdxtTnNcCy 8 SharedDocs

Look at the resulting ACL for the SharedDocs directory.
	
	nfs4_getfacl SharedDocs
	A::OWNER@:rwaDxtTcCy
	A::GROUP@:rwaDxtcy
	A:g:group1@enas.net:rwaDxtcy
	A:g:group2@enas.net:rwaDxtcy
	A::EVERYONE@:tcy
	A:fdi:OWNER@:rwaDxtTcCy
	A:fdi:GROUP@:rwaDxtcy
	A:fdig:group1@enas.net:rwaDxtcy
	A:fdig:group2@enas.net:rwaDxtcy
	A:fdi:EVERYONE@:tcy

#### Test the inherited permissions by logging in as as group1Member.

	ssh group1Member@localhost
	cd ../SharedDocs
	ls -al
	total 16
	drwxrwx---  2 dadmin administrators 8192 Nov  5 19:03 .
	drwxrwx--x 28 root   administrators 8192 Nov  5 19:03 ..

	mkdir g1
	echo abc > f1

	ls -al g1
	total 16
	drwxrwx--- 2 group1member group1         8192 Nov  5 19:22 .
	drwxrwx--- 3 dadmin       administrators 8192 Nov  5 19:22 ..
	
	nfs4_getfacl g1
	A::OWNER@:rwaDxtTcCy
	A::GROUP@:rwaDxtcy
	A:g:group1@enas.net:rwaDxtcy
	A:g:group2@enas.net:rwaDxtcy
	A::EVERYONE@:tcy
	A:fdi:OWNER@:rwaDxtTcCy
	A:fdi:GROUP@:rwaDxtcy
	A:fdig:group1@enas.net:rwaDxtcy
	A:fdig:group2@enas.net:rwaDxtcy
	A:fdi:EVERYONE@:tcy

	nfs4_getfacl f1 
	A::OWNER@:rwatTcCy
	A::GROUP@:rwatcy
	A:g:group1@enas.net:rwatcy
	A:g:group2@enas.net:rwatcy
	A::EVERYONE@:tcy

#### Test the inherited permissions by logging in as as group2Member.

	ssh group2Member@localhost
	cd ../SharedDocs
	mkdir g2
	echo abc > f2
	echo abc > g1/f2

	ls -al g1
	total 24
	drwxrwx--- 2 group1member group1         8192 Nov  5 19:32 .
	drwxrwx--- 4 dadmin       administrators 8192 Nov  5 19:32 ..
	-rw-rw---- 1 group2member group2            4 Nov  5 19:32 f2

	nfs4_getfacl g2
	A::OWNER@:rwaDxtTcCy
	A::GROUP@:rwaDxtcy
	A:g:group1@enas.net:rwaDxtcy
	A:g:group2@enas.net:rwaDxtcy
	A::EVERYONE@:tcy
	A:fdi:OWNER@:rwaDxtTcCy
	A:fdi:GROUP@:rwaDxtcy
	A:fdig:group1@enas.net:rwaDxtcy
	A:fdig:group2@enas.net:rwaDxtcy
	A:fdi:EVERYONE@:tcy

	nfs4_getfacl g1/f2 
	A::OWNER@:rwatTcCy
	A::GROUP@:rwatcy
	A:g:group1@enas.net:rwatcy
	A:g:group2@enas.net:rwatcy
	A::EVERYONE@:tcy

	echo def >> f1
	cat f1
	abc
	def

###The SharedDocs directory is now usable by members of group1 and group2
Unity v1 only supports POSIX level enforcement of file permissions so for example the change owner permission will not work until a future version of Unity.


