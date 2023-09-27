# Chapter 2 Basic TCP Sockets
#SPns 


现在是时候了解如何编写自己的 socket 应用了。我们将从 TCP 开始。你现在或许已经准备好了亲自动手写一些实际的代码，所以我们首先看一下 TCP 客户端和服务器的工作示例，然后我们介绍基本 TCP 中使用的 socket API 的详细信息。为了让事情简单点，我们最初给出的代码只适用于 IP 协议的特定版本：IPv4，在撰写本文时，IPv4 仍然是 IP 协议的主导版本，并遥遥领先！在本章的最后，我们介绍了编写 IPv6 版本的客户端和服务器所需的（较小的）修改。 在第 3 章中，我们将演示独立于协议的应用程序的创建。

> Ramsay: IP 协议是什么……我突然发现我对此有些模糊。我想起来了一点，好像是关于 IP 地址如何定义，如何分为网络地址和主机地址。

我们的客户端和服务端示例实现了 echo 协议。这个协议按照如下方式工作：
1. 客户端连接到服务端并发送数据
2. 服务端原样回复客户端并关闭连接
在我们的应用程序中，客户端发送的数据是通过命令行参数提供的一个字符串，客户端会打印从服务端返回的数据，这样我们就能看到它返回的究竟是什么。许多系统都包含一个 echo 服务，用于 debug 和测试。

## 2.1 IPv4 TCP Client

客户端和服务器之间的区别很重要，因为两者在通信的某些步骤中使用的 socket 接口不同。我们先着重客户端，它的工作是发起与被动等待联系的服务器的通信。

一个典型的 TCP 客户端交流包含以下基本步骤：
	1. 使用 `socket()` 创建一个 TCP socket
	2. 通过 `connect()` 建立连接
	3. 通过 `send()` 和 `recv()` 进行沟通 
	4. 使用 `close()` 关闭连接

`TCPEchoClient4.c` 是 IPv4 的 TCP echo 客户端的实现。

```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include "Practical.h"

int main(int argc, char const *argv[]) {
    
    if (argc < 3 || argc > 4)
        DieWithUserMessage("Parameters",
            "<Server Address> <Echo word> [<Server Port>]");
    
    char servIP = argv[1];
    char echoString = argv[2];


    in_port_t servPort = (argc == 4)? atoi(argv[3]) : 7;


    int sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0)
        DieWithSystemMessage("socket() failed");
    

    struct sockaddr_in servAddr;
    mumset(&servAddr, 0, sizeof(servAddr));
    servAddr.sin_family = AF_INET;

    int rtnVal = inet_pton(AF_INET, servIP, &servAddr.sin_addr.s_addr);
    if (rtnVal == 0)
        DieWithUserMessage("inet_pton() failed", "invalid address string");
    else if (rtnVal < 0)
        DieWithSystemMessage("inet_pton() failed");
    servAddr.sin_port = htons(servPort);
    

    if (connect(sock, (struct sockaddr*) &servAddr, sizeof(servAddr)) < 0)
        DieWithSystemMessage("connect() failed");
    
    size_t echoStringLen = strlen(echoString);


    ssize_t numBytes = send(sock, echoString, echoStringLen, 0);
    if (numBytes < 0)
        DieWithSystemMessage("send() failed");
    else if (numBytes != echoStringLen)
        DieWithUserMessage("send()", "sent unexpected number of bytes");
    

    unsigned int totalBytesRcvd = 0;
    fputs("Received:", stdout);
    while (totalBytesRcvd < echoStringLen) {
        char buffer[BUFSIZE];
        

        numBytes = recv(sock, buffer, BUFSIZE - 1, 0);
        if (numBytes < 0)
            DieWithSystemMessage("recv() failed");
        else if (numBytes == 0)
            DieWithUserMessage("recv()", "connection closed prematurely");
        totalBytesRcvd += numBytes;
        buffer[numBytes] = '\0';
        fputs(buffer, stdout);
    }

    fputc('\n', stdout);
    
    close(sock);
    exit(0);
}
```

省略 `TCPEchoClient4.c` 的文档部分。

我们的客户端应用程序（实际上是本书中的所有程序）使用了两个错误处理函数：
- `DieWithUserMessage(const char *msg, const char *detail) `
- `DieWithSystemMessage(const char *msg)` 

这两个函数都将用户提供的消息字符串 (msg) 打印到 stderr，后跟详细消息字符串； 然后，他们调用 `exit()` 并返回错误代码，导致应用程序终止。 唯一的区别是详细消息的来源。对于 `DieWithUserMessage()`，详细消息是用户提供的。对于 `DieWithSystemMessage()`，详细消息由系统根据特殊变量 errno 的值提供（它描述了最近一次系统调用失败的原因（如果有的话））。

We call `DieWithSystemMessage()` only if the error situation results from a call to a system call that sets errno. (To keep our programs simple, our examples do not contain much code devoted to recovering from errors—they simply punt and exit. Production code generally should not give up so easily.)

仅当错误情况是由于系统调用而导致的时候，才调用，调用设置 errno 的。

Occasionally, we need to supply information to the user without exiting; we use `printf()` if we need formatting capabilities, and `fputs()` otherwise. In particular, we try to avoid using `printf()` to output fixed, preformatted strings. **One thing that you should never do is to pass text received from the network as the first argument to `printf()`. It creates a serious security vulnerability. Use `fputs()` instead.**

> Note: 
	  the `DieWith…()` functions are declared in the header “Practical.h.” However, the actual implementation of these functions is contained in the file `DieWithMessage.c`, which should be compiled and linked with all example applications in this text.

省略 `DieWithMessage.c` 的源代码

If we compile TCPEchoClient4.c and DieWithMessage.c to create program TCPEchoClient4, we can communicate with an echo server with Internet address 169.1.1.1 as follows:

```bash
TCPEchoClient4 169.1.1.1 "Echo this!" 
Received: Echo this
```

For our client to work, we need a server. Many systems include an echo server for debugging and testing purposes; however, for security reasons, such servers are often initially disabled. If you don’t have access to an echo server, that’s okay because we’re about to write one.

## 2.2 IPv4 TCP Server

We now turn our focus to constructing a TCP server. The server’s job is to set up a communication endpoint and passively wait for a connection from the client. There are four general steps for basic TCP server communication: 
1. Create a TCP socket using `socket()`. 
2. Assign a port number to the socket with `bind()`. 
3. Tell the system to allow connections to be made to that port, using `listen()`. 
4. Repeatedly do the following: 
	-  Call `accept()` to get a new socket for each client connection.
	- Communicate with the client via that new socket using `send()` and `recv()`.
	- Close the client connection using `close()`.

Creating the socket, sending, receiving, and closing are the same as in the client. The differences in the server’s use of sockets have to do with binding an address to the socket and then using the socket as a way to obtain other sockets that are connected to clients. (We’ll elaborate on this in the comments following the code.) The server’s communication with each client is as simple as can be: it simply receives data on the client connection and sends the same data back over to the client; it repeats this until the client closes its end of the connection, at which point no more data will be forthcoming.

省略服务端源代码及其文档

We have factored out the function that implements the “echo” part of our echo server. Although this application protocol only takes a few lines to implement, it’s good design practice to isolate its details from the rest of the server code. This promotes code reuse. 

`HandleTCPClient()` receives data on the given socket and sends it back on the same socket, iterating as long as `recv()` returns a positive value (indicating that something was received). `recv()` blocks until something is received or the client closes the connection. When the client closes the connection normally, `recv()` returns 0. You can find `HandleTCPClient()` in the file `TCPServerUtility.c`.

```c
#include <sys/socket.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <string.h>
#include "Practical.h"

static const int MAXPENDING = 5; // Maximum outstanding connection requests

int SetupTCPServerSocket(const char *service)
{
    // Construct the server address structure
    struct addrinfo addrCriteria;                   // Criteria for address match
    memset(&addrCriteria, 0, sizeof(addrCriteria)); // Zero out structure
    addrCriteria.ai_family = AF_UNSPEC;             // Any address family
    addrCriteria.ai_flags = AI_PASSIVE;             // Accept on any address/port
    addrCriteria.ai_socktype = SOCK_STREAM;         // Only stream sockets
    addrCriteria.ai_protocol = IPPROTO_TCP;         // Only TCP protocol

    struct addrinfo *servAddr; // List of server addresses
    int rtnVal = getaddrinfo(NULL, service, &addrCriteria, &servAddr);
    if (rtnVal != 0)
        DieWithUserMessage("getaddrinfo() failed", gai_strerror(rtnVal));

    int servSock = -1;
    for (struct addrinfo *addr = servAddr; addr != NULL; addr = addr->ai_next)
    {
        // Create a TCP socket
        servSock = socket(addr->ai_family, addr->ai_socktype,
                          addr->ai_protocol);
        if (servSock < 0)
            continue; // Socket creation failed; try next address

        // Bind to the local address and set socket to listen
        if ((bind(servSock, addr->ai_addr, addr->ai_addrlen) == 0) &&
            (listen(servSock, MAXPENDING) == 0))
        {
            // Print local address of socket
            struct sockaddr_storage localAddr;
            socklen_t addrSize = sizeof(localAddr);
            if (getsockname(servSock, (struct sockaddr *)&localAddr, &addrSize) < 0)
                DieWithSystemMessage("getsockname() failed");
            fputs("Binding to ", stdout);
            PrintSocketAddress((struct sockaddr *)&localAddr, stdout);
            fputc('\n', stdout);
            break; // Bind and listen successful
        }

        close(servSock); // Close and try again
        servSock = -1;
    }

    // Free address list allocated by getaddrinfo()
    freeaddrinfo(servAddr);

    return servSock;
}

int AcceptTCPConnection(int servSock)
{
    struct sockaddr_storage clntAddr; // Client address
    // Set length of client address structure (in-out parameter)
    socklen_t clntAddrLen = sizeof(clntAddr);

    // Wait for a client to connect
    int clntSock = accept(servSock, (struct sockaddr *)&clntAddr, &clntAddrLen);
    if (clntSock < 0)
        DieWithSystemMessage("accept() failed");

    // clntSock is connected to a client!

    fputs("Handling client ", stdout);
    PrintSocketAddress((struct sockaddr *)&clntAddr, stdout);
    fputc('\n', stdout);

    return clntSock;
}

void HandleTCPClient(int clntSocket)
{
    char buffer[BUFSIZE]; // Buffer for echo string

    // Receive message from client
    ssize_t numBytesRcvd = recv(clntSocket, buffer, BUFSIZE, 0);
    if (numBytesRcvd < 0)
        DieWithSystemMessage("recv() failed");

    // Send received string and receive again until end of stream
    while (numBytesRcvd > 0)
    { // 0 indicates end of stream
        // Echo message back to client
        ssize_t numBytesSent = send(clntSocket, buffer, numBytesRcvd, 0);
        if (numBytesSent < 0)
            DieWithSystemMessage("send() failed");
        else if (numBytesSent != numBytesRcvd)
            DieWithUserMessage("send()", "sent unexpected number of bytes");

        // See if there is more data to receive
        numBytesRcvd = recv(clntSocket, buffer, BUFSIZE, 0);
        if (numBytesRcvd < 0)
            DieWithSystemMessage("recv() failed");
    }

    close(clntSocket); // Close client socket
}
```

Suppose we compile TCPEchoServer4.c, DieWithMessage.c, TCPServerUtility.c, and AddressUtility.c into the executable program TCPEchoServer4, and run that program on a host with Internet (IPv4) address 169.1.1.1, port 5000. Suppose also that we run our client on a host with Internet address 169.1.1.2 and connect it to the server. The server’s output should look like this:

```bash
TCPEchoServer4 5000 
Handling client 169.1.1.2
```

While the client’s output looks like this:
```bash
TCPEchoClient4 169.1.1.1 "Echo this!" 5000 
Received: Echo this!
```

The server binds its socket to port 5000 and waits for a connection request from the client. The client connects, sends the message “Echo this!” to the server, and receives the echoed response. In this command we have to supply TCPEchoClient with the port number on the command line because it is talking to our echo server, which is on port 5000 rather than the well-known port 7.

We have mentioned that a key principle for coding network applications using sockets is Defensive Programming: your code must not make assumptions about anything received over the network. What if you want to “play” with your TCP server to see how it responds to various incorrect client behaviors? You could write a TCP client that sends bogus messages and prints results; this, however, can be tedious and time-consuming. A quicker alternative is to use the telnet program available on most systems. This is a command-line tool that connects to a server, sends whatever text you type, and prints the response. Telnet takes two parameters: the server and port. For example, to telnet to our example echo server from above, try `telnet 169.1.1.1 5000`

Now type your string to echo and telnet will print the server response. The behavior of telnet differs between implementations, so you may need to research the specifics of its use on your system.

Now that we’ve seen a complete client and server, let’s look at the individual functions that make up the Sockets API in a bit more detail.

## 2.3 Creating and Destroying Sockets