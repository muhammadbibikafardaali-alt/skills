# Linux Server Administration & SSH Hardening Lab

## Overview

This lab documents the practical Linux administration and SSH security
skills I developed while configuring and troubleshooting an Ubuntu
server in a virtual machine.

Rather than only configuring services, I focused on understanding how
the components interact: Linux processes, `systemd`, network sockets,
UFW firewall rules, Nginx, SSH authentication, SSH keys, `ssh-agent`,
and OpenSSH server hardening.

A major part of the lab was verification. After making configuration
changes, I checked the effective configuration and tested both
successful and intentionally blocked authentication paths.

> **Security note:** Network addresses in this document use placeholders
> such as `<SERVER_IP>` so the write-up can be published publicly
> without exposing unnecessary environment details.

## Environment

-   Ubuntu Linux virtual machine
-   macOS SSH client
-   OpenSSH client/server
-   `systemd`
-   UFW firewall
-   Nginx web server
-   QEMU-based virtualized lab environment

## Skills Demonstrated

Through this lab, I demonstrated the ability to:

-   Inspect and manage Linux services with `systemctl`
-   Understand Linux PIDs, PPIDs, master processes, and worker processes
-   Identify listening TCP ports and their associated processes
-   Configure and verify UFW firewall rules
-   Troubleshoot Nginx connectivity
-   Understand SSH socket activation and the relationship between
    `ssh.socket` and `ssh.service`
-   Establish remote SSH access from macOS to Ubuntu
-   Understand public/private SSH key authentication
-   Generate and install ED25519 SSH keys
-   Understand SSH key fingerprints
-   Use `ssh-agent` to temporarily cache unlocked private-key identities
-   Inspect and manage identities loaded into `ssh-agent`
-   Inspect OpenSSH server configuration
-   Distinguish `ssh` from `sshd`
-   Use OpenSSH drop-in configuration files
-   Validate SSH configuration before applying changes
-   Disable direct root SSH login
-   Disable SSH password authentication
-   Verify that public-key authentication continues to work after
    hardening
-   Deliberately test that password-based SSH access is rejected

## Linux Services and Processes

I started by inspecting Nginx with:

``` bash
systemctl status nginx
```

This showed whether the unit was loaded and active, its main process ID,
worker processes, and recent service logs.

An important concept from this section was the difference between a
**PID** and a **PPID**:

-   **PID (Process ID):** unique identifier assigned to a running
    process.
-   **PPID (Parent Process ID):** PID of the process that created or
    manages another process.

For Nginx, I observed a master/worker process model:

``` text
systemd (PID 1)
    |
    +-- nginx master process
            |
            +-- nginx worker
            +-- nginx worker
            +-- nginx worker
            +-- nginx worker
```

I inspected processes with:

``` bash
ps -p <PID>
ps -P <PID>
```

This connected the service information reported by `systemctl` with the
actual Linux process hierarchy.

## Understanding systemd

I learned that `systemd` is Ubuntu's system and service manager. PID 1
is normally `systemd`, which manages many of the services running on the
system.

Commands I used included:

``` bash
systemctl status nginx
systemctl status ssh
systemctl status ssh.socket
```

I also refreshed systemd's unit definitions when required:

``` bash
sudo systemctl daemon-reload
```

An important distinction is:

-   `systemctl daemon-reload` makes **systemd re-read unit
    definitions**.
-   `systemctl reload ssh` asks the **SSH service to re-read its own
    configuration**.

These are different operations.

## Inspecting Listening Network Ports

I inspected listening TCP sockets with:

``` bash
sudo ss -ltn
```

I identified services listening on ports including TCP 22 for SSH and
TCP 80 for HTTP/Nginx, as well as local services using ports 53 and 631.

To include process information, I used:

``` bash
sudo ss -ltnp
```

I filtered the output when I only wanted specific ports:

``` bash
sudo ss -ltnp | grep -E ':22 |:80 '
```

This allowed me to connect a network port to the process that owned the
listening socket.

``` text
Application / Service
        |
      Process
        |
      Socket
        |
     TCP Port
        |
      Network
```

## UFW Firewall Configuration

I inspected the firewall with:

``` bash
sudo ufw status
```

The firewall used a default policy that denied incoming connections
unless explicitly permitted.

I allowed HTTP traffic with:

``` bash
sudo ufw allow 80/tcp
```

I also configured SSH access on TCP port 22 and verified the resulting
rules.

This demonstrated an important distinction: **a service listening on a
port does not automatically mean network traffic can reach it.** The
application must be listening and the firewall must permit the traffic.

## SSH Service and Socket Activation

While investigating SSH, I observed that `ssh.service` could appear
inactive while:

``` bash
systemctl status ssh.socket
```

showed the SSH socket as active and listening.

This introduced me to **systemd socket activation**. The socket can
listen on port 22 and trigger the SSH service when a connection arrives.

I later confirmed SSH listening on:

``` text
0.0.0.0:22
[::]:22
```

representing IPv4 and IPv6 listening addresses.

## Establishing an SSH Connection

From my Mac, I connected to the Ubuntu server with:

``` bash
ssh mo@<SERVER_IP>
```

On the first connection, SSH displayed the server's host-key fingerprint
and asked whether I trusted the host. After accepting it, the host key
was stored in the client's `known_hosts` file.

I also used verbose mode while troubleshooting:

``` bash
ssh -v mo@<SERVER_IP>
```

The verbose output helped me observe connection establishment, host-key
verification, key exchange, authentication methods, public-key attempts,
password fallback, and successful authentication.

## Public and Private SSH Keys

SSH key authentication uses a **key pair**.

### Private key

The private key stays on the client machine and must be protected. It
should **never be copied to the server or published in a Git
repository**.

It is used to prove possession of the key without transmitting the
private key itself.

### Public key

The public key can be installed on servers that I want to access. For a
Linux account, authorized public keys are commonly stored in:

``` text
~/.ssh/authorized_keys
```

The server uses the public key to verify a cryptographic proof created
with the corresponding private key.

``` text
Mac / SSH Client                         Ubuntu / SSH Server

Private key                              Public key
~/.ssh/id_ed25519                        ~/.ssh/authorized_keys
       |                                         |
       | signs authentication challenge          |
       +-----------------------> server verifies +
```

## Generating an ED25519 SSH Key

I generated an ED25519 key pair and protected the private key with a
passphrase.

The generated files followed the standard pattern:

``` text
~/.ssh/id_ed25519
~/.ssh/id_ed25519.pub
```

`id_ed25519` is the private key and `id_ed25519.pub` is the public key.

## SSH Key Fingerprints

I learned the difference between an SSH key and its fingerprint.

A **key** contains cryptographic material used for authentication. A
**fingerprint** is a short hash-based identifier derived from a key.

``` text
SSH key
   |
   +-- hash function
           |
           +-- fingerprint
```

The fingerprint is useful for identifying and comparing keys without
displaying the complete key.

## Installing a Public Key on the Server

From the Mac client, I installed my public key on the Ubuntu account
using:

``` bash
ssh-copy-id mo@<SERVER_IP>
```

Afterward, SSH could authenticate using the key pair.

I verified server-side permissions:

``` bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

The permissions were appropriately restrictive:

``` text
drwx------  ~/.ssh
-rw-------  ~/.ssh/authorized_keys
```

These correspond to `700` for `~/.ssh` and `600` for `authorized_keys`.

## Understanding SSH Key Comments

An SSH public key can end with a comment such as:

``` text
mo@ubuntu-lab
```

I learned that this text is only a **label/comment associated with the
key**. It does not determine ownership, permissions, root access, or
authentication privileges.

Authorization comes from where the public key is installed and the SSH
server's configuration.

## Using ssh-agent

Because my private key was protected by a passphrase, I explored
`ssh-agent`.

I inspected loaded identities with:

``` bash
ssh-add -l
```

I loaded my private key with:

``` bash
ssh-add ~/.ssh/id_ed25519
```

and verified the loaded identity again with:

``` bash
ssh-add -l
```

Once loaded, the agent could perform signing operations for the SSH
client without requiring the key passphrase for every connection.

I removed the identity from the agent with:

``` bash
ssh-add -d ~/.ssh/id_ed25519
```

This removed the identity from **the agent only**; it did not delete the
private-key file from disk.

## Inspecting the OpenSSH Server Configuration

I inspected the effective SSH server configuration with:

``` bash
sudo sshd -T
```

An important terminology distinction is:

-   `ssh` = SSH **client**
-   `sshd` = SSH **server daemon**

The `-T` option prints the effective SSH server configuration.

Initially, important settings included:

``` text
permitrootlogin prohibit-password
pubkeyauthentication yes
passwordauthentication yes
```

This meant public-key authentication and password authentication were
enabled, while root could not authenticate using a password but could
potentially use another permitted method.

## Effective Configuration vs. Configuration Files

The main configuration contained commented examples:

``` text
#PermitRootLogin prohibit-password
#PubkeyAuthentication yes
#PasswordAuthentication yes
```

Because these lines begin with `#`, they are comments rather than active
directives.

I also found:

``` text
Include /etc/ssh/sshd_config.d/*.conf
```

This tells OpenSSH to load matching `.conf` files from
`/etc/ssh/sshd_config.d/`.

I used this drop-in mechanism instead of modifying the main
configuration file directly.

## Troubleshooting a Drop-In Configuration File

I initially created:

``` text
99-hardening.config
```

but the include pattern was:

``` text
*.conf
```

Because `.config` did not match `.conf`, OpenSSH ignored the file.

I corrected the filename:

``` bash
sudo mv /etc/ssh/sshd_config.d/99-hardening.config /etc/ssh/sshd_config.d/99-hardening.conf
```

After renaming it, `sshd -T` showed the expected effective setting.

This reinforced an important troubleshooting lesson: **correct
configuration content is not enough; the service must actually load the
file.**

## SSH Hardening

I used:

``` text
/etc/ssh/sshd_config.d/99-hardening.conf
```

with the important security directives:

``` text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
```

### `PermitRootLogin no`

Disables direct SSH authentication as `root`. This does not remove the
root account or change root's local privileges.

### `PubkeyAuthentication yes`

Keeps public-key authentication enabled.

### `PasswordAuthentication no`

Disables password authentication over SSH. It does not delete the Linux
user's password; that password can still be used for other purposes such
as `sudo`, depending on local configuration.

## Validating SSH Configuration Safely

Before applying changes, I validated the SSH server configuration:

``` bash
sudo sshd -t
```

A successful syntax test produced no output.

I learned the important distinction:

``` bash
sudo sshd -t
sudo sshd -T
```

-   `-t` --- test the configuration for validity
-   `-T` --- print the effective configuration

I filtered the effective configuration with:

``` bash
sudo sshd -T | grep -E '^(permitrootlogin|pubkeyauthentication|passwordauthentication) '
```

The hardened state was:

``` text
permitrootlogin no
pubkeyauthentication yes
passwordauthentication no
```

## grep and Case Sensitivity

I initially searched for:

``` bash
sudo sshd -T | grep '^PasswordAuthentication'
```

and received no match because `sshd -T` displayed the directive in
lowercase:

``` text
passwordauthentication no
```

`grep` is case-sensitive by default.

The lowercase search worked:

``` bash
sudo sshd -T | grep '^passwordauthentication'
```

or I could explicitly ignore case:

``` bash
sudo sshd -T | grep -i '^passwordauthentication'
```

## Reloading SSH Safely

After validation, I reloaded SSH:

``` bash
sudo systemctl reload ssh
```

During remote SSH configuration changes, I kept an existing
authenticated session open while testing a second connection.

``` text
Edit configuration
       |
       v
Validate with sshd -t
       |
       v
Inspect with sshd -T
       |
       v
Reload SSH
       |
       v
Keep existing session open
       |
       v
Test a second connection
```

This reduces the risk of locking myself out after a configuration
mistake.

## Verifying Key Authentication

I explicitly tested key authentication while disabling password
authentication for that client connection:

``` bash
ssh -o PasswordAuthentication=no mo@<SERVER_IP>
```

The connection succeeded, demonstrating that public-key authentication
worked independently of password authentication.

## Verifying Password Authentication Was Blocked

After disabling password authentication on the server, I deliberately
disabled public-key authentication on the client:

``` bash
ssh -o PubkeyAuthentication=no mo@<SERVER_IP>
```

The server rejected the connection with:

``` text
Permission denied (publickey).
```

This was the expected result.

``` text
Authorized SSH key     -> SUCCESS
SSH password fallback  -> BLOCKED
Direct root SSH login  -> DISABLED
```

## Final SSH Security State

At the end of the lab:

``` text
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
```

The resulting authentication model was:

``` text
Remote Client
     |
     v
TCP/22
     |
     v
UFW
     |
     v
OpenSSH Server
     |
     +-- Direct root login --------> DENIED
     |
     +-- Password authentication --> DENIED
     |
     +-- Authorized public key ----> ALLOWED
```

# Command Reference

  ----------------------------------------------------------------------------
  Command                                  Purpose
  ---------------------------------------- -----------------------------------
  `systemctl status nginx`                 Inspect Nginx service state

  `systemctl status ssh`                   Inspect SSH service state

  `systemctl status ssh.socket`            Inspect SSH socket activation

  `sudo systemctl daemon-reload`           Reload systemd unit definitions

  `sudo systemctl reload ssh`              Reload SSH server configuration

  `ps -p <PID>`                            Select a process by PID

  `ps -P <PPID>`                           Select processes by parent PID

  `sudo ss -ltn`                           Display listening TCP sockets

  `sudo ss -ltnp`                          Display listening TCP sockets with
                                           process information

  `sudo ufw status`                        Display firewall status and rules

  `sudo ufw allow 80/tcp`                  Permit inbound HTTP traffic

  `ssh mo@<SERVER_IP>`                     Connect to the server using SSH

  `ssh -v mo@<SERVER_IP>`                  Connect with verbose SSH debugging

  `ssh-copy-id mo@<SERVER_IP>`             Install the client's public key on
                                           the server

  `ssh-add -l`                             List identities loaded in
                                           `ssh-agent`

  `ssh-add ~/.ssh/id_ed25519`              Load a private key into `ssh-agent`

  `ssh-add -d ~/.ssh/id_ed25519`           Remove an identity from `ssh-agent`

  `sudo sshd -t`                           Validate SSH server configuration

  `sudo sshd -T`                           Print effective SSH server
                                           configuration

  `sudo nvim <file>`                       Edit a protected configuration file

  `sudo mv <source> <destination>`         Move or rename a protected file

  `ls -ld ~/.ssh`                          Inspect `.ssh` directory
                                           permissions

  `ls -l ~/.ssh/authorized_keys`           Inspect `authorized_keys`
                                           permissions

  `ssh -o PasswordAuthentication=no ...`   Disable password auth for one
                                           client test

  `ssh -o PubkeyAuthentication=no ...`     Disable public-key auth for one
                                           client test
  ----------------------------------------------------------------------------

# Flags and Arguments Used

## `ps`

-   `-p <PID>` --- select a process by process ID.
-   `-P <PPID>` --- select processes by parent process ID.

The lowercase and uppercase options are not interchangeable.

## `ss`

For:

``` bash
sudo ss -ltnp
```

-   `-l` --- show listening sockets
-   `-t` --- show TCP sockets
-   `-n` --- show numeric addresses and ports
-   `-p` --- show the process associated with each socket when
    permissions allow

## `ssh`

-   `-v` --- enable verbose debugging.
-   `-o <option>` --- provide an SSH configuration option directly on
    the command line.

Examples:

``` text
PasswordAuthentication=no
PubkeyAuthentication=no
```

These affected only the client connection being tested; they did not
change the server configuration.

## `sshd`

-   `-t` --- test SSH server configuration validity.
-   `-T` --- print the effective SSH server configuration.

This distinction was especially useful when safely hardening the
service.

## `grep`

-   `-E` --- use extended regular expressions.
-   `-i` --- ignore case.
-   `-n` --- show matching line numbers.
-   `-R` --- search recursively through directories.
-   `^` --- regular-expression anchor for the beginning of a line.

Example:

``` bash
sudo sshd -T | grep '^passwordauthentication'
```

## Shell Pipe: `|`

The pipe sends the standard output of the command on the left to the
standard input of the command on the right.

``` bash
sudo sshd -T | grep '^passwordauthentication'
```

Conceptually:

``` text
sshd -T output
      |
      v
    grep
      |
      v
matching lines only
```

# Security Practices Demonstrated

1.  Never publish private SSH keys.
2.  Protect private keys with restrictive permissions and, where
    appropriate, a passphrase.
3.  Use public-key authentication for remote administrative access.
4.  Disable direct root SSH login when it is not required.
5.  Disable SSH password authentication only after verifying key
    authentication works.
6.  Keep an existing SSH session open while testing remote
    authentication changes.
7.  Validate SSH configuration with `sshd -t` before applying it.
8.  Inspect the effective configuration with `sshd -T` rather than
    assuming an edited file was loaded.
9.  Test both the expected success path and expected failure path.
10. Use firewall rules to expose only services that need network access.
11. Avoid publishing unnecessary IP addresses, usernames, hostnames,
    private key material, or infrastructure details in public
    repositories.

# Key Takeaway

The most important lesson from this lab was that Linux administration is
not just about editing configuration files.

My workflow became:

``` text
Inspect -> Understand -> Change -> Validate -> Apply -> Verify -> Test failure conditions
```

For SSH hardening, I did not assume the server was secure simply because
the configuration file contained the correct directives. I validated the
syntax, inspected the effective configuration, safely reloaded the
service, confirmed authorized key authentication still succeeded, and
deliberately confirmed that the authentication method I intended to
disable was rejected.

That verification process is as important as the configuration change
itself.
