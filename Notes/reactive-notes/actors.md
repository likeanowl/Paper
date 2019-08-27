### Actors
Actors represents objects and the Actor model represents how these objects interact. This is all inspired by human communications. It is fully based on messages, and message transponting is takes time. 
An Actor:
 - is an object with id
 - has a behaviour
 - interacts with other actors only using async message passing

 Actor send a message and then continues doing whatever it wants to do after it, without having to wait for the message to travel to B. This is the most important property of actors.

#### Actor trait
```scala
type Receive = PartialFunction[Any, Unit] //describes the response of actor to a message

trait Actor {
    def receive: Receive
    ...
}
```
Actor returns nothing to its callse, because there is no direct reference to caller due to async message passing.
`Actor` type describes the behavior of an Actor, its response to messages. 

#### Counter example
```scala
class Counter extends Actor {
    var count = 0
    def receive = {
        case "incr" => count += 1
    }
}
```

This object does not exhibit stateful behavior because we can only send it the string and never get any answer back. To make it stateful we must  enable other actors to find out what the value of the counter is.

```scala
class Counter extends Actor {
    var count = 0
    def receive = {
        case "incr" => count += 1
        case ("get", customer: ActorRef) => customer ! count
    }
}
```

Actors can send messages to addresses they know. Addresses are modeled by the ActorRef type. So, in this example if we get tuple with the message "get" and ActorRef, then we can send to a customer the count as a message. The exclamation mark operator is used to send messages and it is pronounced `tell` in Akka. 
Lets see whats else in the `Actor` trait to understand whats happening behind.

```scala
trait Actor {
    implicit val self: ActorRef
    def sender: ActorRef
}

abstract class ActorRef {
    def !(msg: Any)(implicit sender: ActorRef = Acotr.noSender): Unit
    def tell(msg: Any, sender ActorRef) = this.!(msg)(sender)
}
```
Each actor knows its own address, it's an `ActorRef` called `self` and it is implicitly available. Actor ref is an abstract clause which has a method called `tell` and it has implicit parameter of `ActorRef` type. Therefore is you use `tell` within an Actor it will implicitly pick up the sender as being the self reference of that actor. Withing the receiving Actor this value is available as sender which gives back the `ActorRef` which has sent the message which is currently being processed. 
Sending a message from one  actor to the other picks up the sender's address implicitly.
Using `tell` we can rewrite our `Counter` Actor more nicely:
```scala
class Counter extends Actor {
    var count = 0
    def receive = {
        case "incr" => count += 1
        case "get" => customer ! count
    }
}
```

#### The Actor's Context
```scala
trait ActorContext {
    def become(behavior: Receive, descardOld: Boolean = true): Unit
    def unbecome(): Unit
    ...
}

trait Actor {
    implicit val context: ActorContext
}
```
An Actor can do more things than just send messages. It can create other actors and it can change its behviour. To access such functions we need to know about the actor's context. The actor type itself only has the receive method, so it only describes the behavior of the actor. The execution is done by its `ActorContext`. Each actor has a stack of behaviors and the top one is alwasy theacted one. The default mode of `become` is to replace the top of the stack with a new behavior. You can also use it to push an `unbecome` to pop the top behavior. The `ActorContext` within the `Actor` could be accessed just by saying `context`.


#### Behaviors in action
```scala
class Counter extends Actor {
    def counter(n: Int): Receive = {
        def counter(n: Int): Receive = {
            case "incr" => context.become(counter(n + 1))
            case "get" => sender ! n
        }
        def receive = counter(0)
    }
}
```
First we define a method which gives us a behavior. This method takes an argument what the state of the counter currently is. We start out with zero counter, the behavior counter of zero. If we get an `incr` message we change our behavior to the `counter(n + 1)`. It is asynchronous because context.become evaluates what is given here only whe the next message is processed. This is functionally equivalent to the previous version of the counter, but has some advantages:
- state change is explicit
- state is scoped to current behavior


#### Creating and stopping actors
```scala
trait ActorContext {
    def actorOf(p: Props, name: String): ActorRef
    def stop(a: ActorRef): Unit
    ...
}
```
Actors are created by actors. `stop` is often applied to `self`. 

### Messages processing semantics
#### ActorRef
The most important property of actors is that access to their state is only possible by excanging messages. There's no way to directly access current behavior of actor. Only messages can be sent to known addresses (`ActorRef`). Every actor knows its own address and its useful when sending messages to other actors and telling them where to reply.
`ActorRef` could be obtained by following ways:
 - `self`
 - creating an actor returns its address
 - addresses can be sent within messages (e.g. sender)

Actors are completely independent agents of computation:
 - local execution, no global synchronization
 - all actors run fully concurrently
 - message-passing primitive is one-way communication

On the inside, actors are effectively single-threaded:
 - messages are received sequientially
 - behavior change is effective before processing the next message
 - processing on message is the atomic unit of execution

This allows to obtain benefits of synchronized methods, but withoud blocking. Blocking is replaced by enqueueing a message for later execution.

#### Bank account example
It is a good practice to define an actor messages in its companion object
```scala
object BankAccount {
    case class Deposit(amount: BigInt) {
        require(amount > 0)
    }
    case class Withdraw(amount: BigInt) {
        require(amount > 0)
    }
    case object Done
    case object Failed
}

class BankAccount extends Actor {
    import BankAccount._

    var balance = BigInt(0)

    def receive = {
        case Deposit(amount) => balance += amount
            sender ! Done
        case Withdraw(amount) if amount <= balance => balance -= amount
            sender ! Done
        case _ => sender ! Failed
    }
}
```
We would not face any issues like race-condition or deadlock, because all operations inside the actor are synchronized. 
Lets see transferring actors:
```scala
object WireTransfer {
    case class Transfer(from: ActorRef, to: ActorRef, amount: BigInt)
    case object Done
    case object Failed
}

class WireTransfer extends Actor {
    import WireTransfer._

    def receive = {
        case Transfer(from, to, amount) =>
            from ! BankAccount.Withdraw(amount)
            context.become(awaitWithdraws(to, amount, sender))
    }

    def awaitWithdraw(to: ActorRef, amount: BigInt, client: ActorRef): receive = {
        case BankAccount.Done =>
            to ! BankAccount.Deposit(amount)
            context.become(awaitDeposit(client))
        case BankAccount.Failed =>
            client ! Failed
            context.stop(self)
    }

    def awaitDeposit(client: ActorRef): Receive = {
        case BankAccount.Done =>
            client ! Done
            context.stop(self)
    }
}
```

#### Message Delivery Guarantees

Akka provides only `at most one` delivery guarantee, because communication is inherently unreliable.  Delivery of a message requires eventual availability of channel & recipient.
Types of guarantees:
 - `at-most-once`: sending once delivers [0, 1] times
 - `at-least-once`: resending until acknowledged delivers [1, inf) times
 - `exactly-once`: processing only first reception delivers 1 time

 First option can be done without keeping any state in sender or receiver, the second choice requires that the sender need to keep the message to buffer it, in order to be able to resend. And the third choice additionally requires the receiver to keep track of which messages have already been processed. 

 #### Reliable messaging
 Messages support reliability:
 - all messages can be persisted
 - can include unique correlation IDs
 - delivery can be retried until successful

 Making messages explicitly supports reliability quite well. Messages can be persistent, i.e. stored in some persistent storage. Such thing could not be done with local method invocation because context may go away. And on the other hand a message frome one sender to one receiver is something that can be persisted. 

 If a unique relation ID is included in the message, you can enable the exactly once semantics by allowing the recipient to find out whether it has already received this unique message.

 And finally if the messages were persisted then retries can even be restared after a catastrophic failure. 

 An important thing to note is that it is not enough to deliver a message to a recipien bc if you do not get a reply from that recipient you can never be sure whether it has processed it. Therefore, all of these semantics discussed only work if ensured by business-level acknowledgement. 

 Back to our example, making the transfer reliable requires:
 - log activities of WireTransfer to persistent storage
 - each transfer has a unique ID
 - add ID to withdraw and deposit
 - store IDs of completed actions within BankAccount

 #### Message order
 Akka guarantees that if an actor sends multiple messages to the same destination they will not arrive out of order.

 #### Summary
 - Actors are fully encapsulated independent agents of computation
 - Messages are the only way to interact with actors
 - Explicit messageing allows explicit treatment of reliability
 - The order in which messages are processed is mostly undefined

 ### Designing actor systems

 When starting to design an actor system its often useful to imagine giving the task to a group of people, dividing it up. You must consider the group to be of very large size. 
 You should start with designing how people with different tasks will talk with each other. Consider these people to be easily replaceable.
 Drawing a diagram also helpful, draw how the task will split up, including communication lines. 

 #### Example: web crawler
 Write an actor system which given a URL will recursively download the content, extract links and follow them, bounded by a maximum depth; all links encountered shall be returned. 
 There will be a `Receptionist` actor, which will be responsible for accepting some links.

#### @todo finish this lecture


#### Ask pattern

Actor can wait for a message. It is called 'ask' and return `Future` of response. 
#### PipeTo

Do not ever share mutable state of Actor! Imagine we have service actor.
```scala
object Service {
    sealed trait Message
    case object Get extends Message
    case class Result(value: String)
}
import Service._

class Service extends Actor {
    def asyncAction(): Future[Result] = {
        import akka.pattern.after
        after(100.millis, system.scheduler)(Future.successful(Result("res")))
    }

    def receive: Receive = {
        case Get =>
            asyncAction().foreach(res => sender() ! res)
    }
}

val service = system.actorOf(Props(new Service), "service")

val result = (service ? Get).mapTo[Result]

```
It has a `Get` method and should return some result. We don't want to process result as `Any` and therefore using `mapTo` pattern. `mapTo` simply casts value to type and therefore it is not type-safe (you could get classcast exc in Future) but this is usable (if you do not have akka-typed)
In order to obtain result, actor must perform async action. It maps callback on future and when result is ready we send it to sender. We do not guarantee result obtaining time. 
If we run the code above, we will see that our message from service was not delivered and was found in dead letters. This happened because of
```scala
case Get =>
    asyncAction().foreach(res => sender ! res)
```
This lamda expression performs closure. And we can enclosure only local variables (method params are local vars too), but here we enclosuring `this`, a first implicit method argument. Therefore, we invoking a `this.sender` when we already left `receive` and `sender` is not valid. 
This could be fixed easily:
```scala
case Get =>
    val replyTo = sender()
    asyncAction().foreach(res => replyTo ! res)
```
We call sender explicitly here, while it still valid. `sender` is a mutable state of Actor and we shared a mutable state of actor to other thread (`foreach` in this case). Any closure could possibly share actor's local state.
This approach is cool, but we don't want to always write as this. We have a pattern `pipeTo` in order to deal with this. 
```scala
case Get => 
    import akka.pattern.pipe
    asyncAction().pipeTo(sender())
```

Why this works? Because `pipeTo` obtains value not `by name` and `sender` is evaluated right in place.
So, if we want to pass `Future` to some actor, use `pipeTo` for this. 


#### Actor hierarchy, supervision and monitoring
Each actor has a parent. `root` actor is a parent of all actors.
`user` actor is a parent for all actors created by users.
`system` is a parent actor for system actors.
Why separation for `user` and `system` is needed? Because of logging. When all user defined actors have finished their jobs, logging of system must continue. 

Actor path:
1. Protocol
2. System name
3. Address
e.g. `akka.tcp://sys@host:2552/user/parent/child`

