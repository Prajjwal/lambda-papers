---
layout: post
title: Exploring Linux File Locking Mechanisms in Ruby
---

*An exploration of File Locking mechanisms and their pitfalls on Linux in Ruby.*

On a recent automation spree I produced a couple of hundred lines of ruby that
needed persistence. The built-in `PStore` is a natural choice for a project that
size. I've got two tasks that make destructive writes to the same `db.pstore`,
and both could potentially be scheduled together. Here's what the documentation
has to say about that:

> PStore objects are always reentrant. But if thread_safe is set to true, then
> it will become thread-safe at the cost of a minor performance hit.

But a `Mutex` isn't helpful when you've got multiple *processes* writing to
the same resource on disk. Turns out, `PStore` also uses `File#flock` under the
hood, so if you're in the middle of a write transaction, other processes trying
to transact will simply block, just like threads would in a thread-safe
environment. Let's take a look at how it works.

## Locking Files with flock

Consider the following piece of code, saved to a file `flock.rb`:

```ruby
# flock.rb

def log(msg)
  puts "PID #{$$} - #{msg}"
end

def acquire_lock(file)
  file.flock(File::LOCK_EX)
  log "acquired exclusive lock on db"
end

def write(file)
  file.write("#{$$} was here last!")
  file.flush
end

def print_file_contents(file)
  file.rewind
  log file.read
end

log "started!"

file = open('datastore', File::CREAT | File::RDWR)

acquire_lock(file) unless ARGV[0] == 'no_lock'

write(file)
sleep 10
print_file_contents(file)
```

The script acquires an exclusive lock and writes its `PID` to it. The lock
acquisition is skipped if you pass in `no_lock` as the first command line
parameter. If you open two terminal and run `ruby flock.rb` in each, you will
find that the process that starts second simply blocks until the first process
exists, and therefore each process prints out exactly what *it* wrote to the
file last. Now do that again, but invoke the second process as follows `ruby
flock.rb no_lock`. You'll find that *process_2* doesn't hesitate writing its
`PID` to the file, even though *process_1* is holding an exclusive lock on it!

In practice, you would never see this because cooperating processes involved
would go through the correct `flock()` api.  Therefore, a few of you might find
this behavior unremarkable and perfectly acceptable.  We'll get to that in a
second, but first let's look at how `File#flock` actually works.

## File#flock Under The Hood

A cursory inspection of the (ruby source) reveals that `File#flock` actually
uses the `flock(2)` system call. The [manual](https://linux.die.net/man/2/flock)
defines it as follows:

> flock - apply or remove an advisory lock on an open file

Aha! So `File#flock` actually issues an advisory lock, which means that other
processes are free to ignore the locking protocol if they so desire. Advisory
locking is in contrast to *mandatory locking*, which prevents this sort of
thing from happening. Some environments, notably Windows(!), have certain
types of locks enforced by the filesystem by default! Ruby's file locking
inherits all of `flock(2)`'s flaws:

- Not cross-platform. The ruby documentation mentions this up front.
- No mandatory locking.
- Can only lock entire files. This is a big one. Imagine a heavily concurrent
  system such as a database, with dozens of cooperating processes writing to
  different parts of a file only being able to issue a single write at a time.

Linux provides another locking mechanism that addresses those last two points,
which we'll look at next.

## Partial Locking with fcntl

Due to the performance implications of issuing locks on entire files, most
systems provide mechanisms to lock only parts of a file. On Linux, this is
provided by the `fcntl()` system call. To quote [The Linux Programming
Interface](https://en.wikipedia.org/wiki/The_Linux_Programming_Interface):

> This form of file locking is usually called *record locking*. However, this
> term is a misnomer, because files on the UNIX system are byte sequences, with no
> concept of record boundaries. Any notion of records within a file is defined
> purely within an application.

Ruby ships with a poorly documented interface for `fcntl()` if the underlying
system supports it. Let's take a look at some basic usage:

```ruby
# partial_lock.rb

require 'fcntl'

fd = IO.sysopen('/tmp/datastore', Fcntl::O_WRONLY)
f = IO.open fd

pad = 0

# Acquire a lock for a 1 byte segment of the file starting at byte ARGV[0]
# [l_type, l_whence, l_start, l_len, l_pid]
lock = [Fcntl::F_WRLCK, IO::SEEK_SET, pad, ARGV[0].to_i, 1, $$].pack("s!s!i!L!L!i!")

f.fcntl Fcntl::F_SETLKW, lock

puts "Acquired write lock"

sleep 10
```

The example is a bit obtuse because the lock is expressed as a binary string.
Fortunately Ruby's excellent `Array#pack` takes the pain out of creating it.
The lock format is implementation dependent, so if the above code doesn't work
for you, `man fcntl` should have the correct order of arguments. On my Linux
system, the code attempts to acquire a write lock for a 1 byte segment of the
file `/tmp/datastore` starting at at byte supplied on the command line. So,
`ruby partial_lock.rb 1` will acquire a lock on a segment spanning bytes 1 and
2.

Open two terminals and play around with this. You should find that acquiring a
lock on the same segment should block if another process is already holding a
lock, but the operating system happily issues it to you if you try to acquire it
on a different segment.

It is possible to issue a mandatory lock with `fcntl()`.

## Playing Nice With External Processes

Since `File#flock` uses `flock(2)` under the hood, it stands to reason that we
can cooperate with other non ruby processes. The following C program plays nice
with our `flock.rb`.

```c
#include <sys/file.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <errno.h>
#include <unistd.h>

int main (void)
{
	int fd, lock = LOCK_EX;

	fd = open ("datastore", O_RDONLY);

	if (fd == -1)
		exit (1);

	if (flock (fd, lock) == -1) {
		if (errno == EWOULDBLOCK) {
			puts ("file is already locked!");
		}
		else {
			puts ("could not acquire lock on file!");
		}

		exit (1);
	}

	puts ("acquired lock on file, sleeping for 10s");

	sleep (10);

	return 0;
}
```

Polyglots rejoice!

## File Locking in Shell Scripts

If your system supports `flock(2)`, in all likelihood it also ships with the
`flock` command line tool that lets you manage said locks in your shell scripts.
You can also use this to acquire a lock on an existing file descriptor, check
the manual for all sorts of examples. Try running the following on two separate
terminals:

```zsh
flock '/tmp/datastore' -c "echo start; echo hi > /tmp/datastore; sleep 5; echo end"
```

The second invocation should block until the first exits. Really handy!

## Summary

- Ruby supports synchronized file operations with `File#flock`.
- It relies on the `flock(2)` system call, which isn't available on all systems.
- It can only issue a lock on the entire file, which is terrible for massively
  concurrent systems.
- It only issues advisory locks, which means external processes are free to
  ignore it completely.
- Linux provides a non-portable way to issue locks on parts of a file via
  `fcntl()`. These can also be made mandatory if the underlying FS supports it.
- The `flock` utility lets you manage said locks on the command line.

Fin ~
