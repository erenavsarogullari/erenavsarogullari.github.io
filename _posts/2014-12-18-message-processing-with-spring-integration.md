---
layout: post
title: "Message Processing with Spring Integration"
date: 2014-12-18
---

Spring Integration provides an extension of the Spring framework to support the well-known Enterprise Integration Patterns. It enables lightweight messaging within Spring-based applications and supports integration with external systems. One of the most important goals of Spring Integration is to provide a simple model for building maintainable and testable enterprise integration solutions.

**Main Components**
<p>**Message** : It is a generic wrapper for any Java object combined with metadata used by the framework while handling that object. It consists of a payload and header(s). Message payload can be any Java Object and Message header is a String/Object Map covering header name and value. MessageBuilder is used to create messages covering payload and headers as follows :

```java
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

Message message = MessageBuilder.withPayload("Message Payload")
                .setHeader("Message_Header1", "Message_Header1_Value")
                .setHeader("Message_Header2", "Message_Header2_Value")
                .build();
```

<p>**Message Channel :** A message channel is the component through which messages are moved so it can be thought as a pipe between message producer and consumer. A Producer sends the message to a channel, and a consumer receives the message from the channel. A Message Channel may follow either Point-to-Point or Publish/Subscribe semantics. With a Point-to-Point channel, at most one consumer can receive each message sent to the channel. With Publish/Subscribe channels, multiple subscribers can receive each Message sent to the channel. Spring Integration supports both of these.

<p>In this sample project, Direct channel and null-channel are used. Direct channel is the default channel type within Spring Integration and simplest point-to-point channel option. Null Channel is a dummy message channel to be used mainly for testing and debugging. It does not send the message from sender to receiver but its send method always returns true and receive method returns null value. In addition to DirectChannel and NullChannel, Spring Integration provides different Message Channel Implementations such as PublishSubscribeChannel, QueueChannel, PriorityChannel, RendezvousChannel, ExecutorChannel and ScopedChannel.

<p>**Message Endpoint :** A message endpoint isolates application code from the infrastructure. In other words, it is an abstraction layer between the application code and the messaging framework.

**Main Message Endpoints**
<p>**Transformer :** A Message Transformer is responsible for converting a Message’s content or structure and returning the modified Message. For example : it may be used to transform message payload from one format to another or to modify message header values.
<p>**Filter :** A Message Filter determines whether the message should be passed to the message channel.
<p>**Router :** A Message Router decides what channel(s) should receive the Message next if it is available.
<p>**Splitter :** A Splitter breaks an incoming message into multiple messages and send them to the appropriate channel.
<p>**Aggregator :** An Aggregator combines multiple messages into a single message.
<p>**Service Activator :** A Service Activator is a generic endpoint for connecting a service instance to the messaging system.
<p>**Channel Adapter :** A Channel Adapter is an endpoint that connects a Message Channel to external system. Channel Adapters may be either inbound or outbound. An inbound Channel Adapter endpoint connects a external system to a MessageChannel. An outbound Channel Adapter endpoint connects a MessageChannel to a external system.
<p>**Messaging Gateway :** A gateway is an entry point for the messaging system and hides the messaging API from external system. It is bidirectional by covering request and reply channels.
<p>Also Spring Integration provides various Channel Adapters and Messaging Gateways (for AMQP, File, Redis, Gemfire, Http, Jdbc, JPA, JMS, RMI, Stream etc..) to support Message-based communication with external systems. Please visit Spring Integration Reference documentation for the detailed information.
<p>The following sample Cargo messaging implementation shows basic message endpoints’ behaviours for understanding easily. Cargo messaging system listens cargo messages from external system by using a CargoGateway Interface. Received cargo messages are processed by using CargoSplitter, CargoFilter, CargoRouter, CargoTransformer MessageEndpoints. After then, processed successful domestic and international cargo messages are sent to CargoServiceActivator.
<p>Cargo Messaging System’ s Spring Integration Flow is as follows :