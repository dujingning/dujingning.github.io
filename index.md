 Linux网络编程                                 [刷新](http://xpfan.top) 
## I/O多路复用总结(I/O Multiplexing)

### The `select` and `poll` Functions Introduction(select 与 poll函数介绍)

**Background** 

   When the TCP client is handling two inputs at the same time: standard input and a TCP socket, we encountered a problem when the client was blocked in a call to fgets (on standard input) and the server process was killed. The server TCP correctly sent a FIN to the client TCP, but since the client process was blocked reading from standard input, it never saw the EOF until it read from the socket (possibly much later).

   We want to be notified if one or more I/O conditions are ready (i.e., input is ready to be read, or the descriptor is capable of taking more output). This capability is called I/O multiplexing and is provided by the select and poll functions, as well as a newer POSIX variation of the former, called pselect.

I/O multiplexing is typically used in networking applications in the following scenarios:

```markdown
**the point**

- When a client is handling multiple descriptors (normally interactive input and a network socket)
- When a client to handle multiple sockets at the same time (this is possible, but rare)
- If a TCP server handles both a listening socket and its connected sockets
- If a server handles both TCP and UDP
- If a server handles multiple services and perhaps multiple protocols
```
I/O multiplexing is not limited to network programming. Many nontrivial applications find a need for these techniques.


### I/O Models

We first examine the basic differences in the five I/O models that are available to us under Unix:
```markdown
**five I/O models**

1. blocking I/O
2. nonblocking I/O
3. I/O multiplexing (select and poll)
4. signal driven I/O (SIGIO)
5. asynchronous I/O (the POSIX aio_ functions)
```
There are normally two distinct phases for an input operation:
```markdown
1. Waiting for the data to be ready. This involves waiting for data to arrive on the network. When 
the packet arrives, it is copied into a buffer within the kernel.

2. Copying the data from the kernel to the process. This means copying the (ready) data from the 
kernel's buffer into our application buffer.
```


Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

### Blocking I/O Model

The most prevalent model for I/O is the `blocking I/O model` (which we have used for all our examples in the previous sections). By default, all sockets are `blocking`. The scenario is shown in the figure below(you need to click that link for viewing):

[Blocking I/O Model图解](https://cdn-ossd.zipjpg.com/free/3849f02b220d363ad87602f2e26dad12_2_2_photo.png)

We use UDP for this example instead of TCP because with UDP, the concept of data being "ready" to read is simple: either an entire datagram has been received or it has not. With TCP it gets more complicated, as additional variables such as the socket's low-water mark come into play.

We also refer to `recvfrom` as a system call to differentiate between our application and the kernel, regardless of how `recvfrom` is implemented (system call on BSD and function that invokes getmsg system call on System V). There is normally a switch from running in the application to running in the kernel, followed at some time later by a return to the application.

In the figure above, the process calls recvfrom and the system call does not return until the datagram arrives and is copied into our application buffer, or an error occurs. The most common error is the system call being interrupted by a signal, as we described. We say that the process is blocked the entire time from when it calls recvfrom until it returns. When `recvfrom` returns successfully, our application processes the datagram.

### Nonblocking I/O Model

When a socket is set to be `nonblocking`, we are telling the kernel "when an I/O operation that I request cannot be completed without putting the process to sleep, do not put the process to sleep, but return an error instead". The figure is below(you need to click that link for viewing):

[Nonblocking I/O Model图解](https://cdn-ossd.zipjpg.com/free/98321944e0eddf3c659bfcf67ce19560_2_3_art.png)
```markdown
1. For the first three `recvfrom`, there is no data to return and the kernel immediately returns an 
error of EWOULDBLOCK.
2. For the fourth time we call `recvfrom`, a datagram is ready, it is copied into our application 
buffer, and `recvfrom` returns successfully. We then process the data.
```

When an application sits in a loop calling `recvfrom` on a `nonblocking` descriptor like this, it is called `polling`. The application is continually `polling` the kernel to see if some operation is ready. This is often a waste of CPU time, but this model is occasionally encountered, normally on systems dedicated to one function.

### I/O Multiplexing Model

With `I/O multiplexing`, we call `select` or `poll` and `block` in one of these two system calls, instead of `blocking` in the actual I/O system call. The figure is a summary of the `I/O multiplexing model`(you need to click that link for viewing):

[I/O Multiplexing Model图解](https://cdn-ossd.zipjpg.com/free/46f250654d113f51d7e6b0e60738e44b_2_1_photo.png)

We `block` in a call to `select`, waiting for the datagram socket to be readable. When `select` returns that the socket is readable, we then call `recvfrom` to copy the datagram into our application buffer.

**Comparing to the blocking I/O model**

[Comparing to the blocking I/O model图解](https://cdn-ossd.zipjpg.com/free/47f9f62ac85d6b89dac06253aca41709_2_1_photo.png)

`Disadvantage`
using `select` requires two system calls (`select` and `recvfrom`) instead of one
`Advantage`
we can `wait` for more than one descriptor to be ready (see the `select` function later in this chapter)

**Multithreading with blocking I/O**

Another closely related `I/O model` is to use multithreading with blocking I/O. That model very closely resembles the model described above, except that instead of using `select` to block on multiple file descriptors, the program uses multiple threads (one per file descriptor), and each thread is then free to call blocking system calls like `recvfrom`.

### Signal-Driven I/O Model

The `signal-driven I/O model` uses signals, telling the kernel to notify us with the `SIGIO` signal when the descriptor is ready. The figure is below:
[Signal-Driven I/O Model图解](https://cdn-ossd.zipjpg.com/free/47f9f62ac85d6b89dac06253aca41709_2_1_photo.png)

```markdown
1. We first enable the socket for `signal-driven I/O` and install a signal handler 
using the `sigaction` system call. The return from this system call is immediate and our process 
continues; it is not blocked.
2. When the datagram is ready to be read, the `SIGIO` signal is generated for our process. 

**We can either:**
 --read the datagram from the signal handler by calling recvfrom and then notify the main loop 
 that the data is ready to be processed (Section 25.3)
 
 --notify the main loop and let it read the datagram.
```
 
The advantage to this model is that we are not blocked while waiting for the datagram to arrive. The main loop can continue executing and just wait to be notified by the signal handler that either the data is ready to process or the datagram is ready to be read.

### Asynchronous I/O Model

`Asynchronous I/O` is defined by the `POSIX` specification, and various differences in the real-time functions that appeared in the various standards which came together to form the current `POSIX` specification have been reconciled.

These functions work by telling the kernel to start the operation and to notify us when the entire operation (including the copy of the data from the kernel to our buffer) is complete. `The main difference between this model and the signal-driven I/O model is that with signal-driven I/O, the kernel tells us when an I/O operation can be initiated, but with asynchronous I/O, the kernel tells us when an I/O operation is complete.` See the figure below for example:

[Comparing to the blocking I/O model图解](https://cdn-ossd.zipjpg.com/free/b3925a28006fbcce5585e2690b5841e3_2_1_photo.png)

1. We call `aio_read` (the `POSIX` `asynchronous I/O` functions begin with`aio_` or `lio_`) and pass the kernel the following:
```markdown
·- descriptor, buffer pointer, buffer size (the same three arguments for read),
.- file offset (similar to lseek).
·- and how to notify us when the entire operation is complete.
```
This system call returns immediately and our process is not blocked while waiting for the I/O to complete.

2. We assume in this example that we ask the kernel to generate some signal when the operation is complete. This signal is not generated until the data has been copied into our application buffer, which is different from the signal-driven I/O model.

### select Function

The `select` function allows the process to instruct the kernel to either:
```markdown
- Wait for any one of multiple events to occur and to wake up the process only when one or more of these events occurs, or
When a specified amount of time has passed.
```
This means that we tell the kernel what descriptors we are interested in (for reading, writing, or an exception condition) and how long to wait. The descriptors in which we are interested are not restricted to sockets; any descriptor can be tested using select.

```markdown
#include <sys/select.h>
#include <sys/time.h>

int select(int maxfdp1, fd_set *readset, fd_set *writeset, fd_set *exceptset,
           const struct timeval *timeout);

/* Returns: positive count of ready descriptors, 0 on timeout, –1 on error */
```
**The timeout argument:**

The timeout argument tells the kernel how long to wait for one of the specified descriptors to become ready. A timeval structure specifies the number of seconds and microseconds.

```markdown
struct timeval  {
  long   tv_sec;          /* seconds */
  long   tv_usec;         /* microseconds */
};
```
There are three possibilities for the timeout:
```mark
1. Wait forever (timeout is specified as a null pointer). Return only when one of the specified descriptors is ready for I/O.
2. Wait up to a fixed amount of time (timeout points to a `timeval` structure). Return when one of the specified descriptors is ready for I/O, but do not wait beyond the number of seconds and microseconds specified in the `timeval` structure.
3. Do not wait at all (timeout points to a `timeval` structure and the timer value is 0, i.e. the number of seconds and microseconds specified by the structure are 0). Return immediately after checking the descriptors. This is called `polling`.
```
#### **Note:**

1. The wait in the first two scenarios is normally interrupted if the process catches a signal and returns from the signal handler. For portability, we must be prepared for `select` to return an error of `EINTR` if we are catching signals. Berkeley-derived kernels never automatically restart select.
2. Although the `timeval` structure has a microsecond field tv_usec, the actual resolution supported by the kernel is often more coarse. Many Unix kernels round the timeout value up to a multiple of 10 ms. There is also a scheduling latency involved, meaning it takes some time after 3. the timer expires before the kernel schedules this process to run.
On some systems, the timeval structure can represent values that are not supported by select; it will fail with EINVAL if the tv_sec field in the timeout is over 100 million seconds.
4. The const qualifier on the timeout argument means it is not modified by select on return.

**The descriptor sets arguments:***
The three middle arguments, readset, writeset, and exceptset, specify the descriptors that we want the kernel to test for reading, writing, and exception conditions. There are only two exception conditions currently supported:
1. The arrival of out-of-band data for a socket.
2. The presence of control status information to be read from the master side of a pseudo-terminal that has been put into packet mode. (Not covered in UNP)

`select` uses descriptor sets, typically an array of integers, with each bit in each integer corresponding to a descriptor. For example, using 32-bit integers, the first element of the array corresponds to descriptors 0 through 31, the second element of the array corresponds to descriptors 32 through 63, and so on. All the implementation details are irrelevant to the application and are hidden in the `fd_set` datatype and the following four macros:

```markdown
void FD_ZERO(fd_set *fdset);         /* clear all bits in fdset */
void FD_SET(int fd, fd_set *fdset);  /* turn on the bit for fd in fdset */
void FD_CLR(int fd, fd_set *fdset);  /* turn off the bit for fd in fdset */
int FD_ISSET(int fd, fd_set *fdset); /* is the bit for fd on in fdset ? */
```

We allocate a descriptor set of the fd_set datatype, we set and test the bits in the set using these macros, and we can also assign it to another descriptor set across an equals sign (=) in C.

An array of integers using one bit per descriptor, is just one possible way to implement select. Nevertheless, it is common to refer to the individual descriptors within a descriptor set as bits, as in "turn on the bit for the listening descriptor in the read set."

The following example defines a variable of type fd_set and then turn on the bits for descriptors 1, 4, and 5:
```markdown
fd_set rset;

FD_ZERO(&rset);          /* initialize the set: all bits off */
FD_SET(1, &rset);        /* turn on bit for fd 1 */
FD_SET(4, &rset);        /* turn on bit for fd 4 */
FD_SET(5, &rset);        /* turn on bit for fd 5 */
```
It is important to initialize the set, since unpredictable results can occur if the set is allocated as an automatic variable and not initialized.

Any of the middle three arguments to `select`, readset, writeset, or exceptset, can be specified as a null pointer if we are not interested in that condition. Indeed, if all three pointers are null, then we have a higher precision timer than the normal Unix `sleep` function. The `poll` function provides similar functionality.



**|| this is collected by 杜竞宁 || [click here to the top](http://xpfan.top) || 2019.8.3 星期六 || **

# end




























**end line**
