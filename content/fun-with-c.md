+++
title = "More fun with C"
date = 2018-01-22

[taxonomies]
categories = ["programming", "c"]
+++

This happened during the last semester but it was way too interesting to not
write about and I finally got myslef to write again. Similar to the [last
time](@/c-array-iteration.md) this while working on an assignment for my
Operating Systems class. In this assignment we had to implement signals in the
operating system. Before getting into that, a small crash course on signals.

## Linux Signals

Signals are an asynchronous mechanism of notifying a running process about an
event which occurred. These are generally sent by the kernel itself, but
different processes can send each other signals as well.

They are generally controlled via the
[`signal(7)`](http://man7.org/linux/man-pages/man7/signal.7.html) system call
which sets up the deposition for a given signal. You can also register handlers
which will be invoked when the signal is delivered. But not all signals can be
handled.

This leads to an interesting design problem. In a way, signals are similar to
threads in that there is a function run separately to the main execution, but
unlike threads, where the main process might have some control over how threads
are being run, when a signal handler is invoked, the process has no idea that
actually happened.

When a signal is delivered, the execution switches to the signal handler and
then back to the process without the process knowing any better! In a way,
signals are like userspace interrupts. One thing to note, is that the execution
of the handler is still happening in the same context as the process *(thank you
[@siddesh_p](https://twitter.com/siddhesh_p) for the note)*.


So back to to assignment, as a part of implementing the handling of signals,
I needed to keep a copy of the process's `trapframe` so that it could be
restored after the handler was run. To do this, I just added a new member to
the `proc` structure which was a representation of the current executing
process.

```c
struct proc {
    addr_t sz;
    pde_t* pgdir;

    ... // Other process related information

    struct trapframe *tf; // A pointer to the process's trapframe

    ...

    struct trapframe *tf_copy; // A pointer to the trapframe copy
}
```

And in my setup function taking care of setting up the copy of the trapframe
appropriately, I copied the contents of the trapframe over to the backup.

```c
void setuphandler() {
    *proc->tf_copy = *proc->tf; // Copy the contents of memory at proc->tf over to proc->tf_copy

    ... // Rest of the setup
}
```

When I began testing this setup I found my little kernel crashing immediately.
After some digging in gdb (as a side note `gdb -tui` is straight up awesome),
I realised that the userspace code was not running properly. This was becuase
right after the `signal` system call was called and the signals setup, the code
for my userspace program was being modified (Only found out after a lot of hair
pulling and staring at the GDB prompt).

Before the system call was run, the code section of the processes memory was
fine, but right after, the text was completely different! Something was
modifying the processes memory and overwriting it with junk which was being run
as garbage instructions. It took a few hours and a whole lot of help from my
professor to figure this out(I seriously would not have been able to figure
this out without his help). Turns out the culprit was pointer initialization.

See when I added `trapframe` backup to the `proc` structure, I declared it as
a pointer to a `trapframe`. This meant that when a new process was initialzed,
that field contained garbage, and when I wrote the line

```c
    *proc->tf_copy = *proc->tf; // Copy the contents of memory at proc->tf over to proc->tf_copy
```

C did not have any problems and the code ended up copying the contents of the
trapframe to whatever memory location being pointed to by the variable, which
just happened to the text section of the process's memory space. The original
`trapframe` (`struct trapframe *tf`) field in the `proc` was being initialzed
properly when a new `proc` structure was allocated.

The fix was straightforward. Instead of storing a pointer to a `trapframe` as
a copy, my professor suggested to just store the entire `trapframe` in the
process's memory space.


```c
struct proc {
    addr_t sz;
    pde_t* pgdir;

    ... // Other process related information

    struct trapframe tf_copy
}
```

So when now, when the contents were being backed up, they would not be
overwriting the contents of some random memory address.

This adventure just made me love and respect C that much more. You have to
understand how the underlying machine works and in doing so you gain complete

