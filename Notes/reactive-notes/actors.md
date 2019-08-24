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