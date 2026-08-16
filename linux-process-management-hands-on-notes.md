# Linux Process Management --- Hands-On Lab Notes

## Session Goal

Today I moved from Linux permissions and ACLs into **process
management**.

The goal was not just to memorize `ps` and `kill`. I wanted to
understand how processes relate to each other, what their states mean,
and how Linux signals can be used to control them.

I used two terminal windows during the lab so I could leave a process
running in one terminal and inspect or control it from the other.

## 1. Inspecting Processes with `ps`

I started with:

``` bash
ps -f
```

Example from my VM:

``` text
UID          PID    PPID  C STIME TTY          TIME CMD
mo          3750    3712  0 19:21 pts/0    00:00:00 /usr/bin/bash
mo          4833    3750  0 20:24 pts/0    00:00:00 ps -f
```

`-f` gives the full-format process output.

The first important columns were:

-   **PID** --- Process ID, the ID assigned to that process.
-   **PPID** --- Parent Process ID, the PID of the process that started
    it.

Here Bash had PID `3750`, while `ps` had PPID `3750`. This showed that
my Bash shell started the `ps` process.

A child process gets its own PID. It does not inherit the parent's PID;
it references the parent through PPID.

## 2. Following the Process Tree

I followed parent processes upward with:

``` bash
ps -fp <PID>
```

-   `-f` --- full-format output.
-   `-p` --- select a specific PID.

By repeatedly checking the PPID, I traced my terminal session:

``` text
systemd (PID 1, root)
└── systemd --user (PID 2359, mo)
    └── ptyxis (PID 3705)
        └── ptyxis-agent (PID 3712)
            └── bash (PID 3750)
                └── commands started from Bash
```

This made the parent/child relationship much clearer.

For normal userspace administration, following the process tree upward
eventually led me to **PID 1**, which on my VM was:

``` text
/usr/lib/systemd/systemd
```

I also tried:

``` bash
ps -fp 0
```

and got an error. PID 0 is not a normal userspace process that I can
inspect in the same way.

### Practical takeaway

If I find an unfamiliar process on a server, checking its PPID and
following the chain can help me understand **where the process came from
and what started it**.

## 3. Process States

I then used:

``` bash
ps -o pid,ppid,user,stat,comm
```

The `-o` option lets me choose the columns I want:

-   `pid` --- process ID.
-   `ppid` --- parent process ID.
-   `user` --- process owner.
-   `stat` --- process state plus additional flags.
-   `comm` --- command name.

Example:

``` text
PID    PPID USER     STAT COMMAND
3750   3712 mo       Ss   bash
5170   3750 mo       R+   ps
```

The first character in `STAT` is the main process state.

### `R` --- Running or Runnable

`ps` showed `R+`, meaning it was running or ready to run when the
snapshot was taken.

An interesting detail is that `ps` can catch itself while it is
executing.

### `S` --- Interruptible Sleep

Bash showed `Ss`.

The important part here was `S`: the process is alive but currently
waiting.

It might be waiting for user input, a timer, network traffic, an event,
or another resource.

This changed how I think about the word "sleeping." A sleeping process
is not necessarily unhealthy. Many normal services spend most of their
time sleeping while waiting for work.

### `T` --- Stopped

Later I deliberately stopped a process and saw `T`.

A stopped process still exists and keeps its PID, but it is not allowed
to continue executing until it is resumed.

So **stopped is not the same as terminated**.

## 4. Testing a Sleeping Process

I created a simple process with:

``` bash
sleep 1000
```

From another terminal I checked it with:

``` bash
ps -o pid,ppid,user,stat,comm -C sleep
```

`-C sleep` selects processes whose command name is `sleep`.

Example:

``` text
PID    PPID USER     STAT COMMAND
5300   3750 mo       S+   sleep
```

At first I expected it to show `R` because I had just run it. The
important point was that `sleep` only needs the CPU briefly. It then
waits for its timer instead of wasting CPU time.

A simplified lifecycle is:

``` text
process starts
    ↓
R — executes briefly
    ↓
S — waits for timer
    ↓
R — wakes up and finishes
```

## 5. Understanding `kill` as a Signal Command

One useful correction from this lab was the meaning of `kill`.

`kill` is not only for destroying a process. It is a command used to
**send signals to processes**.

The signals I practiced were:

``` text
SIGSTOP
SIGCONT
SIGTERM
SIGKILL
```

## 6. Stopping with `SIGSTOP`

I stopped my test process with:

``` bash
kill -STOP 5300
```

When I checked it again, its state became:

``` text
T
```

The process still existed and could still be inspected. Linux had simply
stopped it from continuing execution.

## 7. Continuing with `SIGCONT`

I resumed it with:

``` bash
kill -CONT 5300
```

Afterward it returned to the sleeping state because it was still waiting
for its timer.

The transition I observed was:

``` text
S — sleeping
    ↓ SIGSTOP
T — stopped
    ↓ SIGCONT
S — sleeping again
```

This practical test made the difference between sleeping, stopped, and
terminated much easier to understand.

## 8. PIDs Are Temporary

During an earlier test I had a `sleep` process with PID `5174`. That
process later finished.

When I accidentally tried:

``` bash
kill -STOP 5174
```

Linux returned:

``` text
No such process
```

The new `sleep` process had a different PID.

This was a useful reminder that I should **check the current PID before
acting on a process** instead of assuming an old PID is still valid.

## 9. Graceful Termination with `SIGTERM`

I tested:

``` bash
kill -TERM <PID>
```

I also learned that:

``` bash
kill <PID>
```

uses `SIGTERM` by default.

So:

``` bash
kill 5385
```

is effectively a termination request to PID `5385`.

`SIGTERM` gives an application the opportunity to terminate normally. A
real application may use that chance to close files or connections,
flush logs or buffered data, save state, and release resources.

For my simple `sleep` process there was no visible cleanup; it simply
exited and disappeared from `ps`.

## 10. Forced Termination with `SIGKILL`

I then tested:

``` bash
kill -KILL 5404
```

and:

``` bash
kill -9 5420
```

Both terminated the target processes.

`SIGKILL` is different from `SIGTERM`: the target process cannot catch,
ignore, or handle it. The kernel forces the process to terminate.

The administration habit I want to keep is:

``` text
1. Try SIGTERM.
2. Check whether the process exits.
3. Use SIGKILL only when forced termination is actually necessary.
```

I should not automatically reach for `kill -9` first.

## 11. Extra `STAT` Characters

During the lab I saw:

``` text
Ss
R+
S+
```

The first character is the main state.

The extra characters provide more information. In today's examples:

-   `s` --- session leader.
-   `+` --- process belongs to the terminal's foreground process group.

The main states I want to retain from this session are:

``` text
R → running / runnable
S → sleeping / waiting
T → stopped
```

I will learn the other states when they become relevant rather than
trying to memorize all of them at once.

## Commands Practiced

``` bash
ps -f
ps -fp <PID>
ps -o pid,ppid,user,stat,comm
ps -o pid,ppid,user,stat,comm -C sleep
ps -o pid,ppid,user,stat,comm -p <PID>

sleep 1000

kill -STOP <PID>
kill -CONT <PID>
kill -TERM <PID>
kill <PID>
kill -KILL <PID>
kill -9 <PID>
```

## What I Can Explain After This Lab

After this session I can explain:

-   what PID and PPID mean;
-   how parent and child processes relate;
-   how to manually follow a process tree;
-   why PID 1 matters on a systemd Linux system;
-   the difference between running, sleeping, and stopped processes;
-   why a sleeping process can be completely healthy;
-   how to inspect a process by PID or command name;
-   why `kill` is really a signal-sending command;
-   how `SIGSTOP` and `SIGCONT` control process execution;
-   the difference between `SIGTERM` and `SIGKILL`;
-   why `kill -9` should normally not be the first choice.

## Troubleshooting Mindset

The main skill I want to keep from this session is not one specific
command.

When I see a process, I should start asking:

``` text
What process is this?
Who owns it?
What is its PID?
Who started it?
What is its PPID?
What state is it in?
Is it working, waiting, or stopped?
Should it be running?
If I need to terminate it, can I do it gracefully first?
```

This gives me a structured way to investigate Linux processes instead of
immediately searching for a command to copy.

These fundamentals will become useful again when I work with **systemd
services, resource monitoring, Docker containers, application
troubleshooting, and security investigations**.
