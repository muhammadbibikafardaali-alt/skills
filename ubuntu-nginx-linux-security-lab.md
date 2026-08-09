# Ubuntu + Nginx + Linux Security Lab

This document summarizes the hands-on Linux administration and
networking lab I completed using an Ubuntu VM, Nginx, UFW, and SSH.

## Lab environment

-   Ubuntu VM running Nginx
-   Mac used as the client
-   Ubuntu VM lab IP: `192.168.64.7`
-   HTTP: TCP port `80`
-   SSH: TCP port `22`

> `192.168.64.7` is a private lab-network address, not a public Internet
> address.

## 1. Nginx and HTTP

I installed Nginx and served HTML from `/var/www/html`. I accessed the
site locally and from my Mac, then inspected Nginx access/error logs.

I observed HTTP results including:

-   `200` --- successful response
-   `304` --- resource not modified; cached copy can be reused
-   `404` --- requested resource not found
-   `Permission denied` --- Linux permissions prevented Nginx from
    reading a file

This demonstrated that `localhost` refers to the local machine, while
the VM's network IP can be used by another machine that can reach that
network.

## 2. Linux permissions, users, groups, and ownership

I practiced reading permissions such as:

``` text
-rw-rw-r--
drwxrwsr-x
```

The permission sets are:

``` text
owner | group | others
```

Where `r` means read, `w` means write, and `x` means execute. On
directories, `x` also controls traversal/access through the directory.

I created and worked with a web administration group and learned that
file ownership and permissions are separate concepts.

For example:

``` text
-rw-r--r-- 1 root root ... mywebsite.html
```

means the owner is `root`, the group is `root`, and everyone else only
has read permission.

Nginx worker processes run as `www-data`. If `www-data` is neither the
owner nor a member of the file's group, Linux evaluates the **others**
permissions.

> A process receives the filesystem privileges of the user under which
> it runs.

## 3. Group inheritance with setgid

I used:

``` bash
sudo chmod g+s /var/www/html
```

Meaning:

-   `chmod` --- change mode/permissions
-   `g` --- group
-   `+` --- add
-   `s` --- set the setgid bit

For a directory, setgid makes newly created content inherit the
directory's group. This is useful for shared web directories where
multiple administrators should create files under a consistent group.

I observed a result like:

``` text
-rw-rw-r-- 1 webdev webadmin ... test2.txt
```

## 4. Nginx processes and least privilege

I inspected Nginx using:

``` bash
ps -o pid,ppid,user,cmd -C nginx
```

Options:

-   `-o` --- choose output columns
-   `pid` --- Process ID
-   `ppid` --- Parent Process ID
-   `user` --- process owner
-   `cmd` --- command
-   `-C nginx` --- select processes named `nginx`

The process tree looked conceptually like:

``` text
systemd (PID 1)
    |
    +-- nginx master (root)
           |
           +-- nginx worker (www-data)
           +-- nginx worker (www-data)
           +-- nginx worker (www-data)
           +-- nginx worker (www-data)
```

The master and workers have different responsibilities and privilege
levels. Running workers as `www-data` demonstrates the **principle of
least privilege**.

## 5. systemd and systemctl

I learned that `systemd` manages services on Ubuntu and `systemctl` is
the command used to interact with it.

Commands practiced:

``` bash
systemctl status nginx
sudo systemctl start nginx
sudo systemctl stop nginx
systemctl is-enabled nginx
sudo systemctl enable nginx
sudo systemctl daemon-reload
```

Important distinction:

``` text
start  = start the service now
enable = configure it to start automatically at boot
```

Stopping and restarting Nginx also showed that PIDs are not permanent. A
restarted process receives a new PID.

## 6. Listening ports and sockets

I inspected listening TCP sockets using:

``` bash
sudo ss -ltnp
```

Flags:

-   `-l` --- listening sockets
-   `-t` --- TCP
-   `-n` --- numeric addresses/ports
-   `-p` --- owning process

I observed ports including `80`, `22`, `53`, and `631`.

I also learned to distinguish:

``` text
0.0.0.0:80    # IPv4 listener on available interfaces
127.0.0.1:631 # IPv4 loopback only
[::]:80       # IPv6 listener
```

A major lesson was:

> A service listening on a port does not automatically mean a firewall
> permits remote access to that port.

## 7. UFW firewall

Initially UFW was inactive. I inspected it with:

``` bash
sudo ufw status verbose
sudo ufw show added
```

I allowed HTTP:

``` bash
sudo ufw allow 80/tcp
```

Here `allow` creates an allow rule, `80` is the destination port, and
`/tcp` specifies TCP.

Then I enabled UFW:

``` bash
sudo ufw enable
```

The resulting model was essentially:

``` text
Default incoming: DENY
Default outgoing: ALLOW
TCP/80: ALLOW IN
```

Later I allowed SSH:

``` bash
sudo ufw allow 22/tcp
```

This connected the concepts of network traffic, firewall policy,
listening sockets, and server processes.

## 8. SSH and systemd socket activation

I inspected:

``` bash
systemctl status ssh
systemctl status ssh.socket
```

I found that `ssh.service` could be inactive while `ssh.socket` remained
active and listening on TCP port 22. `systemd` was holding the listening
socket and could activate SSH when required.

This also explained why `ss -ltnp` showed `systemd` PID 1 associated
with port 22.

## 9. SSH host verification

From the Mac I connected with:

``` bash
ssh mo@192.168.64.7
```

On first connection, SSH presented an ED25519 host-key fingerprint.
Instead of blindly accepting it, I verified the server's fingerprint
directly on Ubuntu:

``` bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Options:

-   `-l` --- display fingerprint
-   `-f` --- specify the key file

After confirming the fingerprints matched, I accepted the host and the
Mac stored its identity in SSH `known_hosts`.

## 10. SSH troubleshooting

When an initial SSH connection closed, I checked logs rather than
randomly changing configuration:

``` bash
sudo journalctl -u ssh.service -n 20 --no-pager
```

Options:

-   `-u ssh.service` --- filter to the SSH service unit
-   `-n 20` --- newest 20 entries
-   `--no-pager` --- print directly

I also used:

``` bash
ssh -v mo@192.168.64.7
```

`-v` means **verbose** and shows debugging information about connection
establishment, host verification, key exchange, and authentication.

## 11. Public and private SSH keys

I learned the asymmetric-key model:

``` text
Private key -> secret; remains under my control
Public key  -> can be distributed to systems that should trust me
```

For SSH authentication:

``` text
Mac private key
      |
      | cryptographic proof
      v
Ubuntu verifies against matching public key
      |
      v
authentication succeeds
```

The private key itself is **not sent to the SSH server**.

On my Mac I generated an ED25519 key pair:

``` bash
ssh-keygen -t ed25519 -C "mo@ubuntu-lab"
```

Options:

-   `-t ed25519` --- key type
-   `-C` --- human-readable comment

Files:

``` text
~/.ssh/id_ed25519       # PRIVATE KEY - never share
~/.ssh/id_ed25519.pub   # PUBLIC KEY
```

## 12. Installing the public key

From the Mac I ran:

``` bash
ssh-copy-id mo@192.168.64.7
```

This installed the **public key** for the Ubuntu `mo` account,
conceptually under:

``` text
~/.ssh/authorized_keys
```

The relationship is:

``` text
Mac                                  Ubuntu

id_ed25519       PRIVATE
     stays here

id_ed25519.pub   PUBLIC ----------> ~/.ssh/authorized_keys
```

I then connected again:

``` bash
ssh mo@192.168.64.7
```

SSH asked for the **private-key passphrase**, rather than the Ubuntu
account password, and authenticated successfully using the SSH key.

The passphrase protects the private key stored on the client. It is
different from the Ubuntu user's password.

## Overall architecture demonstrated

``` text
Mac client
    |
    | HTTP / SSH
    v
Network
    |
    v
UFW firewall
    |
    | TCP/80 or TCP/22 allowed
    v
Listening socket
    |
    v
systemd / service
    |
    +--> Nginx -> www-data -> filesystem permissions -> HTML response
    |
    +--> SSH -> user authentication -> remote shell
```

## Main security lessons

1.  **Least privilege** --- services should have only the permissions
    they need.
2.  **Filesystem permissions apply to services** --- Nginx cannot bypass
    Linux access controls.
3.  **Listening and reachable are different concepts** --- firewall
    policy can block a listening service.
4.  **Default-deny incoming firewall policies reduce exposed attack
    surface.**
5.  **Read logs before changing configuration** when troubleshooting.
6.  **Verify SSH host fingerprints** instead of blindly trusting first
    connections.
7.  **Never share a private key.**
8.  **Public-key SSH lets a server verify possession of a private key
    without receiving the private key.**

## Command reference

``` bash
# Processes
ps -o pid,ppid,user,cmd -C nginx
ps -p <PID>

# Nginx/systemd
systemctl status nginx
sudo systemctl start nginx
sudo systemctl stop nginx
systemctl is-enabled nginx
sudo systemctl enable nginx
sudo systemctl daemon-reload

# Networking
sudo ss -ltnp

# Firewall
sudo ufw status verbose
sudo ufw status numbered
sudo ufw show added
sudo ufw allow 80/tcp
sudo ufw allow 22/tcp
sudo ufw enable

# SSH
systemctl status ssh
systemctl status ssh.socket
sudo journalctl -u ssh.service -n 20 --no-pager
ssh -v mo@192.168.64.7

# SSH keys
ssh-keygen -t ed25519 -C "mo@ubuntu-lab"
ssh-copy-id mo@192.168.64.7
ssh mo@192.168.64.7
```

## What this lab demonstrated

This was more than an Nginx installation. It demonstrated how core Linux
administration concepts connect:

**users and groups -\> permissions -\> processes -\> systemd services
-\> sockets and ports -\> firewall rules -\> HTTP -\> SSH -\>
cryptographic authentication**

A logical next step is SSH hardening, inspecting `authorized_keys`
permissions, learning `ssh-agent`, and eventually deploying an
application behind Nginx.
