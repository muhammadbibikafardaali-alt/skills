# Linux Process Management --- Jobs, Resources and Zombies

## Session Goal

This session continued my process-management practice. I focused on
foreground/background jobs, CPU and memory monitoring, investigating a
high-CPU process, and zombie processes.

The useful part was connecting these topics with what I already learned
about PID, PPID, process states and signals.

## Foreground and Background Jobs

Normally:

``` bash
sleep 500
```

keeps the shell waiting until the command finishes.

Adding `&`:

``` bash
sleep 500 &
```

runs it in the background and gives the shell prompt back immediately.

Bash returned something like:

``` text
[1] 4113
```

These are different identifiers:

``` text
[1]  → Bash Job ID
4113 → Linux PID
```

A PID identifies the process on the system. A Job ID is maintained by
the current Bash shell for job control.

I checked shell jobs with:

``` bash
jobs
```

Example:

``` text
[1]-  Running   sleep 500 &
[2]+  Running   sleep 700 &
```

`jobs` does not list every Linux process. It shows jobs managed by the
current shell.

``` text
+ → current job
- → previous job
```

## `fg`, `Ctrl+Z` and `bg`

I moved Job 2 into the foreground with:

``` bash
fg %2
```

`%2` means Bash Job ID 2, not PID 2.

While it was in the foreground, I pressed:

``` text
Ctrl+Z
```

This stopped/suspended the process instead of terminating it. This
connected back to the `T` process state from the previous session.

I initially made a useful mistake: I ran another `sleep 700 &`, which
created a new process instead of resuming the stopped one.

The correct command to resume the existing stopped job in the background
was:

``` bash
bg %2
```

The basic job-control model is now:

``` text
&       → start in background
jobs    → show jobs managed by this shell
fg %N   → move Job N to foreground
Ctrl+Z  → stop the foreground job
bg %N   → resume Job N in background
```

I also used:

``` bash
kill %2
```

This targets a Bash Job ID instead of writing the process PID directly.
Since no signal was specified, `kill` sent the default `SIGTERM`.

## CPU and Memory with `ps`

I used:

``` bash
ps -o pid,ppid,user,stat,%cpu,%mem,comm
```

The new columns were:

``` text
%CPU → CPU usage
%MEM → percentage of physical memory used
```

This gives a quick process snapshot.

For a continuously updating view I used:

``` bash
top
```

One lesson from `top` was that the process using the most CPU is not
necessarily the process using the most memory. CPU pressure and memory
pressure are separate things to investigate.

## VIRT, RES and SHR

For `ptyxis`, `top` showed values for:

``` text
VIRT
RES
SHR
```

The mental model I am keeping is:

``` text
VIRT → virtual address space associated with the process
RES  → memory currently resident in physical RAM
SHR  → resident memory that may be shared
```

A large `VIRT` value does not automatically mean that amount of physical
RAM is being consumed.

I checked the same idea using:

``` bash
ps -p 3792 -o pid,comm,vsz,rss,%mem
```

Example from the VM:

``` text
PID   COMMAND    VSZ      RSS     %MEM
3792  ptyxis     2925140  484812  14.0
```

Here:

``` text
VSZ ≈ VIRT
RSS ≈ RES
```

For a quick physical-memory investigation, RSS/RES is more useful than
assuming VSZ/VIRT equals actual RAM usage.

## Creating a High-CPU Process

To make CPU usage obvious, I deliberately ran:

``` bash
yes > /dev/null
```

`yes` continuously generates output and `/dev/null` discards it.

From another terminal:

``` bash
ps -C yes -o pid,ppid,stat,%cpu,%mem,comm
```

showed:

``` text
PID    PPID STAT %CPU %MEM COMMAND
4378   3866 R+   99.9  0.1 yes
```

This gave a clear comparison:

``` text
sleep → S → almost no CPU
yes   → R → about 100% of one CPU
```

Moving a CPU-heavy process into the background would not solve the CPU
problem. Background/foreground controls terminal interaction, not how
much CPU the process needs.

## Investigating Before Killing

Instead of immediately using `kill -9`, I investigated where the
high-CPU process came from.

The `yes` process had:

``` text
PID  = 4378
PPID = 3866
```

I checked its parent with:

``` bash
ps -fp 3866
```

and found Bash.

I then followed the PPID chain:

``` text
yes
└── bash
    └── ptyxis-agent
        └── ptyxis
            └── systemd --user
                └── systemd
```

This confirmed that `yes` was the test process I had deliberately
started from my terminal.

The troubleshooting lesson is that finding a process with high CPU is
not automatically enough reason to kill it. I should first ask who owns
it, what it is, who started it, and whether it is expected.

## STOP vs TERM vs KILL

This session made the operational difference clearer.

### `SIGSTOP`

``` bash
kill -STOP <PID>
```

Use when I want to pause/freeze the process but keep it available to
continue later.

### `SIGTERM`

``` bash
kill -TERM <PID>
```

Use when I have decided the process should end and I want to allow
normal termination/cleanup.

### `SIGKILL`

``` bash
kill -KILL <PID>
```

or:

``` bash
kill -9 <PID>
```

Use when forced termination is required, especially when normal
termination is not working.

My decision model:

``` text
Need it to continue later? → STOP
Need it to end?            → TERM first
TERM did not work?         → KILL if appropriate
```

For the test `yes` process I used:

``` bash
kill -TERM 4378
```

and confirmed that it disappeared.

Afterward, `top` showed the CPU mostly idle again.

## Overall CPU in `top`

After terminating `yes`, I saw:

``` text
%Cpu(s): ... 97.8 id ...
```

`id` means idle CPU, so the system had plenty of unused CPU capacity at
that moment.

## Task States and Zombies

`top` also showed:

``` text
Tasks: 214 total, 1 running, 211 sleeping, 2 stopped, 0 zombie
```

I already knew running, sleeping and stopped. The new state was zombie.

A **zombie process** has already finished execution, but its parent has
not yet collected its exit status. Linux keeps a small process-table
entry until that happens.

A zombie normally appears as:

``` text
Z
```

This is different from an orphan:

``` text
Zombie → child is dead, parent has not collected the exit status
Orphan → parent is gone while the child is still alive
```

## Zombie Lab

I created a controlled zombie using Python as a lab tool:

``` bash
python3 -c 'import os,time; pid=os.fork(); time.sleep(300) if pid else os._exit(0)'
```

The Python syntax was not the learning goal. Conceptually:

``` text
Python parent
    |
    └── child
          └── exits immediately
```

The parent remained alive without collecting the child's exit status.

From the second terminal I ran:

``` bash
ps -eo pid,ppid,stat,comm | grep -E 'python|PID'
```

and got:

``` text
PID    PPID STAT COMMAND
4694   4412 S+   python3
4698   4694 Z+   python3
```

So:

``` text
4694 → parent → S → alive and sleeping
4698 → child  → Z → exited but not yet reaped
```

The zombie still showed PPID `4694`, linking it to its parent.

To clean up the lab, the parent can be terminated:

``` bash
kill -TERM 4694
```

and the process table can be checked again to confirm that the zombie no
longer remains.

## Commands Practiced

``` bash
sleep 500 &
jobs
fg %2
bg %2
kill %2
```

``` bash
ps -o pid,ppid,user,stat,%cpu,%mem,comm
ps -p <PID> -o pid,comm,vsz,rss,%mem
top
```

``` bash
yes > /dev/null
ps -C yes -o pid,ppid,stat,%cpu,%mem,comm
ps -fp <PID>
```

``` bash
kill -STOP <PID>
kill -CONT <PID>
kill -TERM <PID>
kill -KILL <PID>
kill -9 <PID>
```

``` bash
ps -eo pid,ppid,stat,comm
```

## What I Should Be Able to Explain

After this session I should be able to explain:

-   foreground vs background processes;
-   what `&` does;
-   Job ID vs PID;
-   `jobs`, `fg`, `bg`, and `Ctrl+Z`;
-   why backgrounding a process does not reduce its CPU usage;
-   CPU vs memory usage;
-   `VIRT/VSZ` vs `RES/RSS`;
-   how to identify a CPU-heavy process;
-   why checking PPID and the parent chain matters before taking action;
-   when `SIGSTOP`, `SIGTERM`, and `SIGKILL` make sense;
-   what a zombie process is;
-   zombie vs orphan;
-   how PID/PPID, states, signals and resource monitoring fit together
    during troubleshooting.

## Troubleshooting Workflow

The process-investigation workflow I practiced was:

``` text
System feels slow
      ↓
Check top / ps
      ↓
Identify high CPU or memory process
      ↓
Check PID, USER, COMMAND and STAT
      ↓
Inspect PPID / parent chain
      ↓
Decide whether the process is expected
      ↓
Choose the appropriate action
      ↓
STOP temporarily / TERM cleanly / KILL only when needed
      ↓
Verify the result
```

The main improvement is that process management is becoming an
investigation workflow instead of just a list of commands.

## Next Session --- Start With a Refresher

Before moving to new material, I want to retrieve the previous knowledge
from memory first.

Review:

-   PID vs PPID;
-   `R`, `S`, `T`, and `Z`;
-   `SIGSTOP`, `SIGCONT`, `SIGTERM`, and `SIGKILL`;
-   foreground/background jobs;
-   Job ID vs PID;
-   `jobs`, `fg`, `bg`, and `Ctrl+Z`;
-   CPU vs memory;
-   `VIRT/VSZ` vs `RES/RSS`;
-   zombie vs orphan.

Finish the refresher with a short practical troubleshooting scenario
where I choose the investigation commands myself instead of being given
the commands.
