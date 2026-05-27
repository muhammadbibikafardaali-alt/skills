# Linux Fundamentals

Notes and skills from completing **TryHackMe — Linux Fundamentals (Parts 1–3)**.
Part of my path toward becoming a **Platform Security Engineer**.

> My approach: I don't just record *what* a command is — I note *what it does and when I'd use it*. If I can't explain it, I don't count it as learned.

---

## Part 1 — Shell Basics, Navigation & Searching

Moving around the filesystem, reading files, and finding things from the command line.

| Command | What it does |
|---------|--------------|
| `echo` | Print text / output to the terminal |
| `whoami` | Show the current logged-in user |
| `pwd` | Print the current directory path |
| `ls` | List files in a directory (`ls -a` shows hidden files) |
| `cd` | Change directory |
| `cat` | Print the contents of a file |
| `find` | Search for files by name, type, size, or permissions |
| `grep` | Search *inside* files for matching text |

**Shell operators**

| Operator | Meaning |
|----------|---------|
| `&` | Run a command in the background |
| `&&` | Run command B only if command A succeeds |
| `>` | Redirect output to a file (overwrite) |
| `>>` | Redirect output to a file (append) |
| `*` | Wildcard — match anything |

---

## Part 2 — Files, Permissions & Remote Access

Connecting to remote machines and managing the filesystem properly.

| Command | What it does |
|---------|--------------|
| `ssh user@ip` | Securely connect to a remote machine |
| `man` / `--help` | Read the manual / see the flags for a command |
| `touch` | Create an empty file |
| `mkdir` | Create a directory |
| `cp` | Copy a file or directory |
| `mv` | Move or rename a file |
| `rm` / `rm -R` | Remove a file / remove a directory recursively |
| `file` | Identify a file's actual type |
| `su` | Switch to another user |

**Important directories**

| Path | Purpose |
|------|---------|
| `/etc` | System configuration files |
| `/var` | Variable data — including logs |
| `/tmp` | Temporary files (cleared on reboot) |
| `/root` | The root user's home directory |

---

## Part 3 — Editors, Processes & System Maintenance

Editing files, moving them between machines, and managing what's running on the system.

**Text editors:** `nano` (beginner-friendly) and `vim` (powerful, steeper learning curve).

**Moving files around**

| Command | What it does |
|---------|--------------|
| `wget <url>` | Download a file from the web |
| `scp` | Securely copy files between machines (over SSH) |
| `python3 -m http.server` | Quickly serve files from the current directory |

**Processes**

| Command | What it does |
|---------|--------------|
| `ps` / `ps aux` | List running processes |
| `top` | Live view of processes and resource usage |
| `kill <PID>` | Terminate a process by its ID |
| `systemctl` | Start / stop / enable system services |
| `fg` / `bg` / `Ctrl+Z` | Manage foreground & background jobs |

**System maintenance**

| Tool | What it does |
|------|--------------|
| `cron` / `crontab` | Schedule tasks to run automatically |
| `apt` | Install / update / remove software packages |
| `/var/log` | Where system and service logs live |

---

## Skills Gained

After these three modules I can:

- Navigate and manage the Linux filesystem entirely from the command line
- Find files by name, type, size, and permissions, and search inside them with `grep`
- Connect to and transfer files between remote machines over SSH
- Read and reason about file permissions and key system directories
- View, manage, and schedule processes and services
- Install software and locate logs for troubleshooting

## Next

- Reinforcing these skills in the terminal via **OverTheWire Bandit** (currently at level 6)
- Next stop: **web & API security** — PortSwigger Web Security Academy

---

*Part of my Platform Security learning roadmap.*
