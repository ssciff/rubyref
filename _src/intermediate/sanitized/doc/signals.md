# Caveats for implementing Signal.trap callbacks

As with implementing signal handlers in C or most other languages, all code
passed to Signal.trap must be reentrant.  If you are not familiar with
reentrancy, you need to read up on it at
[Wikipedia](https://en.wikipedia.org/wiki/Reentrancy_(computing)) or elsewhere
before reading the rest of this document.

Most importantly, "thread-safety" does not guarantee reentrancy; and methods
such as `Mutex#lock` and `Mutex#synchronize` which are commonly used for
thread-safety even prevent reentrancy.

## An implementation detail of the Ruby VM

The Ruby VM defers Signal.trap callbacks from running until it is safe for its
internal data structures, but it does not know when it is safe for data
structures in YOUR code.  Ruby implements deferred signal handling by
registering short C functions with only [async-signal-safe
functions](http://man7.org/linux/man-pages/man7/signal-safety.7.html) as
signal handlers.  These short C functions only do enough tell the VM to run
callbacks registered via Signal.trap later in the main VM loop.

## Unsafe methods to call in Signal.trap blocks

When in doubt, consider anything not listed as safe below as being unsafe.

*   `Mutex#lock`, `Mutex#synchronize` and any code using them are explicitly
    unsafe.  This includes Monitor in the standard library which uses Mutex to
    provide reentrancy.

*   Dir.chdir with block

*   any IO write operations when `IO#sync` is false; including `IO#write`,
    IO#write_nonblock, IO#puts. Pipes and sockets default to `IO#sync = true`,
    so it is safe to write to them unless IO#sync was disabled.

*   `File#flock`, as the underlying flock(2) call is not specified by POSIX


## Commonly safe operations inside Signal.trap blocks

*   Assignment and retrieval of local, instance, and class variables

*   Most object allocations and initializations of common types including
    Array, Hash, String, Struct, Time.

*   Common Array, Hash, String, Struct operations which do not execute a block
    are generally safe; but beware if iteration is occurring elsewhere.

*   `Hash#[]`, `Hash#[]=` (unless Hash.new was given an unsafe block)

*   `Thread::Queue#push` and `Thread::SizedQueue#push` (since Ruby 2.1)

*   Creating a new Thread via Thread.new/Thread.start can used to get around
    the unusability of Mutexes inside a signal handler

*   Signal.trap is safe to use inside blocks passed to Signal.trap

*   arithmetic on Integer and Float (`+', `-', '%', '*', '/')

    Additionally, signal handlers do not run between two successive local
    variable accesses, so shortcuts such as `+=' and `-=' will not trigger a
    data race when used on Integer and Float classes in signal handlers.


## System call wrapper methods which are safe inside Signal.trap

Since Ruby has wrappers around many [async-signal-safe C
functions](http://man7.org/linux/man-pages/man7/signal-safety.7.html) the
corresponding wrappers for many IO, File, Dir, and Socket methods are safe.

(Incomplete list)

*   Dir.chdir (without block arg)
*   Dir.mkdir
*   Dir.open
*   `File#truncate`
*   File.link
*   File.open
*   File.readlink
*   File.rename
*   File.stat
*   File.symlink
*   File.truncate
*   File.unlink
*   File.utime
*   `IO#close`
*   `IO#dup`
*   `IO#fsync`
*   `IO#read`
*   `IO#read_nonblock`
*   `IO#stat`
*   `IO#sysread`
*   `IO#syswrite`
*   IO.select
*   IO.pipe
*   Process.clock_gettime
*   Process.exit!
*   Process.fork
*   Process.kill
*   Process.pid
*   Process.ppid
*   Process.waitpid

...