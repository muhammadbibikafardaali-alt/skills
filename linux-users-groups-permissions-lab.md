# Linux Users, Groups, Ownership & Permissions --- Hands-On Lab

## Session Goal

This lab focused on understanding how Linux decides **who can access a
file or directory and what they are allowed to do**.

Rather than only memorizing permission numbers such as `644`, `640`, or
`700`, I practiced reading permissions, changing them, testing them as
another user, and troubleshooting `Permission denied` errors.

------------------------------------------------------------------------

## 1. Inspecting My Linux Identity

Commands used:

``` bash
whoami
id
```

### Plain-English meaning

-   `whoami` --- shows the username of the current effective user.
-   `id` --- shows the user's UID, primary GID, and group memberships.

Example from the lab:

``` text
uid=1000(mo) gid=1000(mo) groups=1000(mo),...,27(sudo),...
```

I learned the distinction between:

-   **Username** --- the human-readable account name, such as `mo`.
-   **UID** --- the numeric user identifier Linux uses internally.
-   **Primary GID** --- the user's primary group ID.
-   **Supplementary groups** --- additional groups that can grant access
    to resources.

For example, membership in Ubuntu's `sudo` group can grant
administrative access according to the system's sudo policy.

------------------------------------------------------------------------

## 2. Reading File Ownership and Permissions

I inspected SSH files with:

``` bash
ls -l /home/mo/.ssh/authorized_keys
```

Example:

``` text
-rw------- 1 mo mo 95 ... /home/mo/.ssh/authorized_keys
```

The important fields are:

``` text
-rw-------  mo  mo
│           │   │
│           │   └── group owner
│           └────── user owner
└────────────────── file type + permissions
```

A Linux file has a **user owner** and a **group owner**. The group is
not a second user owner.

Permission classes are:

``` text
user/owner | group | others
```

For:

``` text
-rw-------
```

the meaning is:

-   `-` --- regular file
-   `rw-` --- owner can read and write
-   `---` --- group has no permissions
-   `---` --- others have no permissions

------------------------------------------------------------------------

## 3. File Type: Regular Files vs Directories

I compared files with the `.ssh` directory:

``` bash
ls -ld /home/mo/.ssh
```

Example:

``` text
drwx------ 2 mo mo ... /home/mo/.ssh/
```

The first character identifies the object type:

``` text
-  regular file
d  directory
```

### Important: `x` means something different on directories

For a regular file:

-   `r` --- read the contents
-   `w` --- modify the contents
-   `x` --- execute the file

For a directory:

-   `r` --- list names inside the directory
-   `w` --- create, delete, or rename directory entries
-   `x` --- traverse the directory / access entries through it

This distinction became important later when troubleshooting access
through `/home/mo`.

------------------------------------------------------------------------

## 4. Numeric Permission Notation

Linux permissions can also be represented numerically:

``` text
r = 4
w = 2
x = 1
```

Each permission class is calculated independently.

Examples:

``` text
rw- = 4 + 2 = 6
r-- = 4
--- = 0
rwx = 4 + 2 + 1 = 7
```

Therefore:

``` text
-rw-r----- = 640
-rw-r--r-- = 644
-rw------- = 600
drwx------ = 700
```

A mistake I corrected during the lab was calculating the entire
permission string together. The correct method is to calculate **owner,
group, and others separately**.

------------------------------------------------------------------------

## 5. Creating a Permissions Lab

I created a safe practice directory:

``` bash
mkdir /home/mo/permissions-lab
```

Plain English:

> Create a directory named `permissions-lab`.

I inspected it using:

``` bash
ls -ld /home/mo/permissions-lab
```

The `-l` flag requests long-format information.\
The `-d` flag tells `ls` to show the directory entry itself instead of
listing its contents.

I then created an empty test file:

``` bash
touch /home/mo/permissions-lab/test.txt
```

Plain English:

> Create an empty file named `test.txt` if it does not already exist.

------------------------------------------------------------------------

## 6. Changing Permissions with `chmod`

I practiced symbolic permission changes.

Example:

``` bash
chmod u-w /home/mo/permissions-lab/test.txt
```

Meaning:

``` text
u  → user/owner
-  → remove
w  → write permission
```

Plain English:

> Remove write permission from the file owner.

The symbolic classes are:

``` text
u = user/owner
g = group
o = others
a = all classes
```

Operations include:

``` text
+ = add permissions
- = remove permissions
= = set permissions exactly
```

I also practiced setting exact symbolic permissions:

``` bash
chmod u=rw,g=r,o= /home/mo/permissions-lab/test.txt
```

Plain English:

> Give the owner read/write, the group read-only, and others no
> permissions.

This produces:

``` text
-rw-r-----
```

The numeric equivalent is:

``` bash
chmod 640 /home/mo/permissions-lab/test.txt
```

Symbolic notation is useful because the intended policy is very explicit
and can help avoid human mistakes.

------------------------------------------------------------------------

## 7. Creating a Second User

To test permissions properly, I created another user:

``` bash
sudo adduser ali
```

Plain English:

> Run the user-creation tool with administrative privileges and create
> the account `ali`.

I inspected the new identity:

``` bash
id ali
```

Example:

``` text
uid=1004(ali) gid=1004(ali) groups=1004(ali),100(users)
```

At this point, `ali` was not the owner of my test file and was not a
member of its group `mo`.

For a file owned by:

``` text
mo:mo
```

with:

``` text
-rw-r-----
```

Linux would therefore place `ali` in the **others** permission class for
that file.

------------------------------------------------------------------------

## 8. Testing Access as Another User

I tested file access without logging out:

``` bash
sudo -u ali cat /home/mo/permissions-lab/test.txt
```

### Flags and commands

-   `sudo` --- run a command under another security context.
-   `-u ali` --- run specifically as user `ali`.
-   `cat` --- read file contents and write them to standard output.

The first test returned:

``` text
Permission denied
```

Initially it looked like the file's `---` permissions for others were
responsible. Instead of assuming, I investigated the complete pathname.

------------------------------------------------------------------------

## 9. Troubleshooting with `namei -l`

I used:

``` bash
namei -l /home/mo/permissions-lab/test.txt
```

Plain English:

> Break the pathname into components and show the permissions and
> ownership at each level.

`-l` requests long-format permission/ownership information.

The path showed:

``` text
drwxr-xr-x root root /
drwxr-xr-x root root home
drwxr-x--- mo   mo   mo
drwxrwxr-x mo   mo   permissions-lab
-rw-r----- mo   mo   test.txt
```

The real blocker was:

``` text
/home/mo → drwxr-x--- mo mo
```

Because `ali` was neither owner `mo` nor a member of group `mo`, he
received the `others` permissions:

``` text
---
```

He therefore lacked `x` on `/home/mo` and could not traverse the
directory.

### Key troubleshooting lesson

To reach a file through a pathname, the user needs appropriate traversal
access on **every directory in the path**.

A `Permission denied` error does not automatically mean the final file
is the problem.

------------------------------------------------------------------------

## 10. Using `/tmp` to Isolate the File Permission

To test the file independently of `/home/mo`, I copied it to `/tmp`:

``` bash
cp /home/mo/permissions-lab/test.txt /tmp/permission-test.txt
```

Plain English:

> Copy the source file into `/tmp` under a new name.

I checked `/tmp` with:

``` bash
namei -l /tmp/
```

and saw:

``` text
drwxrwxrwt root root tmp
```

`/tmp` is a standard location used for temporary files. It is normally
writable by multiple users, and files there should not be treated as
permanent storage.

The final `t` is the **sticky bit**. I noticed it during this session,
but sticky-bit behavior is a topic for a later lab.

------------------------------------------------------------------------

## 11. Granting Access Through Group Membership

Instead of making the file readable by everyone, I granted `ali` access
through the existing `mo` group:

``` bash
sudo usermod -aG mo ali
```

### Flags

-   `usermod` --- modify an existing user account.
-   `-G mo` --- specify supplementary group `mo`.
-   `-a` --- append the group instead of replacing the existing
    supplementary group list.

The `-a` is important when using `-G`; omitting it can replace existing
supplementary group memberships.

I verified the change:

``` bash
id ali
```

Result:

``` text
uid=1004(ali) gid=1004(ali) groups=1004(ali),100(users),1000(mo)
```

`ali`'s **primary group remained `ali`**, while `mo` became a
**supplementary group**.

After this change, `ali` could traverse `/home/mo` using its group `r-x`
permissions and read `test.txt` using the file's group `r--` permission.

I added visible content to the file and confirmed:

``` bash
sudo -u ali cat /home/mo/permissions-lab/test.txt
```

Output:

``` text
Hello Chat, we're in!
```

This demonstrated group-based access control in practice.

------------------------------------------------------------------------

## 12. Testing Read Permission vs Write Permission

The file remained:

``` text
-rw-r----- mo mo test.txt
```

Because `ali` belongs to group `mo`, he receives:

``` text
r--
```

Therefore he can read the file but should not be able to modify its
contents.

I tested an append operation as `ali`:

``` bash
sudo -u ali sh -c 'echo "Ali was here" >> /home/mo/permissions-lab/test.txt'
```

### Why `sh -c`?

-   `sh` --- starts a shell.
-   `-c` --- tells the shell to execute the following command string.
-   `>>` --- shell redirection that opens the destination for appending.

Running the entire expression through `sudo -u ali sh -c` ensures the
redirection itself is attempted as `ali`, making the permissions test
valid.

The result was:

``` text
cannot create ... Permission denied
```

This confirmed that the group's `r--` allows reading but not writing.

------------------------------------------------------------------------

## 13. File Permissions vs Directory Permissions When Deleting

This was one of the most important experiments of the session.

The file:

``` text
-rw-r----- mo mo delete-me.txt
```

was not writable by `ali`.

However, the parent directory was:

``` text
drwxrwxr-x mo mo permissions-lab
```

Because `ali` belongs to group `mo`, he has `rwx` on the directory.

I tested:

``` bash
sudo -u ali rm /home/mo/permissions-lab/delete-me.txt
```

`rm` warned:

``` text
remove write-protected regular file ...?
```

After confirming, the deletion succeeded.

### Why?

Modifying a file's **contents** and removing its **directory entry** are
different operations.

A useful mental model is:

``` text
Modify file contents
        ↓
Check file write permission

Delete/rename directory entry
        ↓
Check permissions on the parent directory
```

Therefore, a user can potentially be unable to write to a file but still
be able to delete it if they have the required permissions on its parent
directory.

The prompt from `rm` was a warning from the `rm` program; it was not the
kernel denying the operation.

------------------------------------------------------------------------

## 14. Error Messages I Learned to Distinguish

During the lab I encountered:

``` text
Permission denied
```

This indicates that an object/path may exist, but the attempted access
is not permitted.

I also renamed `test.text` to `test.txt` using:

``` bash
mv /home/mo/permissions-lab/test.text /home/mo/permissions-lab/test.txt
```

When source and destination are in the same directory, `mv` effectively
performs a rename.

Trying the old filename produced:

``` text
No such file or directory
```

That is a different problem:

``` text
Permission denied
→ access was blocked

No such file or directory
→ the requested pathname did not resolve to that file
```

Being able to distinguish these errors is part of effective Linux
troubleshooting.

------------------------------------------------------------------------

## Commands and Flags Practiced

  -----------------------------------------------------------------------
  Command                             Purpose
  ----------------------------------- -----------------------------------
  `whoami`                            Show the current effective username

  `id`                                Show UID, GID, and group
                                      memberships

  `id USER`                           Inspect another user's identity and
                                      groups

  `ls -l`                             Long-format listing with
                                      permissions and ownership

  `ls -ld DIR`                        Show information about the
                                      directory itself

  `mkdir DIR`                         Create a directory

  `touch FILE`                        Create an empty file if it does not
                                      exist

  `chmod`                             Change permission bits

  `chmod u-w FILE`                    Remove owner write permission

  `chmod u=rw,g=r,o= FILE`            Set exact symbolic permissions

  `chmod 640 FILE`                    Set the equivalent numeric
                                      permissions

  `sudo adduser USER`                 Create a user using Debian/Ubuntu's
                                      `adduser` tool

  `sudo -u USER COMMAND`              Execute a command as another user

  `namei -l PATH`                     Inspect every component of a
                                      pathname with ownership/permissions

  `cp SOURCE DEST`                    Copy a file

  `usermod -aG GROUP USER`            Append a user to a supplementary
                                      group

  `cat FILE`                          Print file contents

  `sh -c 'COMMAND'`                   Ask a shell to execute a command
                                      string

  `>> FILE`                           Append command output to a file

  `mv SOURCE DEST`                    Move or rename a file

  `rm FILE`                           Remove a file/directory entry
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Key Takeaways

By the end of this session, I could:

-   Read Linux file and directory permission strings.
-   Identify user ownership and group ownership.
-   Explain UID, GID, primary groups, and supplementary groups.
-   Convert between symbolic permissions and numeric permissions such as
    `640`.
-   Change permissions using both numeric and symbolic `chmod`.
-   Explain how `r`, `w`, and `x` differ between regular files and
    directories.
-   Create a test user and evaluate which permission class Linux
    applies.
-   Grant access through group membership instead of opening access to
    everyone.
-   Test commands under another user's identity with `sudo -u`.
-   Trace permission failures through a pathname using `namei -l`.
-   Distinguish file-content permissions from parent-directory
    permissions.
-   Explain why a user can sometimes delete a file they cannot modify.
-   Distinguish `Permission denied` from `No such file or directory`.
-   Verify assumptions experimentally instead of guessing where an
    access failure occurs.

## Sysadmin Mindset Practiced

The most valuable part of this lab was not memorizing `chmod`.

The workflow was:

``` text
Observe the error
      ↓
Inspect identity
      ↓
Inspect ownership
      ↓
Inspect permissions
      ↓
Trace the complete path
      ↓
Predict expected behavior
      ↓
Test as the affected user
      ↓
Verify the result
```

That is the troubleshooting approach I want to continue building in
future Linux administration labs.
