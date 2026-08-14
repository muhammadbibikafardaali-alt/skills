# Linux Permissions & Security Lab --- Session Notes

## Overview

In this session I continued building my Linux administration and
security skills through hands-on testing rather than only reading about
permissions.

The main goal was to understand **special Linux permission bits**, how
Linux decides which identity a process runs as, and how **ACLs (Access
Control Lists)** provide more precise permissions than the traditional
owner/group/others model.

I worked with real users (`mo` and `ali`) and tested the behavior
directly on Ubuntu.

------------------------------------------------------------------------

## 1. Sticky Bit

I first inspected `/tmp`:

``` bash
ls -ld /tmp
```

Example output:

``` text
drwxrwxrwt root root /tmp
```

### What the command means

-   `ls` --- list file or directory information.
-   `-l` --- use long format, including permissions, owner and group.
-   `-d` --- show information about the directory itself instead of
    listing its contents.

### Reading the permissions

``` text
drwxrwxrwt
```

-   `d` --- directory.
-   `rwx` --- owner (`root`) can read, write and traverse.
-   `rwx` --- group can read, write and traverse.
-   `rwt` --- others have read/write/traverse access, with the **sticky
    bit** enabled.

For directories:

-   `r` --- list directory entries.
-   `w` --- create, delete or rename entries.
-   `x` --- traverse/access the directory.

### Why `/tmp` needs the sticky bit

Without the sticky bit, a world-writable directory such as:

``` text
drwxrwxrwx
```

would allow users with `w+x` on the directory to delete other users'
files, even when they cannot read or modify those files.

The sticky bit restricts deletion and renaming in a shared writable
directory.

I tested this by creating a file owned by `mo` in `/tmp` and trying to
remove it as `ali`.

``` bash
sudo -u ali rm /tmp/mo-sticky-test.txt
```

Result:

``` text
Operation not permitted
```

The sticky bit prevented `ali` from deleting another user's file.

### Sticky bit does not protect file contents

Initially the file had:

``` text
-rw-rw-r-- mo mo
```

Because `ali` was a member of group `mo`, he could modify the contents
even though the sticky bit prevented deletion.

This demonstrated an important distinction:

-   **File permissions** control access to the file contents.
-   **Directory permissions and sticky bit** control operations on
    directory entries such as deletion and renaming.

I then changed the file to:

``` bash
chmod u=rw,g=,o= /tmp/mo-sticky-test.txt
```

This produced mode:

``` text
600
-rw-------
```

Now `ali` could neither read nor modify the contents, while the sticky
bit still protected deletion/renaming.

------------------------------------------------------------------------

## 2. Setgid on Directories

I used the existing shared directory:

``` text
/home/mo/permissions-lab
```

Before setgid, a file created by `ali` looked like:

``` text
-rw-r--r-- ali ali ali-created.txt
```

The user owner was `ali`, and the group owner was also `ali`, because
`ali`'s primary group is `ali`.

I enabled setgid on the directory:

``` bash
chmod g+s /home/mo/permissions-lab
```

### Command meaning

-   `chmod` --- change permissions.
-   `g` --- group permission class.
-   `+` --- add a permission.
-   `s` --- enable setgid.

The directory changed from:

``` text
drwxrwxr-x
```

to:

``` text
drwxrwsr-x
```

The lowercase `s` in the group execute position means group execute is
enabled and the setgid bit is set.

### Effect of setgid on a directory

After setgid was enabled, new files created inside the directory
inherited the **group ownership of the parent directory**.

Conceptually:

``` text
Parent directory group: mo
          |
ali creates a file
          |
user owner:  ali
group owner: mo
```

Important lesson:

> Setgid controls group inheritance. It does not automatically decide
> whether that group can read or write the file.

The actual `rwx` permission bits still determine what the group can do.

------------------------------------------------------------------------

## 3. `umask`

I checked my current shell's umask:

``` bash
umask
```

For `mo`:

``` text
0002
```

I then checked the shell used for `ali`:

``` bash
sudo -u ali sh -c 'umask'
```

For `ali`:

``` text
0022
```

### What `umask` does

`umask` is a **file-creation permission mask**. It removes permission
bits from the mode requested when a new file or directory is created.

A common requested mode for normal files is:

``` text
666
rw-rw-rw-
```

Normal files are not automatically created executable.

With:

``` text
umask 0022
```

group write and others write are masked out:

``` text
Requested:  rw-rw-rw-   666
Mask:          -w- -w-   022
Result:     rw-r--r--   644
```

This explained why the file created through `ali` became `644`.

For directories, the common requested mode is:

``` text
777
rwxrwxrwx
```

With `umask 0022`, the resulting directory mode is:

``` text
755
rwxr-xr-x
```

### Four-digit notation

For:

``` text
0022
```

the positions can be viewed as:

``` text
special | owner | group | others
   0        0       2       2
```

The leading position is associated with special permission bits:

-   `4` --- setuid.
-   `2` --- setgid.
-   `1` --- sticky bit.

Important distinction:

> `umask` removes permissions during object creation. It is not the same
> thing as an ACL mask.

------------------------------------------------------------------------

## 4. Setuid (SUID)

I inspected the `passwd` executable:

``` bash
ls -l /usr/bin/passwd
```

Output:

``` text
-rwsr-xr-x root root /usr/bin/passwd
```

The `s` in the **owner execute position** indicates **SUID --- Set User
ID**.

### What SUID does

When an executable has SUID enabled, the process can run with the
**effective user identity of the file owner**.

For `/usr/bin/passwd`:

-   File owner: `root`.
-   User launching it: `mo`.
-   Effective identity while performing privileged operations: `root`.

This does **not** turn `mo` into root. The specific program receives the
elevated effective identity and its code controls what operations are
allowed.

### Real UID vs Effective UID

I started `passwd` and inspected the process from another terminal:

``` bash
ps -o pid,ruid,euid,comm -C passwd
```

### Command meaning

-   `ps` --- inspect running processes.
-   `-o` --- choose output columns.
-   `pid` --- process ID.
-   `ruid` --- real user ID.
-   `euid` --- effective user ID.
-   `comm` --- command name.
-   `-C passwd` --- select processes named `passwd`.

The important result was:

``` text
RUID = 1000
EUID = 0
```

Meaning:

-   `RUID 1000` --- `mo` launched the process.
-   `EUID 0` --- the process is operating with root's effective
    identity.

A useful mental model:

> RUID tells me who the process came from. EUID tells me the effective
> user identity Linux generally uses for permission checks.

------------------------------------------------------------------------

## 5. Finding SUID Executables

I searched `/usr/bin` for SUID files:

``` bash
find /usr/bin -type f -perm -4000 -ls
```

### Command meaning

-   `find /usr/bin` --- search inside `/usr/bin`.
-   `-type f` --- only regular files.
-   `-perm -4000` --- match files with the SUID bit set.
-   `-ls` --- display detailed information.

The `-` before `4000` means the requested bit must be present; the
complete file mode does not have to equal exactly `4000`.

For example:

``` text
4755
-rwsr-xr-x
```

matches because the SUID bit is present.

Some SUID-root programs found on the VM included:

``` text
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/chfn
/usr/bin/umount
/usr/bin/pkexec
/usr/bin/chsh
/usr/bin/su
/usr/bin/mount
```

### Security reasoning

Known SUID programs such as `passwd` have a legitimate reason to perform
controlled privileged operations.

However, an unfamiliar custom executable such as:

``` text
-rwsr-xr-x root root /usr/local/bin/company-backup
```

would deserve investigation.

Questions I should ask include:

-   Why does this program need SUID?
-   Why does it need an effective UID of `0`?
-   Who installed or owns the program?
-   What privileged resources does it access?
-   Is SUID actually necessary?
-   Can an unprivileged user manipulate the program into performing
    unintended privileged operations?

This connected Linux file permissions with **process identity and
privilege-escalation risk**.

------------------------------------------------------------------------

## 6. SGID Executables

To search for SGID executables:

``` bash
find /usr/bin -type f -perm -2000 -ls
```

The VM returned no matches in `/usr/bin`.

That is still a valid result: a command producing no output can simply
mean **no matching files were found**.

The special-bit mapping is:

``` text
4000 -> SUID
2000 -> SGID
1000 -> Sticky bit
```

For executables:

-   SUID affects the **effective user identity**.
-   SGID can affect the **effective group identity**.

For directories, SGID instead provides **group inheritance** for newly
created objects.

------------------------------------------------------------------------

## 7. Access Control Lists (ACLs)

Traditional Linux permissions provide only:

``` text
owner | group | others
```

ACLs allow more precise exceptions.

For example, I created:

``` text
-rw------- mo mo acl-test.txt
```

Normal mode `600` meant only `mo` had access.

I then gave only `ali` read access without granting access to everyone
else:

``` bash
setfacl -m u:ali:r /home/mo/permissions-lab/acl-test.txt
```

### Command meaning

-   `setfacl` --- set or modify an Access Control List.
-   `-m` --- modify the ACL.
-   `u` --- user ACL entry.
-   `ali` --- named user.
-   `r` --- read permission.

After adding an extended ACL, `ls -l` showed a `+`:

``` text
-rw-r-----+
          ^
```

The `+` indicates that extended ACL information exists.

To inspect it:

``` bash
getfacl /home/mo/permissions-lab/acl-test.txt
```

Example:

``` text
user::rw-
user:ali:r--
group::---
mask::r--
other::---
```

Meaning:

-   `user::rw-` --- file owner `mo` has read/write.
-   `user:ali:r--` --- named user `ali` has a read ACL entry.
-   `group::---` --- owning group gets no permission from its normal
    entry.
-   `mask::r--` --- ACL/group-class maximum effective permissions are
    read-only.
-   `other::---` --- everyone else has no access.

I confirmed that `ali` could read the file despite the original
traditional permissions being `600`.

------------------------------------------------------------------------

## 8. ACL Mask

The ACL mask acts as a **ceiling on effective permissions** for
named-user ACL entries and group-class entries.

Example:

``` text
user:ali:rw-
mask::r--
```

The ACL entry requests:

``` text
rw-
```

but the mask permits only:

``` text
r--
```

Therefore Ali's effective permission is:

``` text
r--
```

He can read but cannot write.

When I initially changed Ali's ACL to read/write:

``` bash
setfacl -m u:ali:rw /home/mo/permissions-lab/acl-test.txt
```

`setfacl` recalculated the mask and it became:

``` text
mask::rw-
```

I then manually limited the mask:

``` bash
setfacl -m m:r /home/mo/permissions-lab/acl-test.txt
```

### Command meaning

-   `setfacl` --- change ACL information.
-   `-m` --- modify.
-   `m:r` --- set the ACL mask to read-only.

This left Ali's ACL entry as `rw-`, but his effective access became
read-only.

### `umask` vs ACL mask

These are different concepts:

**`umask`**

> Removes permissions when a new file or directory is created.

**ACL mask**

> Limits the effective permissions available through named-user and
> group-class ACL entries.

------------------------------------------------------------------------

## 9. Testing Permissions Correctly with `sudo`

I discovered an important shell behavior while testing ACL write access.

I first ran:

``` bash
sudo -u ali echo "add this" >> /home/mo/permissions-lab/acl-test.txt
```

The write unexpectedly succeeded.

The reason was that `sudo` applied to `echo`, but the `>>` redirection
was handled by my existing shell as `mo`.

Conceptually:

``` text
shell running as mo
    |
    +-- opens acl-test.txt for append as mo
    |
    +-- runs echo as ali
```

Therefore this was **not a valid test of Ali's write permission**.

The correct command was:

``` bash
sudo -u ali sh -c 'echo "ali added this" >> /home/mo/permissions-lab/acl-test.txt'
```

### Command meaning

-   `sudo -u ali` --- run as user `ali`.
-   `sh` --- start a shell.
-   `-c` --- execute the supplied command string.
-   `>>` --- append output to the file.

Because the redirection occurs inside the shell running as `ali`, Linux
correctly returned:

``` text
Permission denied
```

This confirmed that the ACL mask was limiting Ali to read-only access.

------------------------------------------------------------------------

## 10. Shell Troubleshooting Learned Along the Way

### Lowercase `-c` matters

I accidentally used:

``` bash
sh -C
```

instead of:

``` bash
sh -c
```

For this use case, lowercase `-c` tells the shell to execute the
following command string.

### Continuation prompt `>`

I also entered an incomplete quoted command and Bash displayed:

``` text
>
```

This means the shell believes the command is incomplete and is waiting
for more input, for example because a quote was not closed.

I used:

``` text
Ctrl+C
```

to cancel the unfinished command and return to the normal prompt.

------------------------------------------------------------------------

## Key Concepts I Can Now Explain

### Traditional permissions

``` text
owner | group | others
```

### Sticky bit

Protects directory entries in shared writable directories such as
`/tmp`, restricting who may delete or rename entries.

### Setgid on a directory

Makes newly created files/directories inherit the parent directory's
group ownership.

### Setuid on an executable

Allows a program to operate with the effective user identity of the
executable's owner.

### SGID on an executable

Can give the process the effective group identity associated with the
executable.

### `umask`

Controls which permission bits are masked out when new files/directories
are created.

### ACL

Allows permissions to be assigned to specific additional users or groups
without changing the traditional owner/group/others model.

### ACL mask

Limits the maximum effective permissions available to named-user and
group-class ACL entries.

### RUID / EUID

-   **RUID** --- real identity of the user who started the process.
-   **EUID** --- effective user identity used for privileged operations
    and permission checks.

------------------------------------------------------------------------

## Commands Practiced

``` bash
ls -ld /tmp
ls -l /tmp/mo-sticky-test.txt
chmod u=rw,g=,o= /tmp/mo-sticky-test.txt

ls -ld /home/mo/permissions-lab
sudo -u ali touch /home/mo/permissions-lab/ali-created.txt
chmod g+s /home/mo/permissions-lab

umask
sudo -u ali sh -c 'umask'

ls -l /usr/bin/passwd
ls -l /etc/shadow
ps -o pid,ruid,euid,comm -C passwd

find /usr/bin -type f -perm -4000 -ls
find /usr/bin -type f -perm -2000 -ls
man chsh

getfacl --version
getfacl /home/mo/permissions-lab/acl-test.txt

setfacl -m u:ali:r /home/mo/permissions-lab/acl-test.txt
setfacl -m u:ali:rw /home/mo/permissions-lab/acl-test.txt
setfacl -m m:r /home/mo/permissions-lab/acl-test.txt

sudo -u ali cat /home/mo/permissions-lab/acl-test.txt
sudo -u ali sh -c 'echo "ali added this" >> /home/mo/permissions-lab/acl-test.txt'
```

------------------------------------------------------------------------

## Security Takeaway

The biggest lesson from this session is that Linux access control is not
just about reading a permission string.

I now need to think about several layers:

``` text
Who owns the object?
        |
Which group owns it?
        |
What do the normal rwx bits allow?
        |
Are special bits enabled?
        |
Is there an extended ACL?
        |
What does the ACL mask permit?
        |
Which user/group identity is the process actually using?
```

Understanding those layers is important for both **Linux system
administration** and **Linux/infrastructure security**, especially when
troubleshooting permission problems or investigating unexpected
privileged behavior.
