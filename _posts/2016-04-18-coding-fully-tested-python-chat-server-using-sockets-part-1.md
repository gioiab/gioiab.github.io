---
layout: post
title:  "Coding a fully tested Python chat server using sockets – Part 1"
date:   2016-04-18 00:00:00
author: Gioia Ballin
categories: programming
tags:	python
cover:  "/assets/texting.jpg"
disqus_id: 4757697065
---

Hey, guys! It's time to solve a typical interview challenge. In particular, this (and the following) post will guide you through the creation of a simple fully tested chat server using the Python programming language.

Let's start directly from the statement of the problem.

> Write a very simple chat server that listens on TCP port 10000 for multiple client connections and broadcasts each incoming client message to all the other connected clients. The program should be fully tested too.

Reading carefully, we can see the challenge is twofold. Indeed, in order to solve it, we need to:

1. write the code for a chat server able to broadcast messages to all the clients;
2. write the code for testing the chat server created at the previous point.

So, __the chat server code must be testable__ because, at the other side, __we must be able to test it__.

From my knowledge<small>*</small>, there are two basic ways to solve this kind of problem with Python. The former, combines pure Python with socket programming for the implementation of the chat server and uses standard unit testing tools, i.e. the <tt>unittest</tt> module, for the creation of the tests. The latter, uses [Twisted](https://twistedmatrix.com/trac/){:target="_blank"} to build up the chat server and [twisted.trial](http://twistedmatrix.com/trac/wiki/TwistedTrial){:target="_blank"} for the testing purposes. In this post, we'll leave apart Twisted for a moment and we'll see how to implement a simple pythonic chat server with socket programming. In the next post, we'll dive into the testing of the created chat server.

> Let's start!

Google is always such a seductive tool to look up for solutions to our problems.

![](http://programmers.life/wp-content/uploads/2015/03/tirinhaEN-107.png)
*Source: [Programmer's Life](http://programmers.life/){:target="_blank"}*

Anyway, in this case, if you try googling for an answer you won't get what you really would like to. In particular, among the very first results you will find [BinaryTides](http://www.binarytides.com/){:target="_blank"} and [bogotobogo](http://bogotobogo.com/){:target="_blank"}. [BinaryTides](http://www.binarytides.com/code-chat-application-server-client-sockets-python/){:target="_blank"} provides a solution which is entirely contained in the `if __name__ == "__main__"`. On the other side, [bogotobogo](http://www.bogotobogo.com/python/python_network_programming_tcp_server_client_chat_server_chat_client_select.php){:target="_blank"} shows a slightly improved solution with respect to BinaryTides: here, the code has been structured so that the chat server is given by a function. Is this sufficient for us? Unfortunately no, it isn't.

> What's the main idea then?

Ideally, we want the chat server to be an object. Furthermore, the chat server object should have two main public methods:

- a __start__ method: which actually runs the chat server.
- a __stop__ method: which stops the current execution of the chat server.

> Why do we need an object with a start and a stop method?

A unit test is a fully auto-contained piece of software. What does this software unit do once it is executed? Well, it basically initiates the context of the test, performs the checks for which it has been developed and finally, it cleans up the context. You can see clearly the reason of the __start__ and the __stop__ methods: we can't have the unit test for the chat server running indefinitely so we need both a way to start it and a way to stop it once the checks are completed. This is also the reason why the solutions provided by _BinaryTides_ and _bogotobogo_ don't fit the requirements of the problem statement.

> Let's go down deep into the solution!

The script is quite complex, so we're going to analyse it step by step and finally we'll look at the overall solution.

__Lines 1-8__ simply serves the purpose of importing the necessary packages and defining two main constants: ___HOST__ and ___PORT__. These constants represent respectively the host and the port onto which the socket connection will be bound.

{% highlight python %}
import math
import struct
import socket
import select
import threading

_HOST = '127.0.0.1'  # defines the host as "localhost"
_PORT = 10000        # defines the port as "10000"
{% endhighlight %}

__Lines 10-30__ defines the _ChatServer_ object as a [threading.Thread](https://docs.python.org/2/library/threading.html#thread-objects){:target="_blank"}. This means we'll need to override the __run()__ method in order to call the __start()__ method of the parent class and get the _ChatServer_ running properly. At the beginning, three main class constants are declared for the _ChatServer_:

- __MAX_WAITING_CONNECTIONS__ specifies the maximum number of queued connections before the rejection will start.
- __RECV_BUFFER__ specifies the size (in bytes) of the receiving buffer.
- __RECV_MSG_LEN__ specifies the size (in bytes) of the placeholder contained at the beginning of the messages.

> Why do we need a placeholder at the very beginning of each message?

If we want to closely follow the problem statement, we don't necessarily need it. Think about the situation in which the client is sending a considerable amount of data. Let's say the client sends a 10000 bytes message while our server can receive at most 4096 bytes at a time. Now, we're facing with a decision: we may simply assume the server collects the first 4096 bytes of every client message or we may study a solution to retrieve the full 10000 bytes message (and generally a message of any length). Let's make things harder and try to implement a solution allowing for the complete receipt of each single message. Since TCP/IP is a stream-based protocol, we need to define our own message-based protocol on top of TCP/IP in order to distinguish the start and the end of each individual message. An easy and effective protocol, suggested in a [Stack Overflow thread](http://stackoverflow.com/questions/17667903/python-socket-receive-large-amount-of-data){:target="_blank"} and applied here, expects the client prefixes each message with its length. So that's why we introduced the __RECV_MSG_LEN__ constant.

{% highlight python %}
class ChatServer(threading.Thread):
    """
    Defines the chat server as a Thread.
    """

    MAX_WAITING_CONNECTIONS = 10
    RECV_BUFFER = 4096
    RECV_MSG_LEN = 4

    def __init__(self, host, port):
        """
        Initializes a new ChatServer.

        :param host: the host on which the server is bounded
        :param port: the port on which the server is bounded
        """
        threading.Thread.__init__(self)
        self.host = host
        self.port = port
        self.connections = []  # collects all the incoming connections
        self.running = True  # tells whether the server should run
{% endhighlight %}

Regarding the initialisation method, there are two relevant instance variables. The former is __self.connections__ which stores all the active connections (especially the clients' ones), while the latter is __self.running__ which allows us starting and stopping the _ChatServer_ at will.

__Lines 32-40__ defines the ___bind_socket__function which creates the server socket, binds it to the given host and port and listens for at most __MAX_WAITING_CONNECTIONS__ incoming client connections.

{% highlight python %}
def _bind_socket(self):
        """
        Creates the server socket and binds it to the given host and port.
        """
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((self.host, self.port))
        self.server_socket.listen(self.MAX_WAITING_CONNECTIONS)
        self.connections.append(self.server_socket)
{% endhighlight %}

__Lines 42-52__ defines the convenience method ___send__ which prefixes a given message with its length before sending it through a given socket connection. It will be used by the server to broadcast a client message to all the other connected clients.

{% highlight python %}
    def _send(self, sock, msg):
        """
        Prefixes each message with a 4-byte length before sending.

        :param sock: the incoming socket
        :param msg: the message to send
        """
        # Packs the message with 4 leading bytes representing the message length
        msg = struct.pack('>I', len(msg)) + msg
        # Sends the packed message
        sock.send(msg)
{% endhighlight %}

__Lines 54-84__ defines the ___receive__ function which contains the data receipt implementation logic. According to the message protocol we've chosen to build over TCP/IP, each client message is prefixed by 4 bytes representing its length. So the server, at each message receipt, will unpack it and read its first 4 bytes to get the length. Once this information has been acquired, the server will call multiple times the __recv__ method in order to get the overall message.

{% highlight python %}
    def _receive(self, sock):
        """
        Receives an incoming message from the client and unpacks it.

        :param sock: the incoming socket
        :return: the unpacked message
        """
        data = None
        # Retrieves the first 4 bytes from the message
        tot_len = 0
        while tot_len < self.RECV_MSG_LEN:
            msg_len = sock.recv(self.RECV_MSG_LEN)
            tot_len += len(msg_len)
        # If the message has the 4 bytes representing the length...
        if msg_len:
            data = ''
            # Unpacks the message and gets the message length
            msg_len = struct.unpack('>I', msg_len)[0]
            tot_data_len = 0
            while tot_data_len < msg_len:
                # Retrieves the chunk of max RECV_BUFFER size
                chunk = sock.recv(self.RECV_BUFFER)
                # If there isn't the expected chunk...
                if not chunk:
                    data = None
                    break # ... Simply breaks the loop
                else:
                    # Merges the chunks content
                    data += chunk
                    tot_data_len += len(chunk)
        return data
{% endhighlight %}

__Lines 87-104__ defines the ___broadcast__ method. Given both the incoming client socket and message, it iterates over the relevant socket connections in order to spread the message to all the other connected clients using the ___send__ method. In addition, the method detects possible client disconnections and removes the involved sockets from the list of the active sockets.

{% highlight python %}
    def _broadcast(self, client_socket, client_message):
        """
        Broadcasts a message to all the clients different from both the server itself and
        the client sending the message.

        :param client_socket: the socket of the client sending the message
        :param client_message: the message to broadcast
        """
        for sock in self.connections:
            is_not_the_server = sock != self.server_socket
            is_not_the_client_sending = sock != client_socket
            if is_not_the_server and is_not_the_client_sending:
                try :
                    self._send(sock, client_message)
                except socket.error:
                    # Handles a possible disconnection of the client "sock" by...
                    sock.close()  # closing the socket connection
                    self.connections.remove(sock)  # removing the socket from the active connections list
{% endhighlight %}

__Lines 106-147__ defines the ___run__ method which implements the overall server logic. As long as the server runs, it continuously monitors all the active sockets (contained in the __self.connections__ list) for readable activity. This process is done with the __select.select __call. The __select__ function takes as input three lists of sockets that are, respectively, waiting for reading, writing or for an exceptional condition and, in turn, it returns the lists of socket descriptors that are readable, writeable or simply have fallen into an error status. The only list we care is the list of readable sockets. At this point, if the server socket becomes readable, the server will accept and handle a new client connection. As an addition, the server will broadcast to all the other connected clients a message telling that a new client entered the chat room. On the other hand, if a client socket becomes readable, then the server will acquire the incoming message from the client and forward it to all the other connected sockets  (except for the sending one). The broadcast is performed via the ___broadcast__ function previously described.

{% highlight python %}
    def _run(self):
        """
        Actually runs the server.
        """
        while self.running:
            # Gets the list of sockets which are ready to be read through select non-blocking calls
            # The select has a timeout of 60 seconds
            try:
                ready_to_read, ready_to_write, in_error = select.select(self.connections, [], [], 60)
            except socket.error:
                continue
            else:
                for sock in ready_to_read:
                    # If the socket instance is the server socket...
                    if sock == self.server_socket:
                        try:
                            # Handles a new client connection
                            client_socket, client_address = self.server_socket.accept()
                        except socket.error:
                            break
                        else:
                            self.connections.append(client_socket)
                            print "Client (%s, %s) connected" % client_address

                            # Notifies all the connected clients a new one has entered
                            self._broadcast(client_socket, "\n[%s:%s] entered the chat room\n" % client_address)
                    # ...else is an incoming client socket connection
                    else:
                        try:
                            data = self._receive(sock) # Gets the client message...
                            if data:
                                # ... and broadcasts it to all the connected clients
                                self._broadcast(sock, "\r" + '<' + str(sock.getpeername()) + '> ' + data)
                        except socket.error:
                            # Broadcasts all the connected clients that a clients has left
                            self._broadcast(sock, "\nClient (%s, %s) is offline\n" % client_address)
                            print "Client (%s, %s) is offline" % client_address
                            sock.close()
                            self.connections.remove(sock)
                            continue
        # Clears the socket connection
        self.stop()
{% endhighlight %}

__Lines 149-153__ defines the __run__ method, which automatically overrides the homonym method in the parent class. It simply creates and binds the server socket using the ___bind_socket__ function and then executes the ___run__ method which implements the server logic.

{% highlight python %}
    def run(self):
        """Given a host and a port, binds the socket and runs the server.
        """
        self._bind_socket()
        self._run()
{% endhighlight %}

__Lines 155-161__ defines the __stop__ method which simply sets the __self.running__ variable to __False__ and closes the server socket connection. This will allow us to stop our server once the execution of the test cases is finished.

{% highlight python %}
    def stop(self):
        """
        Stops the server by setting the "running" flag before closing
        the socket connection.
        """
        self.running = False
        self.server_socket.close()
{% endhighlight %}

__Lines 164-175__ defines and executes the main function. It simply creates a ChatServer object and calls its __start()__ method.

{% highlight python %}
def main():
    """
    The main function of the program. It creates and runs a new ChatServer.
    """
    chat_server = ChatServer(_HOST, _PORT)
    chat_server.start()


if __name__ == '__main__':
    """The entry point of the program. It simply calls the main function.
    """
    main()
{% endhighlight %}

__That's it!__ Here it is the full solution based on [the threading module](https://docs.python.org/2/library/threading.html){:target="_blank"}.

{% highlight python linenos %}
import math
import struct
import socket
import select
import threading

_HOST = '127.0.0.1'  # defines the host as "localhost"
_PORT = 10000        # defines the port as "10000"

class ChatServer(threading.Thread):
    """
    Defines the chat server as a Thread.
    """

    MAX_WAITING_CONNECTIONS = 10
    RECV_BUFFER = 4096
    RECV_MSG_LEN = 4

    def __init__(self, host, port):
        """
        Initializes a new ChatServer.

        :param host: the host on which the server is bounded
        :param port: the port on which the server is bounded
        """
        threading.Thread.__init__(self)
        self.host = host
        self.port = port
        self.connections = []  # collects all the incoming connections
        self.running = True  # tells whether the server should run

    def _bind_socket(self):
        """
        Creates the server socket and binds it to the given host and port.
        """
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((self.host, self.port))
        self.server_socket.listen(self.MAX_WAITING_CONNECTIONS)
        self.connections.append(self.server_socket)

    def _send(self, sock, msg):
        """
        Prefixes each message with a 4-byte length before sending.

        :param sock: the incoming socket
        :param msg: the message to send
        """
        # Packs the message with 4 leading bytes representing the message length
        msg = struct.pack('>I', len(msg)) + msg
        # Sends the packed message
        sock.send(msg)

    def _receive(self, sock):
        """
        Receives an incoming message from the client and unpacks it.

        :param sock: the incoming socket
        :return: the unpacked message
        """
        data = None
        # Retrieves the first 4 bytes from the message
        tot_len = 0
        while tot_len < self.RECV_MSG_LEN:
            msg_len = sock.recv(self.RECV_MSG_LEN)
            tot_len += len(msg_len)
        # If the message has the 4 bytes representing the length...
        if msg_len:
            data = ''
            # Unpacks the message and gets the message length
            msg_len = struct.unpack('>I', msg_len)[0]
            tot_data_len = 0
            while tot_data_len < msg_len:
                # Retrieves the chunk i-th chunk of RECV_BUFFER size
                chunk = sock.recv(self.RECV_BUFFER)
                # If there isn't the expected chunk...
                if not chunk:
                    data = None
                    break # ... Simply breaks the loop
                else:
                    # Merges the chunks content
                    data += chunk
                    tot_data_len += len(chunk)
        return data


    def _broadcast(self, client_socket, client_message):
        """
        Broadcasts a message to all the clients different from both the server itself and
        the client sending the message.

        :param client_socket: the socket of the client sending the message
        :param client_message: the message to broadcast
        """
        for sock in self.connections:
            is_not_the_server = sock != self.server_socket
            is_not_the_client_sending = sock != client_socket
            if is_not_the_server and is_not_the_client_sending:
                try :
                    self._send(sock, client_message)
                except socket.error:
                    # Handles a possible disconnection of the client "sock" by...
                    sock.close()  # closing the socket connection
                    self.connections.remove(sock)  # removing the socket from the active connections list

    def _run(self):
        """
        Actually runs the server.
        """
        while self.running:
            # Gets the list of sockets which are ready to be read through select non-blocking calls
            # The select has a timeout of 60 seconds
            try:
                ready_to_read, ready_to_write, in_error = select.select(self.connections, [], [], 60)
            except socket.error:
                continue
            else:
                for sock in ready_to_read:
                    # If the socket instance is the server socket...
                    if sock == self.server_socket:
                        try:
                            # Handles a new client connection
                            client_socket, client_address = self.server_socket.accept()
                        except socket.error:
                            break
                        else:
                            self.connections.append(client_socket)
                            print "Client (%s, %s) connected" % client_address

                            # Notifies all the connected clients a new one has entered
                            self._broadcast(client_socket, "\n[%s:%s] entered the chat room\n" % client_address)
                    # ...else is an incoming client socket connection
                    else:
                        try:
                            data = self._receive(sock) # Gets the client message...
                            if data:
                                # ... and broadcasts it to all the connected clients
                                self._broadcast(sock, "\r" + '<' + str(sock.getpeername()) + '> ' + data)
                        except socket.error:
                            # Broadcasts all the connected clients that a clients has left
                            self._broadcast(sock, "\nClient (%s, %s) is offline\n" % client_address)
                            print "Client (%s, %s) is offline" % client_address
                            sock.close()
                            self.connections.remove(sock)
                            continue
        # Clears the socket connection
        self.stop()

    def run(self):
        """Given a host and a port, binds the socket and runs the server.
        """
        self._bind_socket()
        self._run()

    def stop(self):
        """
        Stops the server by setting the "running" flag before closing
        the socket connection.
        """
        self.running = False
        self.server_socket.close()


def main():
    """
    The main function of the program. It creates and runs a new ChatServer.
    """
    chat_server = ChatServer(_HOST, _PORT)
    chat_server.start()


if __name__ == '__main__':
    """The entry point of the program. It simply calls the main function.
    """
    main()
{% endhighlight %}

<small>* If you know another way to solve the challenge, get in touch with me!</small>
