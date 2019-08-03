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
[图解](https://notes.shichao.io/unp/figure_6.1.png)

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/dujingning/dujingning.github.io/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
