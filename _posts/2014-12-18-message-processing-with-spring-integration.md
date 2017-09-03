---
layout: post
title: "Message Processing with Spring Integration"
date: 2014-12-18
---

Spring Integration provides an extension of the Spring framework to support the well-known Enterprise Integration Patterns. It enables lightweight messaging within Spring-based applications and supports integration with external systems. One of the most important goals of Spring Integration is to provide a simple model for building maintainable and testable enterprise integration solutions.

**Main Components**

**Message :** It is a generic wrapper for any Java object combined with metadata used by the framework while handling that object. It consists of a payload and header(s). Message payload can be any Java Object and Message header is a String/Object Map covering header name and value. MessageBuilder is used to create messages covering payload and headers as follows :

```java
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;

Message message = MessageBuilder.withPayload("Message Payload")
                .setHeader("Message_Header1", "Message_Header1_Value")
                .setHeader("Message_Header2", "Message_Header2_Value")
                .build();
```

**Message Channel :** A message channel is the component through which messages are moved so it can be thought as a pipe between message producer and consumer. A Producer sends the message to a channel, and a consumer receives the message from the channel. A Message Channel may follow either Point-to-Point or Publish/Subscribe semantics. With a Point-to-Point channel, at most one consumer can receive each message sent to the channel. With Publish/Subscribe channels, multiple subscribers can receive each Message sent to the channel. Spring Integration supports both of these.

<p>In this sample project, Direct channel and null-channel are used. Direct channel is the default channel type within Spring Integration and simplest point-to-point channel option. Null Channel is a dummy message channel to be used mainly for testing and debugging. It does not send the message from sender to receiver but its send method always returns true and receive method returns null value. In addition to DirectChannel and NullChannel, Spring Integration provides different Message Channel Implementations such as PublishSubscribeChannel, QueueChannel, PriorityChannel, RendezvousChannel, ExecutorChannel and ScopedChannel.

**Message Endpoint :** A message endpoint isolates application code from the infrastructure. In other words, it is an abstraction layer between the application code and the messaging framework.

**Main Message Endpoints**

**Transformer :** A Message Transformer is responsible for converting a Message’s content or structure and returning the modified Message. For example : it may be used to transform message payload from one format to another or to modify message header values.
**Filter :** A Message Filter determines whether the message should be passed to the message channel.
**Router :** A Message Router decides what channel(s) should receive the Message next if it is available.
**Splitter :** A Splitter breaks an incoming message into multiple messages and send them to the appropriate channel.
**Aggregator :** An Aggregator combines multiple messages into a single message.
**Service Activator :** A Service Activator is a generic endpoint for connecting a service instance to the messaging system.
**Channel Adapter :** A Channel Adapter is an endpoint that connects a Message Channel to external system. Channel Adapters may be either inbound or outbound. An inbound Channel Adapter endpoint connects a external system to a MessageChannel. An outbound Channel Adapter endpoint connects a MessageChannel to a external system.
**Messaging Gateway :** A gateway is an entry point for the messaging system and hides the messaging API from external system. It is bidirectional by covering request and reply channels.

<p>Also Spring Integration provides various Channel Adapters and Messaging Gateways (for AMQP, File, Redis, Gemfire, Http, Jdbc, JPA, JMS, RMI, Stream etc..) to support Message-based communication with external systems. Please visit Spring Integration Reference documentation for the detailed information.

<p>The following sample Cargo messaging implementation shows basic message endpoints’ behaviours for understanding easily. Cargo messaging system listens cargo messages from external system by using a CargoGateway Interface. Received cargo messages are processed by using CargoSplitter, CargoFilter, CargoRouter, CargoTransformer MessageEndpoints. After then, processed successful domestic and international cargo messages are sent to CargoServiceActivator.

<p>Cargo Messaging System’ s Spring Integration Flow is as follows :</p>

![_config.yml]({{ site.baseurl }}/images/otv_si.jpeg)

<p>Let us take a look sample cargo messaging implementation.</p>


**Used Technologies**

* JDK 1.8.0_25
* Spring 4.1.2
* Spring Integration 4.1.0
* Maven 3.2.2
* Ubuntu 14.04

<p>Project Hierarchy is as follows :</p>

![_config.yml]({{ site.baseurl }}/images/otv_si3.jpeg)

**STEP 1 : Dependencies**

Dependencies are added to Maven pom.xml.
```xml
<properties>
       <spring.version>4.1.2.RELEASE</spring.version>
       <spring.integration.version>4.1.0.RELEASE</spring.integration.version>
   </properties>

   <dependencies>
       <!-- Spring 4 dependencies -->
       <dependency>
           <groupId>org.springframework</groupId>
           <artifactId>spring-context</artifactId>
           <version>${spring.version}</version>
       </dependency>
       
       <!-- Spring Integration dependencies -->
       <dependency>
           <groupId>org.springframework.integration</groupId>
           <artifactId>spring-integration-core</artifactId>
           <version>${spring.integration.version}</version>
       </dependency>
</dependencies>
```

**STEP 2 : Cargo Builder**

CargoBuilder is created to build Cargo requests.

```java
public class Cargo {

    public enum ShippingType {
        DOMESTIC, INTERNATIONAL
    }

    private final long trackingId;
    private final String receiverName;
    private final String deliveryAddress;
    private final double weight;
    private final String description;
    private final ShippingType shippingType;
    private final int deliveryDayCommitment;
    private final int region;

    private Cargo(CargoBuilder cargoBuilder) {
        this.trackingId = cargoBuilder.trackingId;
        this.receiverName = cargoBuilder.receiverName;
        this.deliveryAddress = cargoBuilder.deliveryAddress;
        this.weight = cargoBuilder.weight;
        this.description = cargoBuilder.description;
        this.shippingType = cargoBuilder.shippingType;
        this.deliveryDayCommitment = cargoBuilder.deliveryDayCommitment;
        this.region = cargoBuilder.region;
    }

    // Getter methods...
    
    @Override
    public String toString() {
        return "Cargo [trackingId=" + trackingId + ", receiverName="
                + receiverName + ", deliveryAddress=" + deliveryAddress
                + ", weight=" + weight + ", description=" + description
                + ", shippingType=" + shippingType + ", deliveryDayCommitment="
                + deliveryDayCommitment + ", region=" + region + "]";
    }

    public static class CargoBuilder {
        
        private final long trackingId;
        private final String receiverName;
        private final String deliveryAddress;
        private final double weight;
        private final ShippingType shippingType;
        private int deliveryDayCommitment;
        private int region;
        private String description;
        
        public CargoBuilder(long trackingId, String receiverName,
                            String deliveryAddress, double weight, 
                            ShippingType shippingType) {
            this.trackingId = trackingId;
            this.receiverName = receiverName;
            this.deliveryAddress = deliveryAddress;
            this.weight = weight;
            this.shippingType = shippingType;
        }

        public CargoBuilder setDeliveryDayCommitment(int deliveryDayCommitment) {
            this.deliveryDayCommitment = deliveryDayCommitment;
            return this;
        }

        public CargoBuilder setDescription(String description) {
            this.description = description;
            return this;
        }
        
        public CargoBuilder setRegion(int region) {
            this.region = region;
            return this;
        }

        public Cargo build() {
            Cargo cargo = new Cargo(this);
            if ((ShippingType.DOMESTIC == cargo.getShippingType()) && (cargo.getRegion() <= 0 || cargo.getRegion() > 4)) {
                throw new IllegalStateException("Region is invalid! Cargo Tracking Id : " + cargo.getTrackingId());
            }
            
            return cargo;
        }
        
    }
```

**STEP 3 : Cargo Message**

CargoMessage is the parent class of Domestic and International Cargo Messages.

```java
public class CargoMessage {

    private final Cargo cargo;

    public CargoMessage(Cargo cargo) {
        this.cargo = cargo;
    }

    public Cargo getCargo() {
        return cargo;
    }

    @Override
    public String toString() {
        return cargo.toString();
    }
}
```

**STEP 4 : Domestic Cargo Message**

DomesticCargoMessage Class models domestic cargo messages.

```java
public class DomesticCargoMessage extends CargoMessage {
    
    public enum Region {
        
        NORTH(1), SOUTH(2), EAST(3), WEST(4);
        
        private int value;

        private Region(int value) {
            this.value = value;
        }

        public static Region fromValue(int value) {
            return Arrays.stream(Region.values())
                            .filter(region -> region.value == value)
                            .findFirst()
                            .get();
        }
    }
    
    private final Region region; 

    public DomesticCargoMessage(Cargo cargo, Region region) {
        super(cargo);
        this.region = region;
    }

    public Region getRegion() {
        return region;
    }

    @Override
    public String toString() {
        return "DomesticCargoMessage [cargo=" + super.toString() + ", region=" + region + "]";
    }

}
```

**STEP 5 : International Cargo Message**

InternationalCargoMessage Class models international cargo messages.

```java
public class InternationalCargoMessage extends CargoMessage {
    
    public enum DeliveryOption {
        NEXT_FLIGHT, PRIORITY, ECONOMY, STANDART
    }
    
    private final DeliveryOption deliveryOption;
    
    public InternationalCargoMessage(Cargo cargo, DeliveryOption deliveryOption) {
        super(cargo);
        this.deliveryOption = deliveryOption;
    }

    public DeliveryOption getDeliveryOption() {
        return deliveryOption;
    }

    @Override
    public String toString() {
        return "InternationalCargoMessage [cargo=" + super.toString() + ", deliveryOption=" + deliveryOption + "]";
    }

}
```

**STEP 6 : Application Configuration**

AppConfiguration is configuration provider class for Spring Container. It creates Message Channels and registers to Spring BeanFactory. Also **@EnableIntegration** enables imported spring integration configuration and **@IntegrationComponentScan** scans Spring Integration specific components. Both of them came with Spring Integration 4.0.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.integration.annotation.IntegrationComponentScan;
import org.springframework.integration.channel.DirectChannel;
import org.springframework.integration.config.EnableIntegration;
import org.springframework.messaging.MessageChannel;

@Configuration
@ComponentScan("com.onlinetechvision.integration")
@EnableIntegration
@IntegrationComponentScan("com.onlinetechvision.integration")
public class AppConfiguration {

    /**
     * Creates a new cargoGWDefaultRequest Channel and registers to BeanFactory.
     *
     * @return direct channel
     */
    @Bean
    public MessageChannel cargoGWDefaultRequestChannel() {
        return new DirectChannel();
    }

    /**
     * Creates a new cargoSplitterOutput Channel and registers to BeanFactory.
     *
     * @return direct channel
     */
    @Bean
    public MessageChannel cargoSplitterOutputChannel() {
        return new DirectChannel();
    }

    /**
     * Creates a new cargoFilterOutput Channel and registers to BeanFactory.
     *
     * @return direct channel
     */
    @Bean
    public MessageChannel cargoFilterOutputChannel() {
        return new DirectChannel();
    }

    /**
     * Creates a new cargoRouterDomesticOutput Channel and registers to BeanFactory.
     *
     * @return direct channel
     */
    @Bean
    public MessageChannel cargoRouterDomesticOutputChannel() {
        return new DirectChannel();
    }

    /**
     * Creates a new cargoRouterInternationalOutput Channel and registers to BeanFactory.
     *
     * @return direct channel
     */
    @Bean
    public MessageChannel cargoRouterInternationalOutputChannel() {
        return new DirectChannel();
    }

    /**
     * Creates a new cargoTransformerOutput Channel and registers to BeanFactory.
     *
     * @return direct channel
     */
    @Bean
    public MessageChannel cargoTransformerOutputChannel() {
        return new DirectChannel();
    }

}
```

**STEP 7 : Messaging Gateway**

CargoGateway Interface exposes domain-specific method to the application. In other words, it provides an application access to the messaging system. Also **@MessagingGateway** came with Spring Integration 4.0 and simplifies gateway creation in messaging system. Its default request channel is **cargoGWDefaultRequestChannel**.

```java
import java.util.List;

import org.springframework.integration.annotation.Gateway;
import org.springframework.integration.annotation.MessagingGateway;
import org.springframework.messaging.Message;

import com.onlinetechvision.model.Cargo;

@MessagingGateway(name = "cargoGateway", 
                    defaultRequestChannel = "cargoGWDefaultRequestChannel")
public interface ICargoGateway {

    /**
     * Processes Cargo Request
     *
     * @param message SI Message covering Cargo List payload and Batch Cargo Id header.
     * @return operation result
     */
    @Gateway
    void processCargoRequest(Message<List<Cargo>> message);
}
```

**STEP 8 : Messaging Splitter**

CargoSplitter listens **cargoGWDefaultRequestChannel** channel and breaks incoming Cargo List into Cargo messages. Cargo messages are sent to **cargoSplitterOutputChannel**.

```java
import java.util.List;

import org.springframework.integration.annotation.MessageEndpoint;
import org.springframework.integration.annotation.Splitter;
import org.springframework.messaging.Message;

import com.onlinetechvision.model.Cargo;

@MessageEndpoint
public class CargoSplitter {

    /**
     * Splits Cargo List to Cargo message(s)
     *
     * @param message SI Message covering Cargo List payload and Batch Cargo Id header.
     * @return cargo list
     */
    @Splitter(inputChannel = "cargoGWDefaultRequestChannel", 
                outputChannel = "cargoSplitterOutputChannel")
    public List<Cargo> splitCargoList(Message<List<Cargo>> message) {
        return message.getPayload();
    }
}
```

**STEP 9 : Messaging Filter**

CargoFilter determines whether the message should be passed to the message channel. It listens **cargoSplitterOutputChannel** channel and filters cargo messages exceeding weight limit. If Cargo message is lower than weight limit, it is sent to **cargoFilterOutputChannel** channel. If Cargo message is higher than weight limit, it is sent to **cargoFilterDiscardChannel** channel.

```java
import org.springframework.integration.annotation.Filter;
import org.springframework.integration.annotation.MessageEndpoint;

import com.onlinetechvision.model.Cargo;


@MessageEndpoint
public class CargoFilter {

    private static final long CARGO_WEIGHT_LIMIT = 1_000;
    
    /**
     * Checks weight of cargo and filters if it exceeds limit.
     *
     * @param Cargo message
     * @return check result
     */
    @Filter(inputChannel="cargoSplitterOutputChannel", outputChannel="cargoFilterOutputChannel", discardChannel="cargoFilterDiscardChannel")
    public boolean filterIfCargoWeightExceedsLimit(Cargo cargo) {
        return cargo.getWeight() <= CARGO_WEIGHT_LIMIT;
    }
}
```

**STEP 10 : Discarded Cargo Message Listener**

DiscardedCargoMessageListener listens cargoFilterDiscard Channel and handles Cargo messages discarded by CargoFilter.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.integration.annotation.MessageEndpoint;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.handler.annotation.Header;

import com.onlinetechvision.model.Cargo;


@MessageEndpoint
public class DiscardedCargoMessageListener {

	private final Logger logger = LoggerFactory.getLogger(DiscardedCargoMessageListener.class);
	
	/**
     * Handles discarded domestic and international cargo request(s) and logs.
     *
     * @param cargo domestic/international cargo message
     * @param batchId message header shows cargo batch id
     */
	@ServiceActivator(inputChannel = "cargoFilterDiscardChannel")
	public void handleDiscardedCargo(Cargo cargo, @Header("CARGO_BATCH_ID") long batchId) {
		logger.debug("Message in Batch[" + batchId + "] is received with Discarded payload : " + cargo);
	}

}
```

**STEP 11 : Messaging Router**

CargoRouter determines what channel(s) should receive the message next if it is available. It listens cargoFilterOutputChannel channel and returns related channel name in the light of cargo shipping type. In other words, it routes incoming cargo messages to domestic(cargoRouterDomesticOutputChannel) or international(cargoRouterInternationalOutputChannel) cargo channels. Also if shipping type is not set, nullChannel is returned. nullChannel is a dummy message channel to be used mainly for testing and debugging. It does not send the message from sender to receiver but its send method always returns true and receive method returns null value.

```java
import org.springframework.integration.annotation.MessageEndpoint;
import org.springframework.integration.annotation.Router;

import com.onlinetechvision.model.Cargo;
import com.onlinetechvision.model.Cargo.ShippingType;

@MessageEndpoint
public class CargoRouter {
    
    /**
     * Determines cargo request' s channel in the light of shipping type.
     *
     * @param Cargo message
     * @return channel name
     */
    @Router(inputChannel="cargoFilterOutputChannel")
    public String route(Cargo cargo) {
        if(cargo.getShippingType() == ShippingType.DOMESTIC) {
            return "cargoRouterDomesticOutputChannel";
        } else if(cargo.getShippingType() == ShippingType.INTERNATIONAL) {
            return "cargoRouterInternationalOutputChannel";
        } 
        
        return "nullChannel"; 
    }
    
}
```

**STEP 12 : Messaging Transformer**

CargoTransformer listens cargoRouterDomesticOutputChannel & cargoRouterInternationalOutputChannel and transforms incoming Cargo requests to Domestic and International Cargo messages. After then, it sends them to cargoTransformerOutputChannel channel.

```java
import org.springframework.integration.annotation.MessageEndpoint;
import org.springframework.integration.annotation.Transformer;

import com.onlinetechvision.model.Cargo;
import com.onlinetechvision.model.DomesticCargoMessage;
import com.onlinetechvision.model.DomesticCargoMessage.Region;
import com.onlinetechvision.model.InternationalCargoMessage;
import com.onlinetechvision.model.InternationalCargoMessage.DeliveryOption;


@MessageEndpoint
public class CargoTransformer {

    /**
     * Transforms Cargo request to Domestic Cargo obj.
     *
     * @param cargo
     *            request
     * @return Domestic Cargo obj
     */
    @Transformer(inputChannel = "cargoRouterDomesticOutputChannel", 
                    outputChannel = "cargoTransformerOutputChannel")
    public DomesticCargoMessage transformDomesticCargo(Cargo cargo) {
        return new DomesticCargoMessage(cargo, Region.fromValue(cargo.getRegion()));
    }

    /**
     * Transforms Cargo request to International Cargo obj.
     *
     * @param cargo
     *            request
     * @return International Cargo obj
     */
    @Transformer(inputChannel = "cargoRouterInternationalOutputChannel", 
                    outputChannel = "cargoTransformerOutputChannel")
    public InternationalCargoMessage transformInternationalCargo(Cargo cargo) {
        return new InternationalCargoMessage(cargo, getDeliveryOption(cargo.getDeliveryDayCommitment()));
    }
    
    /**
     * Get delivery option by delivery day commitment.
     *
     * @param deliveryDayCommitment delivery day commitment
     * @return delivery option
     */
    private DeliveryOption getDeliveryOption(int deliveryDayCommitment) {
        if (deliveryDayCommitment == 1) {
            return DeliveryOption.NEXT_FLIGHT;
        } else if (deliveryDayCommitment == 2) {
            return DeliveryOption.PRIORITY;
        } else if (deliveryDayCommitment > 2 && deliveryDayCommitment < 5) {
            return DeliveryOption.ECONOMY;
        } else {
            return DeliveryOption.STANDART;
        }
    }

}
```

**STEP 13 : Messaging Service Activator**

CargoServiceActivator is a generic endpoint for connecting service instance to the messaging system. It listens cargoTransformerOutputChannel channel and gets processed domestic and international cargo messages and logs.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.integration.annotation.MessageEndpoint;
import org.springframework.integration.annotation.ServiceActivator;
import org.springframework.messaging.handler.annotation.Header;

import com.onlinetechvision.model.CargoMessage;


@MessageEndpoint
public class CargoServiceActivator {

    private final Logger logger = LoggerFactory.getLogger(CargoServiceActivator.class);
    
    /**
     * Gets processed domestic and international cargo request(s) and logs.
     *
     * @param cargoMessage domestic/international cargo message
     * @param batchId message header shows cargo batch id
     */
    @ServiceActivator(inputChannel = "cargoTransformerOutputChannel")
    public void getCargo(CargoMessage cargoMessage, @Header("CARGO_BATCH_ID") long batchId) {
        logger.debug("Message in Batch[" + batchId + "] is received with payload : " + cargoMessage);
    }

}
```

**STEP 14 : Application**

Application Class is created to run the application. It initializes application context and sends cargo requests to messaging system.

```java
import java.util.Arrays;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.AnnotationConfigApplicationContext;
import org.springframework.messaging.support.MessageBuilder;

import com.onlinetechvision.integration.ICargoGateway;
import com.onlinetechvision.model.Cargo;
import com.onlinetechvision.model.Cargo.ShippingType;


public class Application {

    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfiguration.class);
        ICargoGateway orderGateway = ctx.getBean(ICargoGateway.class);
        
        getCargoBatchMap().forEach(
            (batchId, cargoList) -> orderGateway.processCargoRequest(MessageBuilder
                                                                        .withPayload(cargoList)
                                                                        .setHeader("CARGO_BATCH_ID", batchId)
                                                                        .build()));
    }
    
    /**
     * Creates a sample cargo batch map covering multiple batches and returns.
     *
     * @return cargo batch map
     */
    private static Map<Integer, List<Cargo>> getCargoBatchMap() {
        Map<Integer, List<Cargo>> cargoBatchMap = new HashMap<>();
        
        cargoBatchMap.put(1, Arrays.asList(
                
                new Cargo.CargoBuilder(1, "Receiver_Name1", "Address1", 0.5, ShippingType.DOMESTIC)
                            .setRegion(1).setDescription("Radio").build(),
                //Second cargo is filtered due to weight limit          
                new Cargo.CargoBuilder(2, "Receiver_Name2", "Address2", 2_000, ShippingType.INTERNATIONAL)
                            .setDeliveryDayCommitment(3).setDescription("Furniture").build(),
                new Cargo.CargoBuilder(3, "Receiver_Name3", "Address3", 5, ShippingType.INTERNATIONAL)
                            .setDeliveryDayCommitment(2).setDescription("TV").build(),
                //Fourth cargo is not processed due to no shipping type found           
                new Cargo.CargoBuilder(4, "Receiver_Name4", "Address4", 8, null)
                            .setDeliveryDayCommitment(2).setDescription("Chair").build()));
                                        
        cargoBatchMap.put(2, Arrays.asList(
                //Fifth cargo is filtered due to weight limit
                new Cargo.CargoBuilder(5, "Receiver_Name5", "Address5", 1_200, ShippingType.DOMESTIC)
                            .setRegion(2).setDescription("Refrigerator").build(),
                new Cargo.CargoBuilder(6, "Receiver_Name6", "Address6", 20, ShippingType.DOMESTIC)
                            .setRegion(3).setDescription("Table").build(),
                //Seventh cargo is not processed due to no shipping type found
                new Cargo.CargoBuilder(7, "Receiver_Name7", "Address7", 5, null)
                            .setDeliveryDayCommitment(1).setDescription("TV").build()));
                
        cargoBatchMap.put(3, Arrays.asList(
                new Cargo.CargoBuilder(8, "Receiver_Name8", "Address8", 200, ShippingType.DOMESTIC)
                            .setRegion(2).setDescription("Washing Machine").build(),
                new Cargo.CargoBuilder(9, "Receiver_Name9", "Address9", 4.75, ShippingType.INTERNATIONAL)
                            .setDeliveryDayCommitment(1).setDescription("Document").build()));
        
        return Collections.unmodifiableMap(cargoBatchMap);
    }
    
}
```

**STEP 15 : Build Project**

<p>
Cargo requests’ operational results are as follows :
Cargo 1 : is sent to service activator successfully.
Cargo 2 : is filtered due to weight limit.
Cargo 3 : is sent to service activator successfully.
Cargo 4 : is not processed due to no shipping type.
Cargo 5 : is filtered due to weight limit.
Cargo 6 : is sent to service activator successfully.
Cargo 7 : is not processed due to no shipping type.
Cargo 8 : is sent to service activator successfully.
Cargo 9 : is sent to service activator successfully.

After the project is built and run, the following console output logs will be seen :
</p>

2014-12-09 23:43:51 [main] DEBUG c.o.i.CargoServiceActivator - Message in Batch[1] is received with payload : DomesticCargoMessage [cargo=Cargo [trackingId=1, receiverName=Receiver_Name1, deliveryAddress=Address1, weight=0.5, description=Radio, shippingType=DOMESTIC, deliveryDayCommitment=0, region=1], region=NORTH]
2014-12-09 23:43:51 [main] DEBUG c.o.i.DiscardedCargoMessageListener - Message in Batch[1] is received with Discarded payload : Cargo [trackingId=2, receiverName=Receiver_Name2, deliveryAddress=Address2, weight=2000.0, description=Furniture, shippingType=INTERNATIONAL, deliveryDayCommitment=3, region=0]
2014-12-09 23:43:51 [main] DEBUG c.o.i.CargoServiceActivator - Message in Batch[1] is received with payload : InternationalCargoMessage [cargo=Cargo [trackingId=3, receiverName=Receiver_Name3, deliveryAddress=Address3, weight=5.0, description=TV, shippingType=INTERNATIONAL, deliveryDayCommitment=2, region=0], deliveryOption=PRIORITY]
2014-12-09 23:43:51 [main] DEBUG c.o.i.DiscardedCargoMessageListener - Message in Batch[2] is received with Discarded payload : Cargo [trackingId=5, receiverName=Receiver_Name5, deliveryAddress=Address5, weight=1200.0, description=Refrigerator, shippingType=DOMESTIC, deliveryDayCommitment=0, region=2]
2014-12-09 23:43:51 [main] DEBUG c.o.i.CargoServiceActivator - Message in Batch[2] is received with payload : DomesticCargoMessage [cargo=Cargo [trackingId=6, receiverName=Receiver_Name6, deliveryAddress=Address6, weight=20.0, description=Table, shippingType=DOMESTIC, deliveryDayCommitment=0, region=3], region=EAST]
2014-12-09 23:43:51 [main] DEBUG c.o.i.CargoServiceActivator - Message in Batch[3] is received with payload : DomesticCargoMessage [cargo=Cargo [trackingId=8, receiverName=Receiver_Name8, deliveryAddress=Address8, weight=200.0, description=Washing Machine, shippingType=DOMESTIC, deliveryDayCommitment=0, region=2], region=SOUTH]
2014-12-09 23:43:51 [main] DEBUG c.o.i.CargoServiceActivator - Message in Batch[3] is received with payload : InternationalCargoMessage [cargo=Cargo [trackingId=9, receiverName=Receiver_Name9, deliveryAddress=Address9, weight=4.75, description=Document, shippingType=INTERNATIONAL, deliveryDayCommitment=1, region=0], deliveryOption=NEXT_FLIGHT]

**Source Code**

Source Code is available on Github

**References**

* Enterprise Integration Patterns
* Spring Integration Reference Manual
* Spring Integration 4.1.0.RELEASE API
* Pro Spring Integration
* Spring Integration 3.0.2 and 4.0 Milestone 4 Released