# Stamp

*Sven Van Caekenberghe*

*August 2013*

*(Work in progress)*

Stamp is an implementation of [STOMP (Simple (or Streaming) Text Oriented Message Protocol)](http://en.wikipedia.org/wiki/Streaming_Text_Oriented_Messaging_Protocol) for [Pharo](http://www.pharo.org), a protocol to interact with message-oriented middleware (MOM).

More specifically, Stamp implements [STOMP 1.2](http://stomp.github.io/stomp-specification-1.2.html) and was tested against [RabbitMQ 3.1](http://www.rabbitmq.com). Other message-oriented middleware implementations accessible through STOMP include [Apache ActiveMQ](http://activemq.apache.org), [Glassfish Open MQ](http://mq.java.net) and [Fuse Message Broker based on Active MQ](http://fusesource.com/products/enterprise-activemq/) - but these have not yet been tested.

[Messaging middleware](http://en.wikipedia.org/wiki/Message-oriented_middleware) is an important technology for building scaleable and flexible enterprise software architectures.

## Installation

The MIT licensed [source code for Stamp](http://www.smalltalkhub.com/#!/~SvenVanCaekenberghe/Stamp) lives on [SmalltalkHub](http://www.smalltalkhub.com) and can be loaded using Metacello.

    Gofer it
      smalltalkhubUser: 'SvenVanCaekenberghe' project: 'Stamp';
      configurationOf: 'Stamp';
      loadVersion: #bleedingEdge.

Unit tests and benchmarks are included.

## Usage

To be able to use Stamp, you need a running [RabbitMQ](http://www.rabbitmq.com/download.html) instance with the [STOMP plugin](http://www.rabbitmq.com/stomp.html) enabled. It is helpful to also install the [management plugin](http://www.rabbitmq.com/management.html) which exports a web interface.

Whether you are sending or receiving messages, whether you are a conceptual producer or consumer, you are always a client of the messaging middleware. The key object to use is thus called StampClient.

To connect, you need to specify a host (defaults to localhost), port (defaults to 61613), login and passcode.

    | client |    client := StampClient new.    client login: 'guest'.    client passcode: 'guest'.Use the RabbitMQ web management interface to configure the default virtual host, login and passcode (implicit login/passcode can be configured as well). There is a #debug: true option to enable Transcript logging of the wire protocol.

Using #open and #close you actually connect and disconnect.

Although STOMP resembles HTTP quite a bit, it is fundamentally different. HTTP is a synchronous request/response protocol, while STOMP is asynchronous without the concept of request/response pairs.

The base concept in the STOMP wire protocol is called a frame, sent or received. Stamp has some convenience methods that hide frames, but they will show up. The implementation matches each frame type to a different class, as subclasses of StampFrame.

There is a convenience method to send a plain text message, which means that you put it on a queue.

    client sendText: 'Hello World!' to: 'test-queue'.

Another convenience method allows a client to subscribe or listen to a queue.

    client subscribeTo: 'test-queue'.

Next, you will have to actually wait for incoming messages.

    client readMessage.

Note that #readMessage will eventually time out. To listen in a convenient loop, there is a control structure helper.

    client runWith: [ :message |
      "Do something with message" ]

The block will be invoked for each incoming message while ConnectionTimedOut will be ignored by looping. To exit the loop, you should signal ConnectedClosed. In any case, at the end the client will be closed.

In general though, the convenience methods will not be enough because often extra parameters have to be set. For this, Stamp uses its frame objects.

For example, sending a message uses a StampSendFrame.

    | sendFrame |
    sendFrame := StampSendFrame new.
    sendFrame destination: 'test-queue'.
    sendFrame text: 'Hello there'.
    sendFrame persistent: true.
    sendFrame replyTo: '/temp-queue/greetings'.
    client write: sendFrame.

Similarly, subscribing is done with a StampSubscribeFrame.

    | subscribeFrame |
    subscribeFrame := StampSubscribeFrame new.
    subscribeFrame destination: 'test-queue'.
    subscribeFrame id: client nextId.
    subscribeFrame clientIndividualAck.
    client writeWithReceipt: subscribeFrame.
 
And the messages that you receive are instances of StampMessageFrame with a body and contentType, contentLength and other meta data.

The frame objects allow access to important meta data, needed to set the necessary semantics for proper message middleware usage.

In particular, the current Stamp implementation supports the following:

- ack and nack on received messages
- auto, client and client individual acknowledgments
- receipts when sending messages
- transactions
- persistent queues
- durable subscriptions
- temporary reply queues for RPC
- AMPQ headers and queues
- heartbeats
- arbitrary content types and meta data

Please refer to the unit tests for examples. In particular, #testSimpleRpc, #testSimpleRpcCounter and #testSimpleWorkQueue illustrate the main high level patterns.

Stamp is strictly single threaded by design and thus has no locks to protect itself. A single loop can handle incoming messages, sending outgoing messages as necessary.

Stamp itself is still evolving. Any feedback is welcome. 
## Acknowledgement

This project was inspired by Göran Krampe's older [StompProtocol](http://www.squeaksource.com/StompProtocol.html) implementation. After making a number changes and commits to this code base, I felt I was diverging so much and so fundamentally from the original implementation that I had to fork it. Stamp started as a toy project developed in private but  gradually matured to a point where it is reasonably complete, stands on its own and would be useful for others.