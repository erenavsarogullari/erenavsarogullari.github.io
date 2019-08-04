Most of the prod application can have multiple internal and external dependencies and their readiness & liveness statuses are important for service quality and SLA. 

When Akka Actor pattern is used, ActorSystem will also be one of these dependencies and its lifecycle will also need to be checked as the other dependencies.

This article aims to show how to define an Actor System Health Check to track its health status at runtime. Let' s have a look for sample Actor System Health Check implementation as follows:

Setup:
JDK v1.8
Scala v2.13
Akka v2.5.23

1- We can define 2 Actor Systems. One of them(`app-actor-system`) is used for service' s computation layer and other one(`app-health-actor-system`) is used for service' s dependency health checks(e.g: service itself or its dependencies(Database, Zookeeper etc...) are Up/Down):
```scala
package io.github.erenavsarogullari.actorsystem.health.check.utils

import akka.actor.ActorSystem
import io.github.erenavsarogullari.actorsystem.health.check.actor.{HeartbeatActor, StackOverflowErrorActor}

object AkkaUtils {

  private def createActorSystem(name: String) = ActorSystem(name)

  val appActorSystem = createActorSystem("app-actor-system")
  val appHealthActorSystem = createActorSystem("app-health-actor-system")

  implicit val appHealthEC = appHealthActorSystem.dispatcher
  implicit val appEC = appActorSystem.dispatcher

  val heartbeatActorRef = appActorSystem.actorOf(HeartbeatActor.props(), "heartbeat-actor")
  val stackOverflowErrorActorRef = appActorSystem.actorOf(StackOverflowErrorActor.props(), "stackoverflow-error-actor")

}
```

2- `HeartbeatActor` running on `app-actor-system` can be defined. This actor will be used to decide if `app-actor-system` is alive or not:
```scala
package io.github.erenavsarogullari.actorsystem.health.check.actor

import akka.actor.{Actor, Props}

case object HeartbeatRequest
case object HeartbeatResponse

object HeartbeatActor {
  def props() = Props(new HeartbeatActor())
}

class HeartbeatActor extends Actor {

  override def receive: Receive = {
    case HeartbeatRequest => sender ! HeartbeatResponse
  }

}
```

3- `StackOverflowErrorActor` running on `app-actor-system` can be defined. This actor will be used to simulate fatal-error case on `app-actor-system` by waiting `TriggerStackOverflowError` message:
```scala
package io.github.erenavsarogullari.actorsystem.health.check.actor

import akka.actor.{Actor, Props}

case object TriggerStackOverflowError

object StackOverflowErrorActor {
  def props() = Props(new StackOverflowErrorActor())
}

class StackOverflowErrorActor extends Actor {
  override def receive: Receive = {
    case TriggerStackOverflowError => throw new StackOverflowError("StackOverflow Error occurred!!!")
  }
}
```

4- `HealthStatusType` defines `app-actor-system`' s potential statuses:
```scala
package io.github.erenavsarogullari.actorsystem.health.check.status

object HealthStatusType extends Enumeration {

  type HealthStatusType = Value
  val UP, UNKNOWN, DOWN = Value

}
```

5- `ActorSystemHealthChecker` checks `app-actor-system` status by sending `HeartbeatRequest` to `HeartbeatActor` and expects `HeartbeatResponse` in timeout (5 secs). If it gets consecutive `akka.pattern.AskTimeoutException` in `FailureThreshold` (3 times), it will set `app-actor-system` status as DOWN, otherwise UP or UNKNOWN.
```scala
package io.github.erenavsarogullari.actorsystem.health.check

import java.util.concurrent.TimeUnit
import java.util.concurrent.atomic.LongAdder

import akka.actor.ActorRef
import akka.pattern.ask
import akka.util.Timeout
import io.github.erenavsarogullari.actorsystem.health.check.actor.{HeartbeatRequest, HeartbeatResponse}
import io.github.erenavsarogullari.actorsystem.health.check.status.HealthStatusType
import io.github.erenavsarogullari.actorsystem.health.check.utils.AkkaUtils
import org.slf4j.LoggerFactory

import scala.concurrent.ExecutionContext
import scala.util.control.NonFatal
import scala.util.{Failure, Success}

object ActorSystemHealthChecker {

  private val logger = LoggerFactory.getLogger(classOf[ActorSystemHealthChecker])

  private implicit val timeout = Timeout(5, TimeUnit.SECONDS)

  private val counter = new LongAdder()
  private val FailureThreshold = 3

  def apply(heartbeatActorRef: ActorRef)(implicit ec: ExecutionContext) = new ActorSystemHealthChecker(heartbeatActorRef)

}

class ActorSystemHealthChecker(heartbeatActorRef: ActorRef)(implicit ec: ExecutionContext) {

  import ActorSystemHealthChecker._
  import HealthStatusType._

  def checkActorSystem(): HealthStatusType = {
    try {
      checkHeartbeat()
      counter.intValue() match {
        case v if(v == 0) => UP
        case v if(v > 0 && v <= FailureThreshold) => UNKNOWN
        case v if(v > FailureThreshold) => DOWN
      }
    } catch {
      case NonFatal(t) => DOWN
    }
  }

  private def checkHeartbeat(): Unit = {
    val futureHeartbeatResponse = (heartbeatActorRef ? HeartbeatRequest).mapTo[HeartbeatResponse.type]
    futureHeartbeatResponse.onComplete {
      case Success(value) => {
        counter.reset()
        logger.info(s"${AkkaUtils.appActorSystem.name} Heartbeat Call is successful")
      }
      case Failure(t) => {
        counter.increment()
        logger.warn(s"${AkkaUtils.appActorSystem.name} Heartbeat Call is failed. ", t.getMessage)
      }
    }
  }

}

```

6- `ActorSystemHealthApp` is defined to run the sample application. It initializes `ActorSystemHealthChecker` and schedules `ActorSystemCheckRunnable` and `ErrorOnActorSystemRunnable`.
```scala
package io.github.erenavsarogullari.actorsystem.health.check

import akka.actor.ActorRef
import io.github.erenavsarogullari.actorsystem.health.check.actor.TriggerStackOverflowError

import scala.concurrent.duration._
import scala.language.postfixOps
import io.github.erenavsarogullari.actorsystem.health.check.utils.AkkaUtils
import org.slf4j.LoggerFactory

object ActorSystemHealthApp {

  private val logger = LoggerFactory.getLogger(ActorSystemHealthApp.getClass)

  def main(args: Array[String]): Unit = {
    val actorSystemHealthChecker = ActorSystemHealthChecker(AkkaUtils.heartbeatActorRef)(AkkaUtils.appHealthEC)
    val actorSystemCheckRunnable = new ActorSystemCheckRunnable(actorSystemHealthChecker)
    AkkaUtils.appHealthActorSystem.scheduler.schedule(0 seconds, 10 seconds, actorSystemCheckRunnable)(AkkaUtils.appHealthEC)

    val errorOnActorSystemRunnable = new ErrorOnActorSystemRunnable(AkkaUtils.stackOverflowErrorActorRef)
    AkkaUtils.appActorSystem.scheduler.scheduleOnce(25 seconds, errorOnActorSystemRunnable)(AkkaUtils.appEC)
  }

  private class ActorSystemCheckRunnable(actorSystemHealthChecker: ActorSystemHealthChecker) extends Runnable {

    override def run(): Unit = {
      actorSystemHealthChecker.checkActorSystem() match {
        case t @ _ => logger.info(s"${AkkaUtils.appActorSystem.name} is ${t.toString}")
      }
    }

  }

  private class ErrorOnActorSystemRunnable(stackOverflowErrorActorRef: ActorRef) extends Runnable {

    override def run(): Unit = stackOverflowErrorActorRef ! TriggerStackOverflowError

  }

}
```

For example: we can wonder what happens when fatal-error(e.g: `StackOverflowError`) occurs on an actor and jvm should not be exited?

If a fatal-error occurs on an actor, its ActorSystem will be shut down as default, and then all actors managed by this ActorSystem will be stopped as well. In this case:
1- As the first-step, Root-Cause-Analysis (RCA) of fatal-error will need to be done and proper fix should be applied asap. \
2- Akka supports `akka.jvm-exit-on-fatal-error` property. This can be set on (default)/off (depends on the case). However, when fatal-error occurs, jvm will exit. This behaviour can prefer most of the use-case. However, there may still be some use case to disable this such as if same operation is applied to next service instance, 
it may also get down and this can also cause for all other nodes. To avoid for these kind of cases and keep problem scope limited (specially, multi-tenanted environment), `jvm-exit-on-fatal-error` property can be set as `off`. 
In this case, ActorSystem will be shut down but jvm will not be exited.

7- `akka.jvm-exit-on-fatal-error` property can be set as `off` because preventing to spread the problem(e.g: Fatal Errors => Stackoverflow, OOM Errors) to the other service instances.
```yaml
akka {

  jvm-exit-on-fatal-error=off

}
```
or this can set programmatically as follows:
```scala
private def setConfig(): Unit = {
    import com.typesafe.config.Config
    import com.typesafe.config.ConfigFactory

    System.setProperty("akka.jvm-exit-on-fatal-error", "off")
    val conf: Config = ConfigFactory.load()
    require(conf.getString("akka.jvm-exit-on-fatal-error") == "off", "akka.jvm-exit-on-fatal-error should be off")
}
```

8- Please find the application trace as follows:
```yaml
INFO ActorSystemHealthChecker - app-actor-system Heartbeat Call is successful
INFO ActorSystemHealthApp$ - app-actor-system is UP
INFO ActorSystemHealthChecker - app-actor-system Heartbeat Call is successful
INFO ActorSystemHealthApp$ - app-actor-system is UP
INFO ActorSystemHealthChecker - app-actor-system Heartbeat Call is successful
INFO ActorSystemHealthApp$ - app-actor-system is UP

Uncaught error from thread [app-actor-system-akka.actor.default-dispatcher-4]: StackOverflow Error occurred!!!, shutting down ActorSystem[app-actor-system]
java.lang.StackOverflowError: StackOverflow Error occurred!!!
	at io.github.erenavsarogullari.actorsystem.health.check.actor.StackOverflowErrorActor$$anonfun$receive$1.applyOrElse(StackOverflowErrorActor.scala:13)
	at akka.actor.Actor.aroundReceive(Actor.scala:539)
	at akka.actor.Actor.aroundReceive$(Actor.scala:537)
	at io.github.erenavsarogullari.actorsystem.health.check.actor.StackOverflowErrorActor.aroundReceive(StackOverflowErrorActor.scala:11)
	at akka.actor.ActorCell.receiveMessage(ActorCell.scala:612)
	at akka.actor.ActorCell.invoke(ActorCell.scala:581)
	at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:268)
	at akka.dispatch.Mailbox.run(Mailbox.scala:229)
	at akka.dispatch.Mailbox.exec(Mailbox.scala:241)
	at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
	at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
	at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
	at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)

WARN ActorSystemHealthChecker - app-actor-system Heartbeat Call is failed. 
INFO ActorSystemHealthApp$ - app-actor-system is UNKNOWN
WARN ActorSystemHealthChecker - app-actor-system Heartbeat Call is failed. 
INFO ActorSystemHealthApp$ - app-actor-system is UNKNOWN
WARN ActorSystemHealthChecker - app-actor-system Heartbeat Call is failed.
INFO ActorSystemHealthApp$ - app-actor-system is UNKNOWN 

WARN ActorSystemHealthChecker - app-actor-system Heartbeat Call is failed. 
INFO ActorSystemHealthApp$ - app-actor-system is DOWN
WARN ActorSystemHealthChecker - app-actor-system Heartbeat Call is failed. 
INFO ActorSystemHealthApp$ - app-actor-system is DOWN
```

### References
[Akka](https://doc.akka.io/docs/akka/current/index.html) \