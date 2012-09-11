# WebSockets

*Sven Van Caekenberghe*

*September 2012*

*(This is a draft)*


The WebSokect protocol defines a full-duplex single socket connection 
over which messages can be sent between a client and a server.
It simplifies much of the complexity around bi-directional web communication and connection management.
WebSocket represents the next evolutionary step in web communication compared to Comet and Ajax.


## An Introduction to WebSockets


HTTP, one of the main technologies of the internet, 
defines a communication protocol between a client and a server
where the initiative of the communication lies with the client 
and each interaction consists of a client request and a server repsonse.
When correctly implemented and used, HTTP is enormeously scaleable and very flexible.

With the arrival of advanved Web applications mimicing regular 
desktop applications with rich user interfaces, as well as mobile Web applications,
it became clear that HTTP was not suitable or not a great fit for two use cases:

- when the server wants to take the initiative and send the client a message
- when the client wants to send (many) (possibly asynchroneous) short messages with little overhead 

In the HTTP protocol, the server cannot take the initiative to send a message, 
the only workaround is for the client to do some form of polling.
For short messages, the HTTP protocol adds quite a lot of overhead in the form of meta data headers.
For many applications, the response (and the delay waiting for it) are not needed.
Previously, Comet and Ajax were used as (partial) solutions to these use cases.

The WebSocket protocol defines a reliable communication channel between two equal parties, 
typically, but not necessarily, a Web client and a Web server,
over which asynchroneous messages can be send with very little overhead.
Messages can be any String or ByteArray. Overhead is just a couple of bytes.
There is no such thing as a direct reply or a synchroneous confirmation.

Using WebSockets, a server can notify a client instantly of interesting events, 
and clients can quickly send small notifications to a server, possibly multiplexing
many virtual communications channels over a single network socket.


## The WebSocket Protocol


Zinc WebSockets implements <http://tools.ietf.org/html/rfc6455>, 
not any of the previous development versions.
For an introduction both <http://en.wikipedia.org/wiki/WebSocket> and
<http://www.websocket.org> are good starting points.

As a protocol, WebSocket starts with an initial setup handshake that is based on HTTP.
The initiative for setting up a WebSocket lies with the client, 
who is sending a so called connection upgrade request.
The upgrade request contains a couple of special HTTP headers.
The server begins as a regular HTTP server accepting the connection upgrade request.
When the request is conform the specifications, 
a regular 101 switching protocols response is sent.
This response also contains a couple of special HTTP headers.
From that point on, the HTTP conversation over the network socket stops 
and the WebSocket protocol begins.

WebSocket messages consist of one or more frames with minimal encoding.
Behind the scenes, a number of control frames are used to properly close the WebSocket and 
to manage keeping alive the connection using ping and pong frames.


## Source Code


The code implementing Zinc WebSockets resides in a single package called 'Zinc-WebSocket-Core' in the
Zinc HTTP Components repositories. There is also an accompanying 'Zinc-WebSocket-Tests' package
containing the unit tests. The ConfigurationOfZincHTTPComponents has a group called 'WebSocket' that 
you can load separately.


## Using Client Side WebSockets


An endpoint for a WebSocket is specified using a URL

    ws://www.example.com:8088/my-app

Two new schemes are defined, ws:// for regular WebSockets and wss:// for the secure (TLS/SSL) variant.
The host:port and path specification should be familiar.

Zinc WebSockets supports the usage of client side WebSockets of both the regular 
and secure variants (the secure variant requires Zodiac TLS/SSL). 
The API is really simple, once you open the socket, 
you use #sendMessge: and #readMessage: and finally #close. 

Here is a client side example taking to a public echo service: 

    | webSocket |
    webSocket := ZnWebSocket to: 'ws://echo.websocket.org'.
    [ webSocket 
      sendMessage: 'Pharo Smalltalk using Zinc WebSockets !';
      readMessage ] ensure: [ webSocket close ].

Note that #readMessage: is blocking. 
It always returns a complete String or ByteArray, possible assembled out of multiple frames.
Inside #readMessage: control frames will be handled automagically. 
Reading and sending are completely separate and independent.

For sending very large messages, there are #sendTextFrames: and #sendByteFrames: that take
a collection of Strings or ByteArrays to be sent as different frames of the same message.

In any non-trivial application, you will have to add your own encoding and decoding to messages.
In many cases, JSON will be the obvious choice as the client end is often JavaScript.
A modern, standalone JSON parser and writer is NeoJSON.

To use secure web sockets, just use the proper URL scheme wss:// as in the following example:

    | webSocket |
    webSocket := ZnWebSocket to: 'wss://echo.websocket.org'.
    [ webSocket 
      sendMessage: 'Pharo Smalltalk using Zinc WebSockets & Zodiac !';
      readMessage ] ensure: [ webSocket close ].

Of course, your image has to contain Zodiac and your VM needs access to the proper plugin.
That should not be a problem with the lastest Pharo 1.4 and 2.0 releases.


## Using Server Side WebSockets


Since the WebSocket protocol starts of as HTTP, 
it is logical that a ZnServer delegate is the starting point.
ZnWebSocketDelegate implements the standard #handleRequest: 
to check if the incoming request is a valid WebSocket connection upgrade request.
If so, the matching 101 switching protocols response is constructed and sent.
From that moment on, the network socket stream is handed over to a new, 
server side ZnWebSocket object.

ZnWebSocketDelegate has two properties.
An optional prefix implements a specific path, like /my-ws-app.
The required handler is any object implementing #value: 
with the new web socket as argument.

Let's implement the echo service that we connected to as a client in the previous subsection.
In essense, we should go in a loop, reading a message and sending it back.
Here is the code:

    ZnServer startDefaultOn: 1701.
    ZnServer default delegate: (ZnWebSocketDelegate handler:
      [ :webSocket |
        [ | message |
          message := webSocket readMessage.
          webSocket sendMessage: message ] repeat ]).

We start a default server on port 1701 and replace its delegate with a ZnWebSocketDelegate.
The ZnWebSocketDelegate will pass each correct web socket request on / to its handler.
In this example, a block is used as handler.
The handler is given an instanciated, connected ZnWebSocket instance.
For the echo service, we go into a repeat loop, reading a message and sending it back.

Finally, you can stop the server using

    ZnServer stopDefault.

Although the code above works it will eventually encounter two NetworkErrors:

- ConnectionTimedOut
- ConnectionClosed (or its more specific subclass ZnWebSocketClosed)

The #readMessage call blocks on the socket stream waiting for input until its timeout expires,
which will be signalled with a ConnectionTimedOut exception. 
In most applications, you should just keep on reading,
essentially ignoring the timeout for an infinite wait on incoming messages.

This behavior is implemented in the ZnWebSocket>>#runWith: convenience method:
it enters a loop reading messages and passing them to a block,
continuing on timeouts. This simplifies the examples:

    ZnServer startDefaultOn: 1701.
    ZnServer default delegate: (ZnWebSocketDelegate handler:
      [ :webSocket |
        webSocket runWith: [ :message |
          message := webSocket readMessage.
          webSocket sendMessage: message ] ]).

That leaves us with the problem of ConnectionClosed. 
This exception can occur at the lowest level when the underlying network connection closes,
or at the WebSocket protocol level when the other end sends a close frame.
In either case we have to deal with it as a server.
In our trivial echo example, we can catch and ignore any ConnectionClosed exception.

There is a handy shortcut method on the class side of ZnWebSocket that 
helps to quickly set up a server implementing a WebSocket service.

    ZnWebSocket
      startServerOn: 8080 
      do: [ :webSocket |
        [ 
          webSocket runWith: [ :message |
            message := webSocket readMessage.
            webSocket sendMessage: message ] ]
          on: ConnectionClosed 
          do: [ self crLog: 'Ignoring connection close, done' ] ].

Don't forget to inspect the above code so that you have a reference to the server to close it,
as this will not be the default server.

Although using a block as handler is convenient, for non-trivial examples, 
a regular object implementing #value: will probably be better.
You can find such an implementation in ZnWebSocketEchoHandler.

The current process (thread) as spawned by the server can be used freely by the handler code,
for as long as the web socket connection lasts.
The responsability for closing the connection lies with the handler,
although a close from the other side will be handled correctly.

To test our echo service, you could connect to it using a client side web socket,
like we did in the previous subsection. This is what the unit test ZnWebSocketTests>>#testEcho does.
Another solution is to run some JavaScript code in a web browser.
You can find the necessary HTML page containing JavaScript code invoking the echo service on
the class side of ZnWebSocketEchoHandler. 
The following setup will serves this code:

    ZnServer startDefaultOn: 1701.
    ZnServer default logToTranscript.
    ZnServer default delegate
      map: 'ws-echo-client-remote' 
      to: [ :request | ZnResponse ok: (ZnEntity html: ZnWebSocketEchoHandler clientHtmlRemote) ];
      map: 'ws-echo-client' 
      to: [ :request | ZnResponse ok: (ZnEntity html: ZnWebSocketEchoHandler clientHtml) ];
      map: 'ws-echo'
      to: (ZnWebSocketDelegate map: 'ws-echo' to: ZnWebSocketEchoHandler new).

Now, you can try the following URLs:

- <http://localhost:1701/ws-echo-client-remote>
- <http://localhost:1701/ws-echo-client-remote>

The first one will connect to ws://echo.websocket.org as a reference,
the second one will connect to our implementation at ws://localhost:1701/ws-echo.

Another simple example is available in ZnWebSocketStatusHandler where 
a couple of Smalltalk image statistics are emitted every second for an efficient live view in your browser. 
In this scenario, the server accepts each incoming web socket connection and starts streaming to it,
not interested in any incoming messages. Here is the core loop:

    ZnWebSocketStatusHandler>>value: webSocket
      [ 
        self crLog: 'Started status streaming'.
        [ 
          webSocket sendMessage: self status.
          1 second asDelay wait.
          webSocket isConnected ] whileTrue ] 
        on: ConnectionClosed 
        do: [ self crLog: 'Ignoring connection close' ].
      self crLog: 'Stopping status streaming' 

The last example, ZnWebSocketChatroomHandler, implements the core logic of a chatroom: 
clients can send messages to the server who distributes them to all connected clients.
In this case, the handler has to manage a collection of all connected client web sockets.
Here is the core loop:

    ZnWebSocketChatroomHandler>>value: webSocket
      [
        self register: webSocket.
        webSocket runWith: [ :message |
          self crLog: 'Received message: ', message printString.
          self distributeMessage: message ] ] 
        on: ConnectionClosed 
        do: [
          self crLog: 'Connection close, cleaning up'.
          self unregister: webSocket ]

Distrubuting the message is as simple as (ignoring some details):

    ZnWebSocketChatroomHandler>>distributeMessage: message
      clientWebSockets do: [ :each |
        each sendMessage: message ].

Here is code to setup all examples:

    ZnServer startDefaultOn: 1701.
    ZnServer default logToTranscript.
    ZnServer default delegate
      map: 'ws-echo-client-remote' 
      to: [ :request | ZnResponse ok: (ZnEntity html: ZnWebSocketEchoHandler clientHtmlRemote) ];
      map: 'ws-echo-client' 
      to: [ :request | ZnResponse ok: (ZnEntity html: ZnWebSocketEchoHandler clientHtml) ];
      map: 'ws-echo'
      to: (ZnWebSocketDelegate map: 'ws-echo' to: ZnWebSocketEchoHandler new);
      map: 'ws-chatroom-client' 
      to: [ :request | ZnResponse ok: (ZnEntity html: ZnWebSocketChatroomHandler clientHtml) ];
      map: 'ws-chatroom'
      to: (ZnWebSocketDelegate map: 'ws-chatroom' to: ZnWebSocketChatroomHandler new);
      map: 'ws-status-client' 
      to: [ :request | ZnResponse ok: (ZnEntity html: ZnWebSocketStatusHandler clientHtml) ];
      map: 'ws-status'
      to: (ZnWebSocketDelegate map: 'ws-chatroom' to: ZnWebSocketStatusHandler new).

Vist any of the following URLs:

- <http://localhost:1701/ws-echo-client>
- <http://localhost:1701/ws-status-client>
- <http://localhost:1701/ws-chat-client>

Inside your Smalltalk image, you can also send chat messages, much like a moderator:

    (ZnServer default delegate prefixMap at: 'ws-chatroom')
      handler distributeMessage: 'moderator>>No trolling please!'.


## A Quick Tour of the Implementation


All code resides in the 'Zinc-WebSocket-Core' package. 
The wire level protocol, the encoding and decoding of frames can be found in ZnWebSocketFrame.
The key methods are #writeOn: and #readFrom: as well as the instance creation protocol.
Together with the testing protocol and #printOn: these should give enough information to understand the implementation.

ZnWebSocket implements the protocol above that, either from a server or a client prespective.
Client side setup can be found on the class side of ZnWebSocket.
Server side handling of the setup is implemented in ZnWebSocketDelegate.
The key methods are #readMessage and #readFrame, sending is quite simple.

Two exceptions, ZnWebSocketFailed and ZnWebSocketClosed 
and a shared ZnWebSocketUtils class round out the core code.


The implementation of Zinc WebSockets as an addon to Zinc HTTP Components
was made possibe in part through financial backing by Andy Burnett of 
[Knowinnovation Inc.](http://www.knowinnovation.com) and [ESUG](http://www.esug.org).
