# Typed actors

## Introduction to protocols

As a `protocol` we do not only mean technical protocols like TCP or HTTP, but just everyday protocols when two parties communicating with one another through the exchange of language of messages.

![protocol example](./resources/protocol_example.png)

### Formal description with session types

There are several ways to formalize communications, one of which is called session types.

#### Global session types
![session types](./resources/session_types1.png)

Let's map our example with buyer and seller on it.

![session types 2](./resources/session_types2.png)
A session is an execution of a protocol. It is one concrete exchange between a number of parties participating in that exchange. In session types strive to formalize what can happen in this exchange by giving this exchange a type. The first step is to write down the so-called global session type called `G`.

The exchange starts with participant number 1 sending a message to participant number 2 so number 1 here is the buyer number 2 is the seller and the arrow between them means that a message is sent from one to the other. Quite similarly to Scala syntax after the colon we described the type of that message here the first string would be the title of the book. The global session type consists of several steps, these steps are separated by dots at the end of the line. 

In the second step the seller sends a message to the buyer which is of type integer this is the price of the book for example counted in cents. 

Now the third step depends on whether the buyer likes the price or not hence there are 2 choices. These choices are symbolized by the parentheses. If the buyer does not like the price they will just send the `quit` message here to end the protocol. In the other case the buyer continues by sending the shipping address after which the seller responds by confirming the delivery date. 

#### Local session types
The global session type describes in detail all message exchanges that happen within this session. But how can a compiler enforce that an implementation of for example the buyer conforms to this protocol? This is the second step that we need to take when using session types. We need to project the local type for each
of the participants from the global type. The local session type contains all the same message types as the global one but it uses a different set of notation. 

![local session types](./resources/local_session.png)

The exclamation mark means sending a message, the question mark means waiting for the reception of a message, the plus operator represents the internal choice the buyer makes whether to go forward buying the book or to abort the exchange. With this we can see that the buyer first sends the title of the book, then waits for the reception of the price, then makes the decision whether to go ahead or not. If not the buyer sends the quit token, otherwise the buyer sends the delivery address and waits for the confirmation of the delivery date.

Looking at the seller we need to essentially turn this around because we have an exchange between two parties only, so one will do the opposite of the other. The only thing that must be highlighted is that instead of the plus operator for internal choice we use the ampersand to represent external choice. So the seller needs to be aware that there can be either the okay case or the quit case and the choice is done on the other end of the communications channel. 

To dive deep you can look at `A gentle introduction to multiparty asynchronous session types`.

### The type of a channel

Now that we have a description of what protocol participants shall do locally let us explore how that can be modeled. We have seen that the communications protocol governs not only when something is sent but what is sent. It specifies a message type for each step and since all communication happens across communications channels this is the crucial part that we need to now look at. 

We have two options to deal with it:
- each participant has one channel per interluctor. 
    The first option is to represent all exchanges between participants `A` and `B` with one channel that has one end at `A` and one end at `B`. If we do this it is obvious that this channel will need to have a different type. It will need to accept different types of messages at different times at different steps of the protocol.

- one channel for each message type
    The other option is to exchange channels between the participants such that every step or every message type is represented by its own channel. That one channel can have one fixed type that does not need to change. Since Scala and also Java cannot express that the type of an object changes over time akka typed uses this approach.

What we must achieve is that once we get a channel we can send a message with the correct type, but for example sending that same message again if the protocol didn't allow this should not be legal. 

```scala
val channel ...
channel.send(msg) //works
channel.send(msg) //should not work
```

How we can achieve this? What we need here is the introduction of **linear logic**. In the traditional or common sense logic that everyone uses all the time we are used to only adding knowledge to the pool. Once we have for example allowed `channel.send(msg)` to succeed it must succeed again because we have gained the knowledge that it should be legal. Linear logic on the other hand allows the removal of knowledge. It can prevent us from using a certain fact more than once you can think of it as a permission token that sits on this channel here. Once we call send the permission token is used is gone we cannot send it again because there is no such permission anymore. The channel could return a new channel with a new permissions token for a different message type and that way we could implement that a channel changes its type over time.

## Protocols in akka typed

### Typed actors 101
In akka all communication is mediated by `ActorRef`, hence the main goal is to ensure that an ActorRef does not permit sending the wrong message. In other words, the `!` operator needs to be restricted to accept only the correct message type. 

The way we can get to this:

1. The first consequence is to add a type parameter to `ActorRef`, like `ActorRef[T]` and that actor will allow sending messages of type `T`.
2. Then we use this type parameter in a `!` operator to restrict its argument. 
3. The recepient needs to declare which kind of messages it understands so we need to add that type to the `Actor` trait in some way. 
4. Actor's self reference also needs to have the right type this means that the `ActorContext` also needs to refer to the type T. 
5. Remove `context.sender`. The identity of the sender of the current message can no longer be returned by the `ActorContext` because what would its type be? Since any actor in the same actor system could have sent this message we cannot possibly know the self type of that `ActorRef`

All of these changes are obviously breaking existing source code so they cannot be done in a compatible fashion. This opened up the possibility for cleaning up some more things:
1. Turn stateful trait `Actor` into pure `Behavior[T]`. Stateful trait `Actor` and its subclasses invites certain hard to track programming mistakes like *closing over the internal actor state within future callbacks*. Therefore Akka Typed does not have a trait `Actor` anymore. Instead it uses a pure `Behavior[T]` to describe the reaction of the actor to an incoming message.
2. Remove `system.actorOf`, instead require guardian behavior. This method allowed external code to ask the Guardian actor of the system to create new actors. This has always been troublesome because an actor should be responsible for creating its own child actors without being forced from the outside. Therefore in Akka typed you can specify the guardian's behavior so if this Guardian offers the facilities of creating new top level actors that is your choice, otherwise that guardian actor can do whatever the actor system needs to do and you describe it.
3. `ActorSystem[T]` is an `ActorRef[T]` for the guardian. If the Guardian is a real actor that you can talk to, there needs to be an ActorRef for communicating with this Guardian actor. Since this Guardian actor is responsible for the whole actor system function it makes sense that the `ActorSystem` now also has a type parameter T that matches an ActorRef for the guardian of behavior of type T.

### Akka typed hello world example
Minimal protocol: accept one message and then stop. In Akka Typed we do not create an actor, we create its behavior and for creating behaviors there are several possible behavior factories that we can use.
```scala
val greeter: Behavior[String] = 
    Behaviors.receiveMessage[String] { whom => 
        println(s"Hello $whom!")
        Behaviors.stopped
    }
```

One of the behavior factories is called `receiveMessage`. Within the behaviors object receive message takes two parameters: type that designates the message type allowed for this actor and the second is a function that computes the next behavior upon reception of such a message.


In order to execute this behavior we place it inside an actor system.
```scala
object Hello extends App {
    val greeter: Behavior[String] = 
        Behaviors.receiveMessage[String] { whom =>
            println(s"Hello $whom!")
            Behaviors.stopped
        }

    //start a system with this primitive guardian
    val system: ActorSystem[String] = ActorSystem(greeter, "helloworld")

    //send a message to the guardian
    system ! "world"

    //system stops when guardian stops
}
```
The actor system constructor here takes not only the name as previously, but also the Guardian's behavior.


### Proper channels with ADT
Most actors do not respond only to one kind of message, but to multiple. In Scala this is modeled by using algebraic data types meaning we have a sealed trait for the `Greeter` and then we have two different meanings:  the case class `Greet` and the case object `Stop`.
The `Greet` message contains a parameter: whom to greet, while the `Stop` object does not need any parameters.

```scala
sealed trait Greeter
final case class Greet(whom: String) extends Greeter
final case object Stop extends Greeter

val greeter: Behavior[Greeter] = 
    Behaviors.receiveMessage[Greeter] {
        case Greet(whom) => 
            println(s"Hello $whom!")
            Behaviors.same
        case Stop =>
            println("shutting down ...")
            Behaviors.stopped
    }
```

### Runnin actor programs
The best way is to start an `ActorSystem` and place the initialization code in the guardian's behavior:

```scala
ActorSystem[Nothing](Behaviors.setup[Nothing]) { ctx =>
    val greeterRef = ctx.spawn(greeter, "greeter")
    ctx.watch(greeterRef) // sign death pact

    greeterRef ! Greet("world")
    greeterRef ! Stop

    Behaviors.empty
}, "helloworld")
```

In this case the behavior for any message should be empty. This actor system cannot be talked to from the outside hence also the type nothing is used as a type parameter here for the resulting ActorRef.

We want this actor system to shut down as soon as the greeter stops in response to this stop command and this is why we use the well-known **death watch** feature to sign the death pact between the child actor and its parent. So here we watch greeterRef and we simply never handle the resulting terminated message.

### Handling typed responses

A response of type T must be sent via an `ActorRef[T]`:

```scala
sealed trait Guardian
case class NewGreeter(replyTo: ActorRef[ActorRef[Greeter]]) extends Guardian
case object Shutdown extends Guardian

val guardian = Behaviors.receive[Guardian] {
    case (ctx, NewGreeter(replyTo)) =>
        val ref: ActorRef[Greeter] = ctx.spawnAnonymous(greeter)
        replyTo ! ref
        Behavior.same
    case (_, Shutdown) => Behavior.stopped
}
```

Handling actor responses should be different now. Since context at sender has been removed, in order to send back a response of type T we now need to have an appropriately typed ActorRef for messages of type T.

In this example we create a Guardian actor. This Guardian actor has two commands: it can create a greeter (new greeter command) or it can shut down. The new greeter command shall be answered with an ActorRef to this new greeter so that whoever asked for it can now talk to the created child actor. This means that the new greeter message has a `replyTo` field that is an ActorRef where we can send an ActorRef of a greeter hence its ActorRef of ActorRef of Greeter. 

Implementing this guardian shows one new feature namely that we can of course also spawn anonymous child actors. The resulting ActorRef of type greeter is then sent to the reply to address and the guardian stays in the same behavior. This example also shows the receive constructor for behaviors, it receives a pair of the actor context and the message. We need the context to call spawn anonymous actor. 

### Back to the buyer/seller example

Now we know enough to encode the protocol that we initially talked about, the protocol between the buyer and the seller of a book.

![buyer seller](./resources/buyer_seller.png)

First the buyer requests a `Quote`, then the seller sends back a `Quote`. If the buyer is okay with the quoted price they will send a `Buy` message and the seller will respond with a `Shipping` confirmation. 

These message types represent the sequence of events that will happen during this exchange. Let's draw a graph of states:
![states graph](./resources/graph_of_states.png)

The messages implemented with ADT in Scala look like this:
```scala
case class RequestQuote(title: String, buyer: ActorRef[Quote])
case class Quote(price: BigDecimal, seller: ActorRef[BuyOrQuit])

sealed trait BuyOrQuit
case class Buy(address: Address, buyer: ActorRef[Shipping]) extends BuyOrQuit
case object Quit extends BuyOrQuit

case class Shipping(date: Date)
```

## Testing actors

Testing actors is hard:
- all messaging is async, preventing the use of normal sync testing tools
- async expectations introduce the concept of a timeout
- test procedures become non-deterministic

If instead of testing actors we only test the behaviors, these issues go away. A behavior as we have seen is just a function that takes an input message and computes the next behavior from it. All this happens synchronously and can be driven and asserted by a test procedure.

Therefore, testing actor behaviors should be easy:
- functions from input message to next behavior
- any effects are performed during computation of the function
- full arsenal of testing for functions is available

### Akka typed BehaviorTestKit
Considering the behaviors that we have seen so far, we remember that in addition to the incoming message, there is another input to this function. It is the ActorContext that the behavior can use, for example to create child actors. In a test we need an instance of this ActorContext to evaluate messages, and this actor context is provided by the `BehaviorTestKit`. This test kit is constructed from the initial behavior of the actor under test.

The main idea is to place the behavior into a container that emulates an ActorContext:
```scala
val guardianKit = BehaviorTestKit(guardian)
```

Inject a message:
```scala
guardianKit.ref ! <some message>
guardianKit.runOne()
```

TestKit also supplies an Actor Context that contract effects like child actor creation that are performed on it. Injecting a message into the actor is done as usual using an ActorRef. The guardian kit provides the `.ref` member which is an appropriately typed ActorRef on which the tell operator can be used to send a message to this behavior under test.

This message will not immediately be processed though. The behavioral test kit will enqueue it. It differ its processing until `runOne` is called. That will dequeue the first message, feeded into the current behavior, which computes the next behavior and is then installed for the following message.

The actor context supplied by the `BehaviorTestKit` is the special one that contracts all methods that we called on it. In this way the behavior test kit can know which effects were performed during evaluation.

This includes child actor creations and force termination, DeathWatch setting, receive timeout and scheduling a message. Other effects like file IO or sending a message to another actor are not mediated by the actor context and therefore can not be tracked. There are several methods available on the test kit for inspecting the effects.

```scala
guardianKit.isAlive must be(true)
guardianKit.retrieveAllEffects() must be(Seq(Stopped("Bob")))
```

The `retrieveAllEffects` dequeue from the internal effect queue all effects that are performed by the behavior or enqueue as objects within the test kit. `retrieveAllEffects` empties that queue and returns the values in the sequence so that we can assert them. While this can be very handy and some situations,  please keep in mind that how an actor performs its function should be private to that actor. The standart approach is to observe only messages! Inspecting effects should be considered as whitebox testing. 

### Akka Typed TestInbox

As for untyped actors, there is a test inbox for Akka Typed.
```scala
val sessionInbox = TestInbox[ActorRef[Command]]()
```

This test inbox is parametrized with the type of message that it can enqueue and it will provide an ActorRef that accept this message type. Using this facility we can exchange messages between the test procedure and the behavior on the test:

```scala
guardianKit.ref ! NewGreeter(sessionInbox.ref)
guardianKit.runOne()
val greeterRef = sessionInbox.receiveMessage()
sessionInbox.hasMessages must be(false)
```

### Testing child actor behaviors
For the testing of child actor's behavior we need a suitable behavior test kit. Since the greeter actor was created by the guardian, the greeter's behavior test kit knows this child actor and we can ask for the child test kit for this particular child's actor ref. This results in the greeter kit that we can use just like the guardian kit.

```scala
//retrive child actor's behavior testkit
val greeterKit = guardianKit.childTestKit(greeterRef)

// test a greeting
val doneInbox = TestInbox[Done]()
greeterRef ! Greet("World", doneInbox.ref)
greeterKit.runOne()
doneInbox.receiveAll() must be (Seq(Done))

//test shutting down the greeter
greeterRef ! Stop
greeterKit.runOne()
greeterKit.isAlive must be(false)
```
