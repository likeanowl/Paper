### Handling failure with actors

When the synchronous method fails, usually an exception is thrown, caller gets this exception and handles it. When you send a message to actor, this actor processes the message long after the call in context is gone. So, where should failure go? 

Failure also should be send as a message, but to whom? One possibility is to send it back to that actor which had sent the message during whose processing we just failed. However this is not always a good idea, bc for example when you want to buy smth from vending machine and it fails, in a such model you must fix this machine by yourself.

Actors work in system, ActorSystem and individual failure is handled by the team leader, like in a real life. 

#### Supervision

Resilience means recovering from a deformation. It demands containmernt and delegation of failure. 
- failed Actor is terminated or restarted
- decision must be taken by one other Actor
- supervised Actors form a tree structure
- the supervisor need to create its subordinate

<b>Containment</b> means that failure is isolated and cannot spread to other components. Actor model naturally takes care of this because we have seen that actors are fully encapsulated objects.
<b>Delegation</b> means that failure could not be handled by the failed component, bc it is presumably compromised so the handling of failure must be delegated to another. For actor this means that one other actor needs to decide what to do with the failed actor.
Supervising actor should be able to terminate or restart failed actors. Therefore, the supervision and the creation hierarchies are same. In Akka it called mandatory parental supervision.

#### Supervision startegy
In Akka the parent declares how its child Actors are supervised:
```scala
class Manager extends Actor {
    override val supervisorStrategy = OneForOneStrategy() {
        case _: DBException => Restart
        case _: ActorKiledException => Stop
        case _: ServiceDownException => Escalate
    }
    ...
    context.actorOf(Props[DbActor], "db")
    context.actorOf(Props[ImportantServiceActor], "service")
}
```
This Manager supervises two actors: `DBActor` and `ImportantServiceActor`. `supervisorStrategy` is a partial function which allows to handle some exceptions cases.
Because failure is sent like a message it is also processed like a message. This means that supervisor strategy in actor has full access to inner state of actor. It is also important to point out why `val` used for strategy instead of `def`. If we use `def` here, then the strategy will be re-instantiated during the handling of each failure and that is usually not what you want and it is not particularly efficient.
```scala
calss Manager extends Actor {
    var restarts = Map.Empty[ActorRef, Int].withDefaultValue(0)
    override val supervisorStrategy = OneForOneStrategy() {
        case _: DBException =>
            restarts(sender) match {
                case toomany if toomany > 10 =>
                    restarts -= sender; Stop
                case n =>
                    restarts = restarts.updated(sender, n + 1); Restart
            }
    }
}
```
There are variations on what you can apply. `OneForOneStrategy` always deals with each child in isolation.If a desicion shall apply to all children then there is also the `AllForOneStrategy`. This means the supervisor is managing a group of actors which need to live and die together.
Simple rate trigger included:
- allow a finite number of restarts
- allow a finite number of restarts in a time window
- if restriction violated then Stop instead of Restart

```scala
OneForOneStrategy9maxNrOfRestarts = 10, withinTimeRange = 1.minute) {
    case _: DBException => Restart // will turn into Stop
}
```

#### Actor Identity
Actor is restarted in order to recover from failure. The benefit of this is that other actors can continue communicating with the service provided by this actor before and after the restart without having to fix the problem. This requires that the id by which other actors refer to this actor must be stable across a restart. Therefore, in Akka the `ActorRef` stays valid after a restart. 

What does restart really means? 
- expected error conditions are handled explicitly
- unexpected error indicate invalidated actor state
- restart will install initial behavior/state

#### Actor lifecycle
- start
- (restart)*
- stop



At first it will be started. This happens when the parent calls `context.actorOf[...](...)`. When the call return this, start action will be scheduled to run async. The first thing which happens is that actorcontext cretates a new actor instance. This will run the constructor of the actor class. The second is to run a method, a callback called `preStart`. This is executed before the first msg processed. Until this point messsages may have already been buffered in the actor's mailbox. 

After it the actor begins processing messages and maybe fails. The failure  reaches the actorcontext, which will contact the supervisor to decide what to do. If the verdict was to restart actor, then `preRestart` method executed. Then the old Actor object is terminated and will be collected by GC. Then the new instance of the Actor is created. Again, a hook is run, called `postRestart`. No messages will be processed between the failure and this place in actor's timeline and the message which caused the failure will not be processed again. Then after `postRestarts` actor resumes its actions. 

After a long time, let us say, the actor wants to stop. It signals this to the actor context which will initiate the stop procedure. At this point, last hook will be run, called `postStop`, and after it the actor instance will be terminated. 


Actor local state cannot be kept across restarts, only external state can be managed like this. Child actors not stopped during restart will be restarted recursively.

### Lifecycle monitoring and Error Kernel.
#### Lifecycle monitoring
When we look at an actor from the outside, the only thing we can see about the life cycle is the transition it goes through when it stops. The only actor which can observe that another actor starts is the one which creates it.
And as soon as it has it has the actor ref in hand, the child actor has at least been alive for some period of time. Other actors which learn of the existence of this actor by getting the actor ref, know also that at some point in time, the actor was alive. But in order to find out more, they need to exchange messages with it.

Restarts are not externally visible, the actor ref stays valid. And if the restart is successfull, the actor will handle messages afterwards. That means that an external observer cannot really tell if an actor was restarted or not. 
The only thing which can be seen is if the actor stops responding. 

DeathWatch:
- an Actor registers its interest using context.watch(target)
- it will receive a Terminated(target) message when target stops
- it will not receive any direct messages from target thereafter

To remove the ambiguity between an actor which has terminated and one which is just not replying anymore, there exists the feature called DeathWatch, which is a means to monitor the life cycle of an actor. Any actor can register its interest in Monitoring the life cycle of an Actor for which it has the ActorRef by callin contextx.watch and giving it target's ActorRef. This is required in order to disambiguate between the multiple actors DeathWatch may be monitoring. 

The meaning of this terminated message is that this actor will not receive any further direct communication from the target. 

#### Deathwatch API
```scala
trait ActorContext {
    def watch(target: ActorRef): ActorRef
    def unwatch(target: ActorRef): ActorRef
    ...
}

case class Terminated private[akka] (actor: ActorRef)
    (val existenceConfirmed: Boolean, val AddressTerminated: Boolean)
    extends AutoReceiveMEssage with PossiblyHarmful
```
The Deathwatch API consistes of 2 methods which are offered on the `ActorContext`. Watch given a target when registered that we want to watch that target, and unwatch on the otehr hand will remove this registration. 

Akka takes care that after you called unwatch, the terminated message for that target will no longer be delivered. This holds true even in the case when unwatch is called at the time during which the actor is currently terminating. There is no race condition here.

The third piece of the API is the `Terminated` message itself. 
- a special message, you could not create it yourself
- contains the `ActorRef` of the `Actor` which has terminated
- whether the existence of this Actor could be confirmed
- `addressTermintaed` gives information whether this message has been sythesized by the system.
- it extends `AutoReceiveMessage` which is an akka marker trait signaling that terminated messages are handled by the ActorContext especially. This is done because `Terminated` message should not be delivered after `unwatch` has been called
- `PossiblyHarmful` signals something (@finish this section)

#### The Children list
Each actor maintains a list of the actors it created:
- the child has been entered when `context.actorOf` returns
- the child has been removed when `Terminated` is received
- an actor name is available IFF there is no such child
- actors are identified by names, therefore if you try to create actor with the same name as existing actor, you will get exception bc of `children` list.

#### Example with Deathwatch API
```scala
class Manager exteds Actor {
    def prime(): Receive = {
        val db = context.actorOf(Props[DBActor], "db)
        context.watch(db)

        {
            case Terminated('db') => context.become(backup())
        }
    }

    def backup(): Receive = {...}
    def receive = prime()
}
```

#### Error Kernel
What follow from actor's ability to restart? First of all, you need to keep important data near root node of your Actors hierarchy tree. This is useful because restarts are recursive (supervised actors are part of the state) and when Actor is restarted, it loses all it's local data. Let's summarize:
- restarts are recursive
- restarts are more frequent near the leaves
- avoid restarting Actors with important state

Such approach is called <b>The Error Kernel Pattern</b>. It is forced to use by Akka, bc the supervision is mandatory, not optional, and therefore you must create hierarchies. 

#### Interjection: the EventStream
Actors can send direct messages only at known addresses. The `EventStream` allows publication of messages, it is like to sending it to an unknown audience. Every actor can optionally subscribe to (parts of) the EventStream.
```scala
trait EventStream {
    def subscribe(subscriber: ActorRef, topic: Class[_]): Boolean
    def unsubscribe(subscriber: ActorRef, topic: Class[_]): Boolean
    def unsubscribe(subscriber: ActorRef): Unit
    def publish(event: AnyRef): Unit
}
```

Let's see small example:
```scala
class Listener extends Actor {
    context.system.eventStream.subscribe(self, classOf[LogEvent])
    def receive = {
        case e: LogEvent => ...
    }
    override def postStop(): Unit = {
        context.system.eventStream.unsubscribe(self)
    }
}
```

#### Where do unhandled messages go?
`Actor.Receive` is a partial function, the behavior may not apply. Those messages which do not match are called unhandled and they are passed into the methods with the same name. It accepts `Any` and you can override it, but the default behavior is matching on type and throwing `DeathPactException` on `Terminated`. The default supervisor strategy will treat this `DeathPactException` in a way that it responds with a stop command. All other unhandled messages are published to the eventsStream and you could possibly register a listener to log them.
```scala
trait Actor {
    ...
    def unhandled(message: Any): Unit = message match {
        case Terminated(target) => throw new DeathPactException(target)
        case msg =>
            context.system.eventStream.publish(UnhandledMessage(msg, sender, self))
    }
}
```
### Persistent actor state
Actors representing a stateful resource. In order to recover from, for example, the power outage, we need to somehow save it on hard drive. If we do so, we can then read the last persissted state and start from there.
Same applies for restart.
Two possibilities for persisting state:
- in-place updates, so when the actor's state changes, the persistent location is also updated. Persistent location could be files, in the file system, or e.g. records in db
- persist changes in append-only fashion, which means persisting not the state itself, but it's changes. it's called `append-only`, bc records never will be deleted, only added. Then, if state is requested, the changes could be applied sequentially. 

#### Changes vs State persistence 
Benefits of persisting current state:
- Recovery of latest state in constant time
- Data volume depends on number of records, not their change rate

Benefits of persisting changes:
- History can be replayed, audited, scored, etc...
- Some processing errors can be corrected retroactively
- Additional insight can be gained on business processes
- Writing an append-only stream optimizes IO bandwidth
- Changes are immutable and can be freely replicated

#### Snapshots
Immutable snapshots that can be used to bound recovery time. 