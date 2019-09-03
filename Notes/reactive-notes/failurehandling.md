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
        
    }
}
```