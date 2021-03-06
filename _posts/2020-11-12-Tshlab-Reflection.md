---
title: "Tshlab Reflection"
lang: en
toc: true
permalink: /posts/tshlab
tags:
  - study
---
#  tshlab

“tsh” refers to tiny shell. In this lab, I implement a working Linux shell program that supports a simple form of job control and I/O redirection. The knowledge behind are process control and signalling. Things always turn crazy when in the world of concurrency. Everything is so enthusiastic that **race conditions** are always around the corner. Using signals, a powerful yet hard to master weapon, to maintain order.



### My roadmap

As always, there are many ways to crack this. Feel free to use your imagnation.

1. implement `quit` [this is quite simple]
2. implement "creating a child process" [read the textbook if you do not know where to start]
3. implement "eval: parent: wait for foreground jobs"
4. implement "eval: parent: deal with background jobs" [which is doing nothing right?]
5. redo 1 - 4 and implement using signal [sigchld_handler and sigsuspend are your friends]
6. consider job_list now (together with 5) [**toughest part**, so take your time]
7. implement the `jobs` [this is a simple built-in command. no need to fork]
8. handle job termination conditions [`waitpid` does returns the status]
9. handle other signals the shell receives [yes! the `sigint_handler` and `sigtstp_handler`]
10. implement `fg` and `bg` [mostly about modifying the job state]
11. implement I/O redirection [The key is  `dup2`, do not forget that `jobs` also output things~]
12. improve error handling [use `perror` is better than `sio_eprintf`]



### Hints

1. think about what signals need to be blocked before `sigsuspend`
2. child process copies parent's block vector directly, so remember to unblock signals when necessary
3. change `group id` of child process with `setpgid(0, 0)` to make sure that the only job in the `foreground group` is the `tsh`. 
4. protect all operations that access `job_list` by blocking signals 
5. all reaping should be performed in `sigchld_handler`
6. think about when and how should `sigchld_handler` inform `parent process` that a `foreground job` is finished (reaped)
7. all signal-handlers need to save and restore `errno`
8. can signal-handler be interrupted by other signals? [No!]
9. `fg` and `bg` can be tricky. What should you do in terms of the built-in command `fg`? [wait!]
10. `bg %1` and `bg 1` are different! `%jid`, `pid`. Do not forget the error checking! (`bg %biteme!`)
11. `kill` can do more than killing [eg: send `SIGCONT` to certain process]



### Function Cheatsheet

Following are some functions that I use a lot (or learnt during coding.) Please refer to `man linux` always! They are the law...

#### C/C++

* `strspn` 
  * `size_t strspn(const char* str1, const char* str2);`
  * returns the length of the initial portion of str1 which consists only of characters that are part of str2
  *  if the first character in *str1* is not in *str2*, the function returns zero.

#### Linux 

##### I/O and error handling

* `open`  open (possibly create) a file or device

  `int open(const char* pathname, int flags, mode_t mode);`

  * `flags`
    * `O_RDONLY`, `O_WRONLY`, `O_RDWR` read-only, write-only, read-write
    * `O_APPEND` open file in append mode
    * `O_CREAT` create a file if it does not exist
    * ...
  * `mode` specifies the permission to use in case a new file is created
    * **MUST SUPPLY when O_CREAT is SPECIFIED**
    * `S_IRWXU`, `S_IRUSER`, ...
    * Check [man open](https://linux.die.net/man/2/open)
  * returns -1 on error, new file descriptor on success

* `perror` produces a message on standard error describing the last error encountered

  `void perror(const char *s)`

  * if `s` is not NULL and `*s` is not a null byte, then return

    `[s]: [error msg corresponding to the current value or errno] \n`

##### Signal related

* `sigpromask` fetch and/or change the signal mask of the calling thread

  ` int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);`

  * `how` 
    * `SIG_BLOCK`: (or) blocked signals is the union of the current set and the input set arg
    * `SIG_UNBLOCK`: signals in sets are removed from the current set of blocked signals
    * `SIG_SETMASK`: (replace) the set of blocked signals is set to the argument set

  * returns 0 on success, -1 on error. In event of error, `errno` is set to indicate the cause

* `sigsuspend` temporarily replaces the signal mask of the calling thread with the mask given by `mask` and then suspends the thread until delivery of a signal whose action is to invoke a signal handler or to terminate a process

  `int sigsuspend(const sigset_t *mask);`

  * return -1, with `errno` set to indicate the error

* `setpgid` set the PGID of the process specified by `pid` to `pgid`

  `int setpgid(pid_t pid, pid_t pgid);`

  * if `pid` is 0, process ID of the calling porcess is used
  * if `pgid` is 0, the PGID of the process specified by pid is made the same as its process ID

  * returns 0 on success, 01 on error with `errno` set appropriately

* `kill` send signal to a process (**It can accomplish more than killing**)

  `int kill(pid_t pid, int sig)`

* `waitpid` suspends execution of the calling process until a child specified by `pid` arg has changed state

  `pid_t waitpid(pid_t pid, int* status, int options);`

  * `pid`
    * `<-1`: wait for any child process whose process group ID == `|pid|`
    * `-1` : wait for any child process
    * `0`: wait for any child whose process group ID == that of the calling process
    * `>0`: wait for the child whose PID == value of `pid`
  * `options`
    * `WNOHANG`: return immediately if no child has exited
    * `WUNTRACED`: return if a child has stopped
    * ...
    * check [man waitpid](https://man7.org/linux/man-pages/man2/waitpid.2.html)
  * `status` output

  * return PID of the child whose state has changed. If `WNOHANG` was specified and one or more child specified by `pid` exist, but have not yet changed state, then 0 is returned. return -1 on error



