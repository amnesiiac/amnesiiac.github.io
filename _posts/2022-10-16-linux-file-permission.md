---
layout: post
title: "linux file permission (linux, filesys)"
author: "melon"
date: 2022-10-16 21:30
categories: "2022"
tags:
  - linux
---

### # ls -l
in computing, ls is a command to list computer files and directories in Unix and Unix-like operating systems. It is specified by POSIX and the Single UNIX Specification.

```text
$ ls -l
total 160
drwx------@  8 mac  staff   256B Jul  5 14:47 Applications
drwx------@ 10 mac  staff   320B Jul 27 11:31 Desktop
drwx------+  7 mac  staff   224B Jun 17 15:01 Documents
drwx------@ 22 mac  staff   704B Jul 29 16:35 Downloads
drwx------@ 94 mac  staff   2.9K Jul 23 19:02 Library
drwx------   4 mac  staff   128B Nov 13  2021 Movies
drwx------+  6 mac  staff   192B Nov 18  2021 Music
drwx------+  9 mac  staff   288B Apr 26 10:25 Pictures
drwxr-xr-x+  5 mac  staff   160B Nov 14  2021 Public
drwxr-xr-x   5 mac  staff   160B Jul 29 17:48 file-share
drwxr-xr-x   3 mac  staff    96B Jul 26 17:17 node_modules
-rw-r--r--   1 mac  staff    27B Jun 24 13:47 package-lock.json
drwxr-xr-x  20 mac  staff   640B Jul 29 22:20 workspace
-rw-r--r--   1 mac  staff    86B Jul 26 17:17 yarn.lock
```

corresponding meanings are: filetype, privilege(owner/group/others), extended attribute, links, owner, group, size in bytes, lastmodified time and file/dir.

<hr>

### # filetype
```text
-: normal file
d: directory
l: link file
b: block deivce file
p: pipe file
c: char device file
s: socket file/
```

<hr>

### # umask/mask: permission
in Unix-like systems, each file has a set of attributes that control who can read, write or execute it. When a program creates a file, the file permissions are restricted by the mask.

if the mask has a bit set to "1", then the corresponding initial file permission will be disabled. A bit set to "0" in the mask means that the corresponding permission will be determined by the program and the file system.

in other words, the mask acts as a last-stage filter that strips away permissions as a file is created; each bit that is set to a "1" strips away its corresponding permission. Permissions may be changed later by users and programs using chmod.
```text
r    read     4
w    write    2
x    exec     1
```

setting `umask(0)` means: directory created will have privilege with 777, file created will have privilege of 666.
```text
drwx rwx rwx  # 777
-rw- rw- rw-  # 666
r-- -w- --x  # owner=4, group=2, other=1
```

the umask cmd and corresponding permission relatioship is like:
```txt
+----------------------------------------------------------------------------------------------+
| umask command <br> (oct) | umask = Prohibit Permissions during file creation                 |
|--------------------------|-------------------------------------------------------------------|
| 0  (7-0=7, rwx)          | any permission may be set                  (read, write, execute) |
| 1  (7-1=6, rw-)          | execute permission is prohibited           (read and write)       |
| 2  (7-2=5, r-x)          | write permission is prohibited             (read and execute)     |
| 3  (7-3=4, r\-\-)        | write and execute permission is prohibited (read only)            |
| 4  (7-4=3, -wx)          | read permission is prohibited              (write and execute)    |
| 5  (7-5=2, -w-)          | read and execute permission is prohibited  (write only)           |
| 6  (7-6=1, \-\-x)        | read and write permission is prohibited    (execute only)         |
| 7  (7-7=0, -\-\-)        | all permissions are prohibited             (no permissions)       |
+----------------------------------------------------------------------------------------------+
```

<hr>

### # extended attribute
extended file attributes are file system features that enable users to associate computer files with metadata not interpreted by the filesystem, whereas regular attributes have a purpose strictly defined by the filesystem (such as permissions or records of creation and modification times).

unlike forks, which can usually be as large as the maximum file size, extended attributes are usually limited in size to a value significantly smaller than the maximum file size. Typical uses include storing the author of a document, the character encoding of a plain-text document, or a checksum, cryptographic hash or digital certificate, and discretionary access control information.
```text
drw- rw- rw-@  # indicate the file has extended attribute
```

can use `xattr` to view/modify extended attribute
```text
$ xattr -l file                       # lists the names of all xattrs
$ xattr -w attr_name attr_value file  # sets xattr attr_name to attr_value
$ xattr -d attr_name file             # deletes xattr attr_name
$ xattr -c file                       # deletes all xattrs
$ xattr -h                            # prints help
```

<hr>

### # difference between umask(0022) and umask(022)
What does the extra bit in 0022 means? <br>
The extra bit are `setuid` bit, `setgid` bit and `sticky` bit. <br>

<hr>

### # setuid bit & setgid bit (elevated bit)
setuid and setgid are Unix access rights flags, which allow $(user) to run the exxcuable with the file system permissions of the executable's owner or group respectively and to change behaviour in directories.  
they are often used to allow users on a computer system to run programs with temporarily elevated privileges in order to perform a specific task. 
```text
$ chmod 2775  # setuid=2
$ chmod 4775  # setgid=4
$ chmod 6775  # seetuid=2 + setgid=4 = 6
```

```text
$ ls -l file 
-rw-r--r--  1 mac  staff  4 10 21 22:18 file

$ chmod 2644 file 
$ ls -l file 
-rw-r-Sr--  1 mac  staff  4 10 21 22:18 file

$ chmod 4644 file 
-rwSr--r--  1 mac  staff  4 10 21 22:30 file

$ chmod 6644 file 
-rwSr-Sr--  1 mac staff   4 10 21 22:31 file
```

<hr>

### # sticky bit (unelevated bit)
when a directory's sticky bit is set, the filesystem treats the files in such directories in a special way so only the file's owner, the directory's owner, or root can rename or delete the file.  
without the sticky bit set, any user with write and execute permissions for the directory can rename or delete contained files, regardless of the file's owner. 

```text
$ ls -l file
-rw-r--r--  1 mac  staff  4 10 21 22:18 file

$ chmod 1644 file
$ ls -l file
-rw-r--r-T  1 mac  staff  4 10 21 22:18 file
```

<hr>

### # alternate access control: "+" and "."
with reference to [coreutils.git/ls.c](http://git.savannah.gnu.org/cgit/coreutils.git/tree/src/ls.c?id=v8.21#n3785).

```text
drw- rw- rw-+  # General Access Control List
-rw- rw- rw-.  # SELinux Access Control List
```

following the file mode bits is a single character that specifies whether an alternate access method such as an access control list applies to the file: <br>
1) when the character following the file mode bits is a `space`, there is no alternate access method. <br>
2) GNU ls uses a `.` character to indicate a file with an security context, but no other alternate access method. <br>
3) a file with any other combination of alternate access methods is marked with a `+` character.

<hr>

### # access control list (ACL)
ACLs allow us to apply a more specific set of permissions to a file or directory without (necessarily) changing the base ownership and permissions. They let us "tack on" access for other users or groups.

```text
$ getfacl [option] file/dir
 1:  # file: somedir/     # filename
 2:  # owner: lisa        # owner
 3:  # group: staff       # owner group
 4:  # flags: -s-         # setuid(s), sestgid(s), sticky(t)
 5:  user::rwx            # user permission mode ----------+
 6:  user:joe:rwx         # certain user permission mode   |
 7:  group::rwx           # group permission mode ---------+-> base ACL entries
 8:  group:cool:r-x       # certain group permission mode  |
 9:  mask::r-x            # effective rights mask          |
10:  other::r-x           # other permission mode ---------+
11:  default:user::rwx    # ---+
12:  default:user:joe:rwx # ---+
13:  default:group::r-x   # ---+-> default ACL entries for dir 
14:  default:mask::r-x    # ---+   file has no default ACL)
15:  default:other::---   # ---+
```

for `9: fmask::r-x`: this entry limits the effective rights granted to all groups and to named users. File owner and Others permissions are not affected by the effective rights mask.

```text
$ setfacl [option] [action/specification] file/dir

$ setfacl --set file/dir          # replace previous ACL 
$ setfacl --set-file file/dir     # replace previous ACL 
$ setfacl --modify file/dir       # modify previous ACL -m
$ setfacl --modify-file file/dir  # modify previous ACL -M
$ setfacl --remove file/dir       # remove ACL from previous -x
$ setfacl --remove-file file/dir  # remove ACL from previous -X

$ setfacl -m kenny:rwx /dir       # add rwx for certain user kenny in dir 
```

ref: [get/set ACL](https://www.redhat.com/sysadmin/linux-access-control-lists).  

<hr>

### # Reference
> https://superuser.com/a/230561 <br>
> https://linux.die.net/man/1/getfacl <br>
> https://linux.die.net/man/1/setfacl <br>
> https://www.redhat.com/sysadmin/linux-access-control-lists <br>
> https://stackoverflow.com/a/37792732
