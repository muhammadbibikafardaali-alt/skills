# Linux Service Management and Logging

## Session Focus

This session moved from individual process management into service
management. I focused on basic process priority, `systemd`, controlling
services with `systemctl`, and starting to troubleshoot through
`journalctl`.

The important improvement was connecting earlier process knowledge with
how Linux actually manages long-running services.

## Process Priority: `nice` and `renice`

I checked process priority using:

``` bash
ps -o pid,comm,pri,ni -p $$
```

`NI` is the nice value. The practical model is:

``` text
lower NI  -> higher CPU scheduling priority
NI 0      -> normal/default
higher NI -> lower CPU scheduling priority
```

A higher nice value does not mean a process does not need much CPU. It
means it gets less scheduling preference when processes compete for CPU.

I started a process with:

``` bash
nice -n 10 sleep 500 &
```

and changed an existing process with:

``` bash
renice -n 15 -p <PID>
```

When `ps` showed `SN`, I learned:

``` text
S -> sleeping
N -> positive nice value
```

For my current path, this is enough depth on process priority. I do not
need scheduler internals.

## PID 1 and systemd

Earlier process-tree work showed:

``` text
root  1  0  systemd
```

The correct interpretation is not that root "always has PID 1." PID 1 is
the special first userspace process. On this Ubuntu VM it is `systemd`,
running as root.

One of systemd's main responsibilities is managing services and other
units.

## SSH and Socket Activation

I checked:

``` bash
systemctl status ssh
```

and found the service loaded but inactive, with:

``` text
TriggeredBy: ssh.socket
```

Then:

``` bash
systemctl status ssh.socket
```

showed:

``` text
Active: active (listening)
Listen: 0.0.0.0:22
        [::]:22
Triggers: ssh.service
```

This demonstrated socket activation:

``` text
connection on port 22
        |
        v
ssh.socket
        |
        v
systemd can trigger ssh.service
        |
        v
sshd handles the connection
```

This was also a reminder that `inactive (dead)` does not automatically
mean something is broken.

## Listing Running Services

I used:

``` bash
systemctl list-units --type=service --state=running
```

Breakdown:

``` text
systemctl list-units -> list units known/loaded by systemd
--type=service       -> only service units
--state=running      -> only running services
```

A systemd unit is an object systemd manages. Units can include services,
sockets, timers, mounts and other types.

A key distinction:

``` text
ps / top    -> process states: R, S, T, Z...
systemctl   -> service/unit states: active, inactive, failed...
```

A service can be `active (running)` while one of its processes is `S`
because that process is waiting for work.

## Inspecting Nginx

I used:

``` bash
systemctl status nginx
```

and inspected:

``` text
Loaded
Active
Main PID
Tasks
Memory
CPU
CGroup
```

Important interpretations:

-   `Loaded: loaded` means systemd found and loaded the unit definition.
-   `enabled` describes automatic-start configuration.
-   `Active: active (running)` describes the current runtime state.
-   `Main PID` is the main process systemd associates with the service.
-   `Tasks` shows tasks associated with the service cgroup.
-   `Memory` shows current use and can include a peak.
-   `CPU: 34ms` is accumulated CPU time, not `%CPU`.

The Nginx cgroup showed one master process and multiple worker
processes.

## Connecting Services to Processes

I checked the Nginx master process:

``` bash
ps -fp 4820
```

and found that its PPID was `1`.

That connected the concepts:

``` text
systemd (PID 1)
      |
      `-- nginx.service
             |
             |-- nginx master
             |-- worker
             |-- worker
             |-- worker
             `-- worker
```

This explains why service management is higher-level than simply killing
one PID.

## `systemctl stop` vs `kill`

My first instinct was:

``` bash
kill -TERM <nginx-main-pid>
```

That operates at the process level.

For a systemd-managed service, the normal administrative approach is:

``` bash
systemctl stop nginx
```

Systemd then performs the configured service shutdown procedure.

After stopping Nginx, I verified:

``` text
Active: inactive (dead)
Main PID: ... status=0/SUCCESS
```

and the logs showed successful deactivation.

## Start and Stop

Runtime controls:

``` bash
systemctl start nginx
systemctl stop nginx
```

These change what is happening **now**.

Stopping Nginx made it inactive. Starting it again created new processes
and therefore a new Main PID.

## Active vs Enabled

This was one of the most important concepts:

``` text
active / inactive
-> current runtime state

enabled / disabled
-> automatic-start configuration
```

These are independent.

Possible combinations include:

``` text
enabled + active
enabled + inactive
disabled + active
disabled + inactive
```

The three questions to keep separate are:

``` text
Loaded?  -> Does systemd know/load the unit definition?
Enabled? -> Is it configured for automatic activation/startup?
Active?  -> Is it running right now?
```

## Enable and Disable

I checked:

``` bash
systemctl is-enabled nginx
systemctl is-active nginx
```

Initially the result was:

``` text
enabled
active
```

Then:

``` bash
sudo systemctl disable nginx
```

removed:

``` text
/etc/systemd/system/multi-user.target.wants/nginx.service
```

Afterward:

``` text
disabled
active
```

This proved:

``` text
disable != stop
```

Nginx continued running. Only its startup configuration changed.

I restored it with:

``` bash
sudo systemctl enable nginx
```

and verified that it was enabled and active again.

The model is:

``` text
start / stop       -> runtime state
enable / disable   -> startup configuration
```

## Reload vs Restart

I compared:

``` bash
systemctl reload nginx
systemctl restart nginx
```

`restart` performs a stop/start cycle. I observed that the Main PID and
worker PIDs changed.

`reload` asks a service that supports it to reread/reapply configuration
without a full stop/start.

With Nginx I observed:

``` text
Main PID stayed the same
worker PIDs changed
service remained active
```

So for Nginx:

``` text
reload
-> master stays alive
-> configuration is reread
-> workers can be replaced gracefully
-> service stays available
```

Reload is not a generic way to fix a stuck service.

## Core systemctl Commands Practiced

``` bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl enable nginx
systemctl disable nginx
systemctl is-active nginx
systemctl is-enabled nginx
systemctl list-units --type=service --state=running
```

## Introduction to journalctl

After learning how to control a service, the next question was how to
find out **why** it failed.

Repeatedly restarting a failed service is not investigation.

`journalctl` queries the systemd journal, which contains logs from
services and other parts of the system.

For Nginx:

``` bash
journalctl -u nginx
```

`-u nginx` filters entries for the Nginx unit.

I could identify events we caused ourselves during the lab:

``` text
start
stop
reload
restart
successful deactivation
startup
```

## Recent Journal Entries

I used:

``` bash
journalctl -u nginx -n 10
```

Meaning:

``` text
-u nginx -> only Nginx unit entries
-n 10    -> latest 10 matching entries
```

This is useful when I only need recent activity.

## Following Logs Live

I connected this with:

``` bash
tail -f some.log
```

The journal equivalent is:

``` bash
journalctl -u nginx -f
```

`-f` means follow.

I tested this with two terminals: one followed the journal while the
other reloaded Nginx. The new log entries appeared immediately.

Workflow:

``` text
perform service action
        |
        v
systemd/service generates event
        |
        v
journal records it
        |
        v
journalctl -f displays it live
```

`Ctrl+C` exits the live `journalctl` command. It does not stop Nginx.

## Troubleshooting Workflow So Far

``` text
Service has a problem
        |
        v
systemctl status <service>
        |
        v
Check active/inactive/failed
        |
        v
Inspect Main PID and status messages
        |
        v
journalctl -u <service>
        |
        v
Look at recent/live events
        |
        v
Understand the cause
        |
        v
Choose reload, restart, configuration fix, or another action
        |
        v
Verify recovery
```

The important change is moving away from immediately killing or
restarting something and toward investigating first.

## What I Can Explain Now

I should now be able to explain:

-   basic `nice` and `renice`;
-   what PID 1 represents on this Ubuntu VM;
-   the basic role of `systemd`;
-   process vs service;
-   what a systemd unit is;
-   process state vs service state;
-   `Loaded`, `Active`, `Main PID`, `Tasks`, memory and CPU in service
    status;
-   why a running service can contain sleeping processes;
-   why a systemd service is normally managed through systemd rather
    than blindly killing its Main PID;
-   `start` vs `stop`;
-   `enable` vs `disable`;
-   `active` vs `enabled`;
-   why `disabled + active` is possible;
-   `reload` vs `restart`;
-   basic SSH socket activation;
-   `journalctl -u`;
-   `journalctl -n`;
-   `journalctl -f`.

## Commands Practiced

``` bash
ps -o pid,comm,pri,ni -p $$
nice -n 10 sleep 500 &
renice -n 15 -p <PID>
ps -o pid,comm,stat,pri,ni -p <PID>
```

``` bash
systemctl status ssh
systemctl status ssh.socket
systemctl list-units --type=service --state=running
```

``` bash
systemctl status nginx
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
systemctl is-active nginx
systemctl is-enabled nginx
sudo systemctl enable nginx
sudo systemctl disable nginx
```

``` bash
journalctl -u nginx
journalctl -u nginx -n 10
journalctl -u nginx -f
```

## Next Session

Start with a short retrieval refresher rather than rereading
definitions.

Review:

``` text
active vs enabled
start/stop vs enable/disable
reload vs restart
process state vs service state
PID 1 and systemd
journalctl -u / -n / -f
```

Then continue into a practical service troubleshooting lab:

-   filter journal entries by time and severity;
-   deliberately create a safe Nginx configuration problem in the VM;
-   observe the failure;
-   diagnose it with `systemctl` and `journalctl`;
-   fix the configuration;
-   verify recovery.

The next goal is to move from:

``` text
I know systemctl commands
```

to:

``` text
I can investigate why a Linux service failed.
```
