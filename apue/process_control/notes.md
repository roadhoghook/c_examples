# Process control

## Introduction

* process creation
* process execution
* process termination
* real, effective, saved IDs
* interpreter files and `system`
* process accounting on UNIX

## Process identifiers

* every process has an unique process ID, a non-negative integer
* although unique, process IDs can be reused:
  * whenever a program terminates, its ID can ben reused by another process
  * UNIX systems have some algorithms to delay reuse, so that a new process can't be mistaken with a freshly-closed one
* process ID 0 is usually the scheduler process (swapper), it's not a program on disk but it's a system process
* process ID 1 is usually the init process (`/sbin/init`), which is invoked by the kernel at the end of the bootstrap procedure
  * this process reads the initialization files, such as `/etc/rc*`, `/etc/inittab` and the files in `/etc/init.d/`
  * then brings up the system to a certain state, i.e. multiuser
  * this process never dies, and normally is a user process with superuser priviledges (unlike the swapper)
* useful functions:
  * `pid_t getpid(void)`
  * `pid_t getppid(void)`
  * `uid_t getuid(void)`
  * `uid_t geteuid(void)`
  * `gid_t getgid(void)`
  * `gid_t getegid(void)`

## fork function

* a process can create a new process with `pid_t fork(void)`
* the new process is called the *child* process
* this function is called once, but returns twice:
  * it returns 0 in the child
  * it returns the child pid in the parent process
* after fork, both processes continue executing:
  * the child gets a complete copy of the parent's data space, stack and heap
  * the child and the parent do not share any of those information
  * the only thing that is shared is the text segment
* generally the parent's data, stack and heap are not copied immediately after the fork, since many times it is followed by an `exec`
  * instead, copy-on-write is performed
  * those memory regions are marked as read-only by the kernel
  * if any of the processes tries to modify these regions, then the kernel performs the copy (generally, a new page)
* in the "fork.c" example
  * if we run it from the terminal, we only see "before fork" once
  * if we redirect its output to a file, wee see "before fork" twice
  * this is because the standard I/O is line buffered on the terminal, and fully buffered otherwise
  * the parent and the child both have the line in their I/O buffers, so when they are flushed, they print the string twice

## File sharing

* file descriptors open in the parent are duplicated in the child too
* after fork the both have a set of file descriptors pointing to the same entries in the kernel file table
* therefore, they share the same file offsets (it's much easier for both processes to write on the same files)
* while the output can be intermixed, generally after a fork:
  * the parent waits for the child to complete
  * both the parent and the child go their own ways, closing the descriptors they don't need.
  * In this way, they don't interfere with the other's open file descriptors
* other properties are inherited by the child
  * real user/group ID, effective user/group ID
  * supplementary group IDs
  * process group ID
  * session ID
  * terminal
  * set-user-ID and set-group-ID flgas
  * current working directory
  * root directory
  * umask
  * signal masks and dispositions
  * close-on-exec flag for any open file descriptor
  * environment
  * attached shared memory segments
  * memory mappings
  * resource limits
* what is different:
  * return values from fork()
  * process IDs
  * parent PIDs
  * file locks of the parent are not inherited
  * pending alarms are cleared for the child
  * the set of pending signals for the child is set to empty
* a fork can fail if:
  * too many processes are already in the system
  * total number of processes for this real user ID exceeds system's limit
* a fork is used when:
  * a process wants to duplicate itself so that the parent and the child can execute different sections of code at the same time
    * this is usual for network servers, the parent waits for connections; when it gets one, it forks and let the child handle the connection
  * a process wants to execute a different program.
    * this is common for shells, which use `exec` after forking
    * in some OS, the action fork+exec is called spawn

## vfork function

* vfork should be used when we want to exec right after forking
* child shares same address space as parent
* no copy is performed of the parent data structures
  * better than copy-on-write since no copying is better than a little copying
* vfork guarantee that the child will run first until it calls exec or exit

## exit functions

* a process can exit normally in five ways:
  * return from main
  * calling `exit`, launching all atexit handles and closing standard I/O streams
  * calling `_exit` or `_Exit`, ignoring all handlers
  * returning from the start routine of the last thread
  * calling `phtread_exit` from the last thread in the process
* a process can terminate abnormally in three ways:
  * calling `abort` (generates SIGABRT)
  * after receiving a signal
  * the last thread responds to a cancellation request
* regardlessly of how a process terminates, the kernel performs the same procedures:
  * it closes all open file descriptors
  * releases the memory used by the process and so on..
* parent must be notified of the child termination
  * when the process terminates normally, it returns an exit status
  * if it didn't, the kernel generates one to be sento to the parent process
* if the parent terminates before the child:
  * the child is inherited by init
* if the child terminates before the parent:
  * the kernel keeps some information about each terminating process, such as:
    * process ID, termination status, amount of CPU time
  * then the kernel can discard the process, it's open file des and free the memory
  * a child which parent has not yet waited for is called *zombie process*
* `ps` shows zombie processes as Z
* if a process inherited by init dies, it will never become a zombie because init always waits for its children

