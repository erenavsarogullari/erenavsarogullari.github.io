---
layout: post
title: "High Available AMQP Backed Message Channels via Spring Integration and RabbitMQ"
category: Dev
tags: [spring integration, enterprise integration patterns, spring, rabbitmq, amqp]
date: 2015-01-23
---

<p>Spring Integration message channels store messages in memory by default. This is because memory is fast, easy to implement and it does not create extra network cost. However, in some cases, this can cause problem because all the messages will be lost if the application crashes or the server shuts down accidentally. For such situations, Spring Integration introduces JMS & AMQP backed message channels so the messages are stored within a JMS & AMQP broker instead of in memory.</p>
<p>Advanced Message Queuing Protocol (AMQP) is an open standard for messaging protocol. It allows applications to communicate asynchronously, reliably, and securely. RabbitMQ is an open source message broker that supports the AMQP standard. One of the most important features of RabbitMQ is highly available queues.</p>
<p>In this article, Spring Integration’ s AMQP backed point-to-point message channel approach is explained by creating two messaging nodes and a RabbitMQ cluster covering two RabbitMQ Servers. Two messaging nodes start to process Order messages by using the RabbitMQ cluster. If First Messaging Node and First RabbitMQ Server are shut down accidentally, Second Messaging Node and Second RabbitMQ Server will continue to process Order messages so potential message loosing and service interruption problems can be prevented by using high available AMQP backed channel.</p>
<p>Message Processing with Spring Integration Article is also recommended to have a look Spring Integration main components.
Spring Integration Flow of Order Messaging System is as follows:</p>

![_config.yml]({{ site.baseurl }}/images/si_flow.png)

<p>Order Lists are sent to Order Splitter’ s input channel via Order Gateway. Order Splitter splits order list to order messages and sends them to Order Process Service Activator. processChannel is a point-to-point AMQP backed message channel. It creates a ha.rabbit.channel queue managed by RabbitMQ cluster and sends order messages to ha.rabbit.channel Rabbit queue for high availability.
Let us have a look sample order messaging implementation.</p>

**Used Technologies :**
* JDK 1.8.0_25
* Spring 4.1.4
* Spring Integration 4.1.2
* RabbitMQ Server 3.4.2
* Maven 3.2.2
* Ubuntu 14.04

Project Hierarchy is as follows:

![_config.yml]({{ site.baseurl }}/images/spring_integration_amqp_support.png)

**STEP 1 : Dependencies**

Spring and Spring Integration Frameworks’ Dependencies are as follows :

```xml
<properties>
    <spring.version>4.1.4.RELEASE</spring.version>
    <spring.integration.version>4.1.2.RELEASE</spring.integration.version>
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
    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-amqp</artifactId>
        <version>${spring.integration.version}</version>
        <scope>compile</scope>
    </dependency>
</dependencies>
```

**STEP 2 : rabbitmq.config**

First RabbitMQ Server’ s config file(rabbitmq.config) is as follows. It should be put under ../rabbitmq_server-version/etc/rabbitmq/

```yaml
[
 {rabbit, [ {tcp_listeners, [5672]},
 {collect_statistics_interval, 10000},
 {heartbeat,30},
 {cluster_partition_handling, pause_minority},
 {cluster_nodes, {[ 'rabbit@master',
                    'rabbit2@master'],
                  disc}} ] },
 {rabbitmq_management, [ {http_log_dir,"/tmp/rabbit-mgmt"},{listener, [{port, 15672}]} ] },
 {rabbitmq_management_agent, [ {force_fine_statistics, true} ] }
]
```

Second RabbitMQ Server’ s rabbitmq.config file :

```yaml
[
 {rabbit, [ {tcp_listeners, [5673]},
 {collect_statistics_interval, 10000},
 {heartbeat,30},
 {cluster_partition_handling, pause_minority},
 {cluster_nodes, {[ 'rabbit@master',
                    'rabbit2@master'],
                  disc}} ] },
 {rabbitmq_management, [ {http_log_dir,"/tmp/rabbit-mgmt"},{listener, [{port, 15673}]} ] },
 {rabbitmq_management_agent, [ {force_fine_statistics, true} ] }
]
```

**STEP 3 : Integration Context**

Spring Integration Context is created as follows. Order Lists are sent to Order Splitter’ s input channel via Order Gateway. Order Splitter splits order list to order messages and sends them to Order Process Service Activator. processChannel is a point-to-point AMQP backed message channel. It creates a ha.rabbit.channel queue managed by RabbitMQ cluster and sends order messages to ha.rabbit.channel RabbitMQ queue for high availability.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns:int="http://www.springframework.org/schema/integration"
  xmlns:int-amqp="http://www.springframework.org/schema/integration/amqp"
  xmlns:rabbit="http://www.springframework.org/schema/rabbit"
  xmlns:context="http://www.springframework.org/schema/context"
  xsi:schemaLocation="http://www.springframework.org/schema/beans
    http://www.springframework.org/schema/beans/spring-beans.xsd
    http://www.springframework.org/schema/integration
    http://www.springframework.org/schema/integration/spring-integration.xsd
    http://www.springframework.org/schema/integration/amqp 
    http://www.springframework.org/schema/integration/amqp/spring-integration-amqp.xsd
    http://www.springframework.org/schema/rabbit 
    http://www.springframework.org/schema/rabbit/spring-rabbit.xsd
    http://www.springframework.org/schema/context 
  http://www.springframework.org/schema/context/spring-context.xsd">

  <!-- Configuration for Component Scan -->
  <context:component-scan base-package="com.onlinetechvision" />
  
  <context:property-placeholder location="classpath*:rabbitmq.properties"/>
  
  <int:channel id="inputChannel"/>
  
  <int:gateway id="orderGateway" service-interface="com.onlinetechvision.integration.OrderGateway" default-request-channel="inputChannel" />
  
  <int-amqp:channel id="processChannel"
    connection-factory="connectionFactory" 
    message-driven="true"
    queue-name="ha.rabbit.channel" />

  <!-- RabbitMQ Connection Factory -->
  <rabbit:connection-factory id="connectionFactory"
        addresses="${rabbitmq.addresses}" 
        username="${rabbitmq.username}"
        password="${rabbitmq.password}" />
        
  <int:splitter id="orderSplitter" input-channel="inputChannel" output-channel="processChannel" />
  
  <int:service-activator input-channel="processChannel" ref="orderProcessService" method="process" />
  
</beans>
```

**STEP 4 : rabbitmq.properties**

rabbitmq.properties is created as follows. If first RabbitMQ server is shut down accidentally, Second RabbitMQ will continue to listen the Order messages.

```yaml
rabbitmq.addresses=localhost:5672,localhost:5673
rabbitmq.username=guest
rabbitmq.password=guest
```

**STEP 5 : Order Model**

Order Bean models order messages.

```java
import java.io.Serializable;

public class Order implements Serializable {

    private static final long serialVersionUID = -2138235868650860555L;
    private int id;
    private String name;

    public Order(int id, String name) {
        this.id = id;
        this.name = name;
    }

    //Getter and Setter Methods...

    @Override
    public String toString() {
        return "Order [id=" + id + ", name=" + name + "]";
    }

}
```

**STEP 6 : OrderGateway**

OrderGateway Interface provides an application access to the Order messaging system. Its default request channel is inputChannel.

```java
import java.util.List;
import org.springframework.messaging.Message;
import com.onlinetechvision.model.Order;

public interface OrderGateway {

  /**
     * Processes Order Request
     *
     * @param message SI Message covering Order payload.
     */
  void processOrderRequest(Message<List<Order>> message);
}
```

**STEP 7 : OrderSplitter**

OrderSplitter listens inputChannel and breaks incoming Order List into Order messages. Order messages are sent to AMQP backed processChannel.

```java
import java.util.List;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Component;
import com.onlinetechvision.model.Order;

@Component("orderSplitter")
public class OrderSplitter {

    /**
     * Splits Order List to Order message(s)
     *
     * @param message SI Message covering Order List payload.
     * @return order list
     */
    public List<Order> splitOrderList(Message<List<Order>> message) {
        return message.getPayload();
    }
}
```

**STEP 8 : ProcessService**

Generic Process Service Interface exposes process service functionality to messaging system.

```java
import org.springframework.messaging.Message;

public interface ProcessService<T> {

    /**
     * Processes incoming message(s)
     *
     * @param message SI Message.
     */
    void process(Message<T> message);
    
}
```

**STEP 9 : OrderProcessService**

Order Process Service Activator listens AMQP backed processChannel and logs incoming Order messages. Sleep is added to fill ha.rabbit.channel RabbitMQ queue.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.messaging.Message;
import org.springframework.stereotype.Component;
import com.onlinetechvision.model.Order;

@Component("orderProcessService")
public class OrderProcessService implements ProcessService<Order> {
  
  private final Logger logger = LoggerFactory.getLogger(OrderProcessService.class);
  private final static long SLEEP_DURATION = 1_000;
  
  @Override
  public void process(Message<Order> message) {
    try {
      Thread.sleep(SLEEP_DURATION);
    } catch (InterruptedException e) {
      Thread.currentThread().interrupt();
    }
    logger.debug("Node 1 - Received Message : " + message.getPayload());
  }
  
}
```

**STEP 10 : Application**

Application Class runs the application by initializing application context and sends order messages to messaging system. Just first messaging node creates Order messages and two messaging nodes process them. Please find first and second messaging nodes’ Application Class as follows :

First Messaging Node’ s Application Class :

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Executors;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import com.onlinetechvision.integration.OrderGateway;
import com.onlinetechvision.model.Order;

public class Application {

  private final static int MESSAGE_LIMIT = 1_000;
  private final static int ORDER_LIST_SIZE = 10;
  private final static long SLEEP_DURATION = 50;
  private static OrderGateway orderGateway;
  
  /**
     * Starts the application
     *
     * @param  String[] args
     *
     */
  public static void main(String[] args) {
    ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
    orderGateway = context.getBean(OrderGateway.class);
    
    Executors.newSingleThreadExecutor().execute(new Runnable() {
      @Override
      public void run() {
        try {
          int firstIndex = 0, lastIndex = ORDER_LIST_SIZE;
          while(lastIndex <= MESSAGE_LIMIT) {
            Message<List<Order>> message = MessageBuilder.withPayload(getOrderList(firstIndex, lastIndex)).build();
            orderGateway.processOrderRequest(message);
            firstIndex += ORDER_LIST_SIZE;
            lastIndex += ORDER_LIST_SIZE;
            Thread.sleep(SLEEP_DURATION);
          }
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
        }
      }
    });
  }
  
  /**
     * Creates a sample order list and returns.
     *
     * @return order list
     */
    private static List<Order> getOrderList(final int firstIndex, final int lastIndex) {
      List<Order> orderList = new ArrayList<>(lastIndex);
      for(int i = firstIndex; i < lastIndex; i++) {
        orderList.add(new Order(i, "Sample_Order_" + i));
      }
      
        return orderList;
    }
  
}
```

Second Messaging Node’ s Application Class :

```java
import org.springframework.context.support.ClassPathXmlApplicationContext;

public class Application {
  
  /**
     * Starts the application
     *
     * @param  String[] args
     *
     */
  public static void main(String[] args) {
    new ClassPathXmlApplicationContext("applicationContext.xml");
  }
  
}
```

**STEP 11 : RabbitMQ Cluster Bash Scripts**

First RabbitMQ Server’ s sample bash script is as follows. Also please have a look RabbitMQ Cluster documentation for other configuration steps.

```yaml
#!/bin/bash
echo "*** First RabbitMQ Server is setting up ***"

export RABBITMQ_HOSTNAME=rabbit@master
export RABBITMQ_NODE_PORT=5672
export RABBITMQ_NODENAME=rabbit@master
export RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15672}]"

/DEV_TOOLS/rabbitmq_server-3.4.2/sbin/rabbitmq-server &

echo "*** Second RabbitMQ Server is set up succesfully. ***"

sleep 5

echo "*** First RabbitMQ Server' s status : ***"

/DEV_TOOLS/rabbitmq_server-3.4.2/sbin/rabbitmqctl status
```

Second RabbitMQ Server’ s sample bash script is as follows :

```yaml
#!/bin/bash
echo "*** Second RabbitMQ Server is setting up ***"

export RABBITMQ_HOSTNAME=rabbit2@master
export RABBITMQ_NODE_PORT=5673
export RABBITMQ_NODENAME=rabbit2@master
export RABBITMQ_SERVER_START_ARGS="-rabbitmq_management listener [{port,15673}]"

/DEV_TOOLS/rabbitmq_server-3.4.2_2/sbin/rabbitmq-server &

echo "*** Second RabbitMQ Server is set up succesfully. ***"

sleep 5

echo "*** Second RabbitMQ Server' s status : ***"

/DEV_TOOLS/rabbitmq_server-3.4.2_2/sbin/rabbitmqctl status

sleep 5

echo "*** Second RabbitMQ Server is being added to cluster... ***"

/DEV_TOOLS/rabbitmq_server-3.4.2_2/sbin/rabbitmqctl -n rabbit2@master stop_app
/DEV_TOOLS/rabbitmq_server-3.4.2_2/sbin/rabbitmqctl -n rabbit2@master join_cluster rabbit@master
/DEV_TOOLS/rabbitmq_server-3.4.2_2/sbin/rabbitmqctl -n rabbit2@master start_app
/DEV_TOOLS/rabbitmq_server-3.4.2/sbin/rabbitmqctl -n rabbit@master set_policy ha-all "^ha\." '{"ha-mode":"all"}'

echo "*** Second RabbitMQ Server is added to cluster successfully... ***"

sleep 5

echo "*** Second RabbitMQ Server' s cluster status : ***"

/DEV_TOOLS/rabbitmq_server-3.4.2_2/sbin/rabbitmqctl cluster_status
```

**STEP 12 : Build & Run Project**

Order Messages’ operational results are as follows :

1- First RabbitMQ Server is started.
2- Second RabbitMQ Server is started and added to cluster.

RabbitMQ Cluster Overview is as follows :

![_config.yml]({{ site.baseurl }}/images/rabbitmq_cluster_overview1.png)

3- First RabbitMQ Server‘ s high available(HA) policy is set.
4- First messaging Node is started. It creates order messages and processes.

When First messaging Node is started, A ha.rabbit.channel RabbitMQ queue is created automatically by Spring Integration context as follows :

![_config.yml]({{ site.baseurl }}/images/ha_rabbit_channel.png)

5- Second messaging Node is started. It does not create Order messages so just processes.
6- Order Lists start to be processed.

After First and Second Messaging Nodes connect to RabbitMQ Cluster, ha.rabbit.channel Queue details are as follows :

![_config.yml]({{ site.baseurl }}/images/ha_rabbit_channel_details1.png)

ha.rabbit.channel Queue on Second RabbitMQ Server :

![_config.yml]({{ site.baseurl }}/images/ha_rabbit_channel_details_21.png)

7- First Messaging Node shuts down.
8- First RabbitMQ Server shuts down and is left from cluster.
9- Second Messaging Node and Second RabbitMQ Server process incoming Order messages for high availability so there is no service interruption. Second RabbitMQ Node’ s Screenshot is as follows :

![_config.yml]({{ site.baseurl }}/images/ha_rabbit_channel_details_2_after_first_node_is_down1.png)

The following console output logs will be seen as well :

First Messaging Node Console :

```yaml
...

22:32:51.838 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 1 - Received Message : Order [id=260, name=Sample_Order_260]
22:32:52.842 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 1 - Received Message : Order [id=261, name=Sample_Order_261]
22:32:53.847 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 1 - Received Message : Order [id=263, name=Sample_Order_263]
22:32:54.852 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 1 - Received Message : Order [id=264, name=Sample_Order_264]
```

After Message id : 264 is delivered to First Messaging Node, it and First RabbitMQ Node are shut down and Second Messaging Node and Second RabbitMQ Node process the remaining order messages as follows :

Second Messaging Node Console :

```yaml
...

22:32:54.211 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=262, name=Sample_Order_262]
22:32:56.214 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=265, name=Sample_Order_265]
22:32:58.219 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=266, name=Sample_Order_266]
22:33:00.223 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=267, name=Sample_Order_267]
22:33:02.229 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=268, name=Sample_Order_268]
22:33:04.234 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=269, name=Sample_Order_269]
22:33:06.239 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=270, name=Sample_Order_270]
22:33:08.241 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=271, name=Sample_Order_271]
22:33:10.247 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=272, name=Sample_Order_272]
22:33:12.252 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=273, name=Sample_Order_273]
22:33:14.255 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=274, name=Sample_Order_274]
22:33:16.258 [SimpleAsyncTaskExecutor-1] DEBUG c.o.p.s.OrderProcessService - Node 2 - Received Message : Order [id=275, name=Sample_Order_275]

...
```

**Source Code**
Source Code is available on Github

**References**
Enterprise Integration Patterns
Spring Integration Reference Manual
Spring Integration 4.1.2.RELEASE API
Pro Spring Integration
RabbitMQ Server Documentation