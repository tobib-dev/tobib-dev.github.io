---
layout: post
title: "Pop the hood: Trnl - A lightweight HTTP Package"
date: 2026-03-04 08:42:30 -0600
categories: project breakdown
---

I remember reading the second edition of The C Programming Language by Brian Kernighan and Dennis Ritchie about a year ago. And something that really stood out to me was how they explained the simple ‘Hello, world!’ program we all know today. Prior to reading the book, I always thought of print functions in most languages as very trivial. Although they aren’t exactly complex, the book opened my eyes to just how many steps are required to print a simple string such as “hello, world”. This includes compiling/interpreting text files, loading them, running them, and locating the object file. Of course, the vast majority of this process is hidden from us by a concept known as abstraction, but understanding the necessary steps required to print out a simple string of text made me realize just why something as trivial as “hello, world” has become a staple for teaching programming concepts and introducing programmers to a new language. To paraphrase the K&R, once these technical steps are mastered, even for something so fleeting, everything else required to write programs becomes easy.
In homage to K&R, I start this writing in a similar fashion. Many of you reading this article have probably done some form of API development. And if you have, you have likely used tools such as Express.js or some other web framework, which means you have come across a function similar to ```listen()``` or ```listenAndServe()```. Similar to the print function in most languages, the details that go behind the scenes when these functions are called have been abstracted from their users, so that they can be thought of as trivial, but these functions are anything but. So what actually happens when you call this function? Your OS does some magic in the background such as creating something known as a socket (don’t fret if you have no idea what this is, I will be touching it again), binds the port you provided to your server, sets up a listener to listen for incoming connections from clients using your socket, accepts new connections, then creates a pipe to send and receive data between your client and your server. This heavy lifting is something your framework/package and OS collaborate on, letting you focus on your application logic.
The outline for this writing is as follows:
What exactly is HTTP?
How the transport layer powers HTTP
Building an HTTP Package
Routing
Going Dynamic (Beyond Static Routing)

## What exactly is HTTP?
According to MDN:
> HTTP is an application-layer protocol for transmitting hypermedia documents, such as HTML. It was designed for communicating between web browsers and web servers, but can also be used for other purposes, such as machine-to-machine communication, programmatic access to APIs, and more.

Two phrases stand out to me in this definition and will shape what I discuss in the section and the next two sections: the first is the application-layer protocol, and the second is communication between web browsers and web servers. Chances are that if you are reading this, you know what an application-layer protocol is, but if you aren’t, you are in luck. The vast majority of what we do today as software engineers is built on the network stack, sometimes referred to as TCP/IP or the OSI Model. The OSI model is a conceptual framework for understanding networking layers, while TCP/IP is more of a practical, real-world implementation. The OSI model divides networking into seven conceptual layers, but for this article, we are only concerned about the Transport and Application layers:
Transport Layer: This layer is what powers web servers as we know them today, using Sockets. There are different types of sockets, but we are most focused on IP Sockets, which can be further classified into two: TCP and UDP Sockets.
Application Layer: This is the layer where we do much of our work as developers. We aren’t really referring to applications in the sense we know them today (mobile and web applications); we are referring to the standards on which modern applications are built. Examples of this include HTTP, HTTPS, SSH, FTP, SMTP, and IMAP.
![Alt text](https://cf-assets.www.cloudflare.com/slt3lc6tev37/6ZH2Etm3LlFHTgmkjLmkxp/59ff240fb3ebdc7794ffaa6e1d69b7c2/osi_model_7_layers.png "OSI Model")
*Source: Cloudflare*


So now that you know what an application-layer protocol is, HTTP is just a way to broker communication between a host device that provides a service and a host device that requests a service, using HTTP messages (a request-response feedback loop). HTTP usually runs on top of TCP (HTTP/1.1 and HTTP/2). HTTP/3 instead runs over QUIC, which uses UDP. For more on HTTP, see the MDN docs reference link below.
If you remember nothing else from this section, remember this: HTTP defines the message format, TCP moves the bytes of data.

## How Does the Transport Layer Power HTTP
The transport layer, as briefly explained above, uses IP sockets to power HTTP. What exactly are sockets? I remember a couple of years ago hearing the famous saying “Everything in Linux is a file” and I took it at face value, thinking it meant that everything in Linux is a literal file (a config, a file, a folder), but it doesn’t. In Unix/Linux systems a file is represented by an integer called a file descriptor, the file descriptor is what systems uses to write and read from a file, but the file descriptors are used by everything on the systems from devices (keyboard, mice, mic, speakers), to actual files, to network connections (databases, cache, clients, servers), and even the terminal itself. To quote Beej: 
> A file descriptor is simply an integer associated with an open file.

### Sockets
A socket, according to Wikipedia, is defined as follows:
> A software structure within a network node that serves as an endpoint for sending and receiving data across the network.
In layman’s terms, a socket is just a means for communicating between two programs (a client and server in our case). In this article, I will mainly focus on one type of socket: IP Sockets, which are used to communicate between two programs over the Internet Protocol.
There are two types of IP sockets: TCP sockets (aka Stream Sockets) and UDP sockets (aka Datagram Sockets). HTTP 1.1 uses TCP, and since Trnl is an HTTP 1.1 implementation, we will focus mainly on TCP. TCP sockets are the primary type of socket used in most API development today, since most APIs use HTTP 1.1 or HTTP 2. They are so well used because they are very reliable and connection-oriented. Hence, they are often used in applications where reliability is very important. UDP, unlike TCP, isn’t as reliable because it’s connectionless, but it’s very performant.
Sockets provide interfaces to system calls that enable us to create file descriptors in our programs, which other programs can use to initiate a communication channel. These APIs are essential because every HTTP library, package, or framework ultimately sits on top of these socket abstractions.

## Building an HTTP Package
Different programming languages provide libraries that serve as APIs, allowing programmers to access socket system calls. In C, the <sys/socket.h> header file is the go-to. This library is great for learning, as you get to see how programs are set up to listen for and initiate connections with other programs with very little abstraction. The trade-off of using this library, however, is that you have to handle every minute detail yourself. For this project, I went with the net package (Network package provided by the Go standard library). I went with this package because it abstracts a good amount of the socket setup for me, and I wanted to focus more on the HTTP implementation.
So what do we need to build an HTTP Package? From a high-level, we need three major components to set up an HTTP Package
Listener: This is the simplest part of the package, as it just initializes a TCP server interface that waits for incoming connections and accepts them.
Parser: The server reads bytes of data from the incoming connection and parses them into our custom request type. A custom response type is then created and bound to the same connection, so it can write bytes back to the client.
Router: This routes requests to the functions that handle them.
![Alt text](https://mdn.github.io/shared-assets/images/diagrams/http/messages/http-message-anatomy.svg "HTTP messages format")
*Source: MDN Web Docs*


Let’s walk through a sample request:
```GET /api/users/42 HTTP/1.1```

The client initiates a TCP connection to the server.
The server accepts the connection via Accept().
The server reads raw bytes from the socket.
Parser converts bytes into a structured request object (header, metadata, body).
Router matches /api/users/:id and extracts the id (42).
The handler function generates and writes a Response.
The response is encoded in HTTP format and written to the client over a TCP connection.
These three components are what’s required to create a simple HTTP package; other features and middlewares, such as logging and authentication, can be provided, but that’s beyond the scope of this project.
After setting up the listener, it is ready to accept incoming connections. When a connection is established, a connection interface is provided. This connection interface is the implementation of the file descriptor that I mentioned earlier. Hence, the connection interface is the “file,” and it’s what allows a server and a client to send and receive streams of data. The connection interface is passed to the parser, which loads it into a response type to write data back to the client. You can’t just load the connection into a response type, however, because the response needs to include metadata (as described above). Go provides a Writer interface that supports buffered input and output operations, enabling easy back-and-forth of data between hosts. This connection interface can be wrapped around a buffered writer and supplied to one of the response type fields for delivering responses back to clients. With these two components, you practically have a working package.

```
func (s *Server) ListenAndServe() error {
	//...

	list, err := net.Listen("tcp", port)
	if err != nil {
		return err
	}
	defer list.Close()

	return s.Serve(list)
}

/*
 * Serves using a net.Listener interface as parameter
 */
func (s *Server) Serve(l net.Listener) error {
	for {
		conn, err := l.Accept()
		if err != nil {
			return err
		}

		//...

		go s.handleConnection(conn)
	}
}

/*
 * Handle connections on listeners. Respond with bad request if error
 * is encoutered while parsing. Else, serve request if valid
 */
func (s *Server) handleConnection(conn net.Conn) {
	defer conn.Close()

	res := &response{
		conn:    conn,
		writer:  bufio.NewWriter(conn),
		header:  make(Header),
		protVer: "HTTP/1.1",
	}

	req, err := parseRequest(conn)
	//...

	res.header.Set("location", req.Header.Path)
	s.Handler.ServeHTTP(res, req)
	res.Flush()
}
```

## How About Routing?
At this point, you have a semi-functional package since you can receive requests and send back responses. But what good is an HTTP package without routing?
The simplest way to implement a router is to use a hash map. Implementing a router with a hash map is very efficient, as hash maps have O(1) time complexity, enabling routes to be retrieved quickly when clients make requests. So you can implement one with the snippet below.

```
type Router struct {
 routes map[routeKey]Handler
}
```

In the snippet above, routeKey is a type with 2 fields: the HTTP method and the path. The routeKey maps to a Handler type, which is just a type for handler or callback functions. You can even make this much simpler by concatenating both the HTTP method and path into a single string, and that will be fine.
But while hash maps are excellent for exact matches, they don’t naturally support dynamic segments such as ```/api/users/:id``` without more complex parsing logic. What’s a data structure that is very flexible for this type of use case but is still fast enough that we won’t have a lot of performance overhead?

## Going Dynamic (Beyond Static Routing)
If you guessed Tries, you would be right. If you’ve taken any form of Data Structures and Algorithms class or attempted Leetcode, you have likely come across Tries. 
>Tries are a tree-based data structure used primarily to store and retrieve a dynamic set of strings.
[Tries Wikipedia](https://en.wikipedia.org/wiki/Trie)
In this section, we will cover a special type of tries called Radix Tries, which are themselves a special type of Prefix Tries. Prefix Tries are a type of tree data structure mainly used in storing strings, especially for cases where we need distinct strings. They work well for this scenario because you can store similar words, such as ```point``` and ```pointer```, in the trie while reusing matched characters. For instance, if you want to store a word point, you can load it into the trie and denote the last character ```t``` as the end of the word. If you later get the word ```pointer```, you just need to iterate all the way up to ```t```, then add characters ```e``` and ```r```, marking ```r``` as the last character for the word ```pointer```. This means we can store lots of strings in a trie with as few duplicates as possible. Prefix Tries are usually implemented as below.

```
type Node struct {
 children map[string]*Node
 endOfWord bool
}

type Trie struct {
 root *Node
}
```
![Alt text](https://pages.cs.wisc.edu/~kolbly/img.jpg "Prefix Trie Example")
*Source: University of Wisconsin-Madison*


A standard prefix trie stores one character per node. But in routing, our segments are already in meaningful units such as (api, users, :id). A radix trie compresses common prefixes into single nodes, reducing memory overhead and traversals.
With a Radix trie, we can easily identify which routes are dynamic and which are static because all we need to do is iterate over all subpaths and the trie in a single pass. During traversal, we check whether each subpath exists as a node in the trie; if not, we check whether there is a child node with a dynamic parameter. If a dynamic parameter exists, it’s a dynamic route; else, it’s an unregistered path. The cool thing about Radix tries is that there is no overhead for static or dynamic routes; it performs the same for both. And the time complexity is O(k), where k is the number of partitions in a given route. Therefore, in the endpoint “/api/users”, k is 2, and in “/api/users/:id”, k is 3. See below for the radix trie implementation for Trnl; the fields parameter is added to store params such as id. A parameter retrieval method is provided for retrieving parameters along a dynamic route, encapsulating the trie from users.

```
type rNode struct {
 children     map[string]*rNode  // Map subpath string to children node
 handlers     map[string]Handler // Map HTTP methods to handler functions
 dynamicRoute bool
 param        string
 endOfPath    bool
}

// Mux is the radix trie
type Mux struct {
 root *rNode
}

// Return value for matchRoute
type routeMatch struct {
 handler    Handler
 params     map[string]string
 hasParams  bool
 pathExists bool
}
```
![Alt text](https://upload.wikimedia.org/wikipedia/commons/6/63/An_example_of_how_to_find_a_string_in_a_Patricia_trie.png "Radix Trie Example")
*Source: Wikipedia*


The time and space optimizations of tries (specifically Radix tries) make them ideal not just for HTTP Package routing but IP Address routing.
Trnl is not to be used in production, as it still lacks more reliable features, such as the following:
Client Interface: You might have noticed that the Trnl README only provides examples from an HTTP server. This is because Trnl doesn’t have a client interface at the moment; hence, there is no way to create client apps using Trnl. Go’s standard HTTP package does provide both client and server implementations. Trnl is an experimental project, and while I do plan on adding a client interface at a later date, it’s not in the roadmap for the short-term
Context: The current Trnl implementation doesn’t provide request context. Contexts are very important because they provide databases with information about the request state. This is especially useful in a network interruption, as you don’t want databases proceeding with transactions during an interruption. This is also a feature that I plan to introduce later.
Query Parameters: There are currently no query parameter features for Trnl, which means that if a user makes a request with a query parameter such as ```“/api/users?sort=asc”```, the sort parameter will be lost.
If you made it this far, I’m extremely grateful you took the time to read this article. I know it was a long read; I was dreading it while writing, but there was just no easy way for me to get this across. I hope you have learned something from this article. See you at the next one.

## References 
- Beej’s Guide to Network Programming - [BGNET](https://beej.us/guide/bgnet/)
- TCP/IP Illustrated - [TCP/IP Illustrated](https://www.amazon.com/TCP-Illustrated-Vol-Addison-Wesley-Professional/dp/0201633469)
- MDN Docs - [MDN HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP)
- Neetcode - [Trees & Prefix Trees](https://neetcode.io/courses/advanced-algorithms/6)
- Radix Trees - [LWN.net](https://lwn.net/Articles/175432/)
- Wikipedia -  [Radix Trees](https://en.wikipedia.org/wiki/Radix_tree)
