### Distributed computing
Actors are independent agents of computations and distributed by default. Usually they are run just on different CPUs in the same system, but could easily run on separate machines in same network.

### Akka cluster
Lets look at the impact of network communication on a distributive program compared to communication within the same process by calling methods, for example.
We will encounter some differences when communicating over network:
1. Memory is not shared anymore, data can be only shared by value because a copy will be made, the object will be serialized, sent over the network and de-serialized, tehrefore the object is no longer the same. The stateful object is one whose behavior depends on its history, but the history of the copy on the other side of the network would not be the same. This leads to that only immutable objects can be sensibly shared. 
2. Bandwith a lot lower
3. Latency becomes higher, you can call a method in nano second, but cant transfer a package this fast
4. When you send a message over the network, partial failure could occur
5. Data corruption could happen

Multiple processes on the same machine are quantitatively less impacted, but qualitatively the issues are the same.
Distributed computing breaks assumptions made by the synchronous programming model. 

#### Actors are distributed
**Actor communication is async, one-way and not guaranteed**
Actors are distributed by their nature, they model exactly what the network gives us. Instead of takin the local model and trying to extend it to the network, actors tooke the inverse approach: looking at the network model and using that on the local machine. 

**Actor encapsulation makes them look the same, regardless where they live**

**Actors are "Location transparent", hidden behind `ActorRef`**
Regardless where they live it will be the same sending a message to them, this is what we call "Location transparency". The current semantics in feature set offered by Akka Actors have been reached by rigorously treating everything as if it were remote. And all features  which cannot be modeled in this world, were removed.

**As a result, writing distributed program on Actors is the same way as writing the local program. The code itself will not look much different**

#### Actor Paths
Every actor system has an Address, forming scheme and authority of a hierarchical URI. Therefore their names form a tree, like a filesystem. Actor names form the URI's path elements. Lets look at the example of spawning an actor from Guardian Actor with name "user":
```scala
val system = ActorSystem("HelloWorld")
val ref = system.actorOf(Props[Greeter], "greeter")
println(ref.path) // prints akka://HelloWorld/user/greeter
```
Remote address example with akka system using TCP protocol, actor system is named "HelloWorld", hostname "10.2.4.6" and port "6565": `akka.tcp://HelloWorld@10.2.4.6:6565`. This description is enough for any other actorsystem to send a message to any actor within this one. E.g.: `akka.tcp://HelloWorld@10.2.4.6:6565/user/greeter`

Every actor is identified by at least one URI. Multiple URI could occur if actor system is reachable by multiple protocols or multiple IP addresses.

####Difference between ActorRef and ActorPath

ActorPath has relationship to it ActorRef: actor names are unique within a parent, but once the terminated message has been delivered for a child, the parent knows that the name can be reused. It could create an actor which has exactly the same name but it will not be the same actor, the ActorRef will be a different one. Therefore, you should keep in mind that ActorPath is just the full name of the actor and the ActorPath exists whether the actor exists or not. An ActorRef on the other hand point exactly to one actor, which was stareted at some point, an 'incarnation'.

ActorPath can only optimistically send a message. 

ActorRef can be used to monitor life cycle of an Actor. ActorRef example is: `akka://HelloWorld/user/greeter#43428347`

#### Resolving an ActorPath

When communicating with remote systems, it is necessary to talk to actors which you have not created and for which you have no means to acquire an ActorRef, you just know at which host and prot the Actor lives, what the systems name is and where the Actor is in the hierarchy over there. To obtain ActorRef from this, ActorContext has a method called `actorSelection`. It can accept any actor path and will construct something which you can send to. 

```scala
import akka.actor.{Identify, ActorIdentity}
case class Resolve(path: ActorPath)
case class Resolved(path: ActorPath, ref: ActorRef)
case class NotResolved(path: ActorPath)

class Resolver extends Actor {
    def receive = {
        case Resolve(path) => context.actorSelection(path) ! Identify((path, sender))
        case ActorIdentity((path, client), Some(ref)) => client ! Resolved(path, ref)
        case ActorIdentity((path, client), None) => client ! NotResolved(path)
    }
}
```
And, there is one message that every actor automatically handles. And that is akka.actor.identify, imported here, which takes one piece of data, a correlation identifier. So sending the Identify message with ActorPath will result in us geeting back an ActorIdentity. If the actor is currently alive, we get `Some(ref)`. 

#### Relative Actor Paths
```scala
// Looking up a grand-child:
context.actorSelection("child/grandchild")
//Looking up a sibling:
context.actorSelection("../sibling")
//Looking up from the local root:
context.actorSelection("/user/app")
//Broadcasting using wildcards:
context.actorSelection("/user/controllers/*")
```

The ability to send to a name instead of to an ActorRef is exploited by the Akka cluster module.

#### What is a cluster

A cluster is foremost a set of nodes (of actor systems in our case) and this set is defined such that all member of it (all nodes of the same cluster) have the same idea about who is in the cluster and who is not. These nodes can collaborate on a common task. 

#### Formation of the cluster
Clusters are formed in a form of inductive reasoning. It starts with one node, which basically joins itself, it declares itself to be a cluster of size one, and then any given node can join a given cluster, to enlarge that cluster by 1. This is achieved by sending a request to any node in cluster and once all the current members have learned of this, they agreee to accept a new node in. One importan property of Akka Cluster is that it does not contain any central leader or coordinator, which would be a single point of failure. 

Information is disseminated in an epidemic fashion. THis gossip protocol is resilent to failure because every note will gossip to a few of tis peers, revery second, regardless of whether that was successful or not, so evntually all information will be spread throughout the cluster. 

#### Starting up a cluster
Prerequisites:
- You need an `akka-cluster` dependency
- configuration enabling cluster module:
```scala
akka {
    actor {
        provider = akka.cluster.CLusterActorRefProvider
    }
}
```
in `application.conf` or as `-Dakka.actor.provider=...`

All calls to context actor of are in the end handled by the ActorRefProvider and the cluster one supports a few operations that the local one cannot, as we will see further. 

Next, we need to write a new main program. 
```scala
class ClusterMain extends Actor {
    val cluster = CLuster(context.system)
    cluster.subscribe(self, classOf[ClusterEvent.MemberUp])
    cluster.join(cluster.selfAddress)

    def receive = {
        case ClusterEvent.MemberUp(member) => if (member.address != cluster.selfAddress) {
            // someone joined
        }
    }
}
```
This will start a single-node cluster on port 2552. This acotr when it starts up, obtains the cluster extension of the system, subscribes to some events (This is works the same way as the event stream). And finally it joins its own address. In the behavior we declare logic of proccessing events. 

Lets write a second node: (this needs configureation akka.remote.netty.tcp.port = 0, this means that random port would be used) Since all actor systems will need to listen on a TCP port, but only one of them can have the port 2552 we need to configure a different port for the `ClusterWorker`. 
```scala
class ClusterWorker extends Actor {
    val cluster = Cluster(context.system)
    cluster.subscribe(self. classOf[ClusterEvent.MemberRemoved])
    val main = cluster.selfAddress.copy(port = Some(2552))
    cluster.join(main)
    
    def receive = {
        case ClusterEvent.MemberRemoved(m, _) => 
            if (m.address == main) context.stop(self)
    }
}
```
We don't need to know at which port the cluster worker lives because it will just join the main one. The address of the cluster main can be derived from this worker's self address by replacing the port with the number 2552. And then the worker joins the main. Also, this worker receives not member up, but member remove events, and whenever the address of the removed member is the main one. So when the main program shuts down, this also stops. 

With those 2 main programs we could observe that they will join, but nothing much will happen. We need to define some actor which makes use of the cluster. For this we write Receptionist which would spawn actors on cluster nodes which are not the current node.

```scala
class ClusterReceptionist extends Actor {
    val cluster = Cluster(context.system)
    cluster.subscribe(self, classOf[MemberUp])
    cluster.subscribe(self, classOf[MemberRemoved])

    override def postSTop(): Unit = {
        cluster.unsubscribe(self)
    }

    def receive = awaitingMembers

    val awaitingMembers: Receive = {
        case current: ClusterEvent.CurrentClusterState =>
            val addresses = current.members.toVector map (_.address)
            val notMe = addresses filter (_ != cluster.address)
            if (notMe.nonEmpty) context.become(active(notMe))
        case MemberUp(member) if member.address != cluster.selfAddress => context.become(active(Vector(member.address)))
        case Get(url) => sender ! Failed(url, "no nodes available")
    }

    def active(addresses: Vector [Address]): Receive = {
        case MemberUp(member) if member.address != cluster.selfAddress => context.become(active(addresses :+ member.address))
        case MemberRemoved(member, _) => 
            val next = addresses filterNot (_ == member.address)
            if (next.isEmpty) context.become(awaitingMembers)
            else context.become(active(next))
        case Get(url) if context.children.size < addresses.size => 
            val client = sender
            val address = pick(addresses)
            context.actorOf(Props(new Customer(client, url, address)))
        case Get(url) => sender ! Failed(url, "too many parallel queries")
    }
}
```
How this will work? First of all, the receptionist needs to know who is in the cluster and who is not. Therefore, it subscribes to `MemberUp` and `MemberRemoved` events.  And when it stops, it unsubscribes itself in response to `cluster.subscribe`. The actor will always receive the current cluster state with the members list. We convert this list to addressess of the cluster nodes. Then from these addresses we filter out the self address. So whatever remains is not `Receptionist` node, and if there is anotehr node, then we change behvaior to `active(notMe)`.

The `active` behavior will also have to monitor the cluster because after all, members can be added or removed at any point in time. When more members are added, again which is not the self address, then we just change to the active state with the addition of the newly known address. And when members are removed from this set, we filter it out from the addresses. And if that was the last one, then we go back to the awaiting members state, otherwise we continuer with the reduced list. 

Now we get closer to the interesting part, using the information we just obtained. In the active state, when a get request comes in. We look whether the currently running requests that is `context.children.size` is less than the addresses we know about.

Otherwise, we have one request running per cluster node. Lets say that is the limit and then we reject it. But If it is the first request which comes in, that will always work. So we copy the client, that's the sender of this get request. Then we pick an address randomly from the list and extract it here. And then we create a new actor, a customer, which gets the client, URL, which is supposed to be retrieved, and a cluster address, where the work is supposed to be performed.

We can create actor locally but also we can pass other arguments when creating the actor. For example, we can use `withDeploy` and pass the description of how to deploy actor:

```scala
class Customer(client: ActorRef, url: String, node: Address) extends Actor {
    implicit val s = context.parent

    override val supervisorStrategy = SupervisorStrategy.stoppingStrategy
    val props = Props[Controller].withDeploy(Deploy(scope = RemoteScope(node))
    val controller = context.actorOf(props, "controller)
    context.watch(controller)

    context.setReceiveTimeout(5.seconds)
    controller ! Controller.Check(url, 2)

    def receive = ...
}
```

There are several different things that could be modified, one of them is the scope in which actor shall be deployed. One of them is a RemoteScope, when we deploy actor remotely on a given node. This is the only change which is necessary to perform the work on a remote node, the rest (actorOf) looks just the same, but will go over the network now. `context.watch` will watch this actor also over the network.
There is one more special detail in this actor: `implicit val s = context.parent`. This line is changes the meaning of who the sender of messages of this actor shall be. Therefore, all messages sent by this customer will appear to have been sent by its parent instead. Let's draw a diagram to show what happens.
![remote actor](./resources/remote_customer.png)
The whole program starts, because we instantiate the cluster main application Actor. We need to mark out in which actor system that happens. This is `ClusterMain` and it's guardian actor. The app then goes on to create the `receptionist`, and when a request comes in this one will create a `Customer`. Up to this point everything has been local. But the `Customer` deploys the controller into the `ClusterWorker` system, so we need to draw that as well. This system also has a `user` guardian and application. But this application does not do anything besides waiting for the termination of the program. This system has another thing which is interesting here. It has a remote guardian, and when the `Customer` deploys the controller, what it really does is it sends a message to this one to create the controller for it.

This one will first create a folder so that it can keep actors deployed from different other systems. And within it, it will create a marker for, that it was user, app, receptionist, customer, and so forth. And then it will finally create the controller actor. These here are not really actors. They are just names inserted such that, the controller can be found for remote communication. But the controller will be the child actor of the customer logically. So when in the controller we say `contexts.parent`, the message will go here. Then as in this link checker example, it will spawn getters as needed during the retrieval of the URL.

Now when we remote deployed the controller, we just need to supervise it. 
```scala
class Customer(client: ActorRef, url: String, node: Address) extends Actor {
    implicit val s = context.parent

    override val supervisorStrategy = SupervisorStrategy.stoppingStrategy
    val props = Props[Controller].withDeploy(Deploy(scope = RemoteScope(node))
    val controller = context.actorOf(props, "controller)
    context.watch(controller)

    context.setReceiveTimeout(5.seconds)
    controller ! Controller.Check(url, 2)

    def receive = ({
        case ReceiveTimeout => 
            context.unwatch(controller)
            client ! Receptionist.Failed(url, "controller timed out")
        case Terminated(_) =>
            client ! Receptionist.Failed(url, "controller died)
        case Controller.Result(links) => 
            context.unwatch(controller)
            client ! Receptionst.Result(url, links)
    }: Receive) andThen (_ => context.stop(self))
}
```
As usual we handle `ReceiveTimeout` in which case we unwatch the controller and send `Failed` because we didn't get anything in time. The deathwatch works exactly the same even though the controller is remote deployed, therefore in that case we also give a failure message. If we get back a successfull result we unwatch and send back the successful `Result`. After handling any of those 3 we need to stop and stop recursive so once parent actor is stopped the controller will be stopped too. 

Cluster has a leader node which is statically determined by this set of node addresses. They are sorted in a certain format and then always the first address which is in the membership list will be the leader. Since everybody agrees on who is in the list and on the sort order, everybody will see the same as the leader without the need for communication. 

### How cluster actually works
![node lifecycle](./resources/node_lifecycle.png)
When a cluster node wants to join it enters the joining state. When the leader of the cluster determined that all members of the cluster had seen all new nodes joining. When the program was done, the cluster main shut itself down and it did so by declaring that it wants to leave the cluster. This is expressed by going to the leaving state. This transition can be done by any node without the leader having anything to do with it. The information that the node wants to leave needs to be disseminated to all others, so that they are prepared to remove it from the membership list. This is required because all nodes need to have the same picture. In order to keep order, the leader determines when all have seen and moves the node to the exiting state. This is the signal for all the other nodes to remove this entry from their membership list at their next convenience. This was the sequence of states which the cluster main went through. 

#### Cluster needs failure detection

In order to form a cluster, everyone within it need to be in consensus about who is in it. That was the definition of it. The reachibility of all cluster members must be closely monitored. 
- Consensus is unattainable if some members are unreachable
- Every node is monitored using heartbeats from several other nodes
- A node unreachable from one other is considered unreachable for all
- Nodes can be removed to restore the cluster consensus

In order to reduce the number of required connections in big clusters, we apply a technique of that the neighbors monitor each other. We can enable order by nodes addresses and therefore every node can draw the same kind of ring with the same order of nodes in it. On the picture below we can see that each node monitors 3 next nodes in ring.

![nodes neighbours](./resources/nodes_neigh.png)

Usually we wait for 2-3 fails and after it we can detect that the node had failed. This spreads by gossip messages. After a some time nodes will split in several clusters (functioning and malfunctioning).

![nodes neighbours](./resources/cluster_splitting.png)

Going to back to this state diagram, unreachability is not really a state. It is a flag, because this node can become reachable after a period of time. But unreachibility is important enough and can happen at any time. In order to form a new cluster, without the unreachable node, that node needs to be communicated that it is leaving. But it is not the same kind of leaving as we've seen over here. Because, the node went down. This transition here is a policy decision, because you cannot in all cases say whether the node is really down or whether it is transiently unreachable, whether it will come back. So in the end the system monitor decides after which an unreachability period to move a node to down. After that happended and disseminated among all of the remaining cluster nodes, the leader removes the node. 
![node lifecycle](./resources/node_lifecycle_full.png)

#### Cluster and DeathWatch
Whenever a node is removed from the cluster, all the actors which are on it might still be alive if it's just a network partition. Even though the actors possibly have not really stopped yet, they are considered to be dead after the node has been removed. 
Actors on nodes which are removed from the cluster must be dead
 - allows clean-up of remote-deployed child actors, bc those actors will have no parent actor to handle their failures
 - decision must be takend consistently within the cluster
 - once Terminated was delivered the actor cannot come back. Once a node has been removed and DeathWatch has fired on all the actors within it, this node cannot join the cluser again because that would mean that the actors could also start sending messages againt and that would violate the contract of the terminated message. Therefore, a noe which was removed from teh cluster needs to be restarted completely before it can join again.

Lifecycle monitoring is important for distributed fail-over:
 - delivery of Terminated is guaranteed
 - this is only possible because it can be syntesized when needed
In Akka message delivery is not guaranteed, but still we do guarantee that the terminated message arrives. The need for this guarantee should have become clear, and the reason why we can afford to implement this guarantee is that terminated can just be synthesized.
It can be put together even though the sender is currently not able to do that anymore.

To tri this out, we will modify the ClusterWorker, not to rely on the MemberRemoved event, but instead to use DeathWatch so that it terminates once the cluster main stops.

```scala
val cluster = Cluster(context.system)
cluster.subscribe(self, classOf[ClusterEvent.MemberUp])
val main = cluster.selfAddress.copy(port = Some(2552))
cluster.join(main)

def receive = {
    case ClusterEvent.MemberUp(member) => 
        if (member.address == main) {
            val path = RootActorPath(main) / "user" / "app" / "receptionist
            context.actorSelection(path) ! Identify("42")
        }
    case ActorIdentity("42", None) => context.stop(self)
    case ActorIdentity("42", Some(ref)) => context.watch(ref)
    case Terminated(_) => context.stop(self)
}
```

We need an actor to which we can apply DeathWatch, we know one which will exist, that is the receptionist. We can construct the receptionist's actor path because we have the main address, and we can form a root actor path from it. And then these paths have a little DSL built in where you can separate by slashes, the path components. We can resolve this actor path using actorSelection and sending it an Identify message with some random tag here, and If we cannot find, if we cannot resolve the receptionist, something must be wrong, we stop the program. 

### Eventual consistency 
Everything takes time. It takes time for a node joining, to disseminate the information among the rest of the cluster, and then it takes time until the welcome message arrives, and so on.
And the decisions are taken in some consistent fashion, but they are not taken immediately. The cluster is one example of a system which is eventually consistent.


#### Strong consistency

After an update completes, all reads will return the updated value
```scala
private var field = 0
def update(f: Int => Int): Int = synchronized {
    field = f(field)
    field
}

def read(): Int = synchronized {field}
```
This is called strong consistency, and it can be easily achieved here, because we are doing it in a fully local system.

The locks offered by Scala's objects only work because we are executing all code here in the same Java Virtual Machine, on the same computer. But even so we have already discussed that this can be problematic because synchronization blocks the thread which wants to execute it potentially. And that is not good for CPU utilization.

We can remove the need to block the calling thread by moving the synchronization into another execution context.
#### Weak consistency
```scala
private @volatile var field = 0
def update(f: Int =. Int): Future[Int] = Future {
    synchronized {
        field = f(field)
        field
    }
}

def read(): Int = field
```

Here, this synchronized block is executed in a Future. That means that the update method, which takes this function, does not return the new value itself, it returns a Future of that new value and the thread can continue normally without having to wait for the update to occur. In order to properly publish the new value of the field, we write it back. And if we mark the variable as volatile, that makes sure that other threads see the updated value when they do their next read. Therefore, no synchronization is necessary for the read message here. By removing the blocking nautre of the update method, we have also removed strong consistency, because calling update and then immediately calling read will probably not give us back the new value.

It will take some time until the new value is visible. This is called **weak consistency** and that means that after an update, certain conditions need to met until the new value is visible. And this is called the **inconsistency window**.

#### Eventual consistency
Eventual consistency is a special kind of weak consistency.

First of all, it takes awhile until all reads return a consistent value, and second, this only really works once the system becomes quiescent. So once the updates stop, and then, after awhile, once everyone has communicated the new value then everyone is on the same page and consistency will be achieved.

```scala
case class Update(x: Int)
case object Get
case class Result(x: Int)
case class Sync(x: Int, timestamp: Long)
case object Hello

class DistributedStore extends Actor {
    var peers: List[ActorRef] = Nil
    var field = 0
    var lastUpdate = System.currentTimeMillis()

    def receive = {

    }
}
```

This actor has the main purpose to hold this field, but the goal is to have different actors collaborate and share the value of this field, such that updates can be performed on any of these. And eventually, the update will be seen on all other copies. For the outside protocol, we need an update command, which gives a new value. Then we need a get request, and a result Reply type. The actors will also use Sync messages and share their existence using Hello. 

How can we serialize updates of the field? The diffuculty is that there are multiple copies of this field, one in every actor and when the update happens in two different actors, they need to decide which value to keep. 

![simplified conflict](./resources/conflict_simpl.png)

Let's look at simplified example. We have an actor D1 here, for distributed store one, and D2. And D1 get's a command, Update(42). And D2 gets another, Update(43). They will both locally process these messages, and afterwards, they will talk and sync up. And they need to keep either the 42 or the 43.

In order to serialize these updates, we use here the current time in milliseconds. Whenever an update is made, we capture a timestamp and associate it with this value of the field. And then when D1 tells D2, I have a new value for you, then D2 can check, is this actually a new value, or do I have something better?

```scala
case class Update(x: Int)
case object Get
case class Result(x: Int)
case class Sync(x: Int, timestamp: Long)
case object Hello

class DistributedStore extends Actor {
    var peers: List[ActorRef] = Nil
    var field = 0
    var lastUpdate = System.currentTimeMillis()

    def receive = {
        case Update(x) =>
            field = x
            lastUpdate = System.currentTImeMillis()
            peers foreach (_ ! Sync(field, lastUpdate))
        case Get => sender ! Result(field)
        case Sync(x, timestamp) if timestamp > lastUpdate =>
            field = x
            lastUpdate = timestamp
        case Hello =>
            peers ::= sender
            sender ! Sync(field, lastUpdate)
    }
}
```

This is a simplified implementation of this protocol. Every time an update comes in, we write to the field, and we take a current timestamp. When a get request comes in we reply with the current field value.
When we need to handle a Sync message, we compare timestamps and take the most recent version. 

#### Actors and eventual consistency
Actors and eventual consistency are deeply related concepts. 
- **An actor forms an island of consistency**. Everything you do within an actor is sequentially consistent Everything happens like it was on a single thread.
- **Collaborating actors can at most be eventually consistent**. They can never be strongly consistent. They can never agree on a shared thing. Because they always need to exchange messages, and a message takes time to travel. So by the time it arrives this date which should be agreed upon might already have changed Therefore, collaborating actors can at most be eventually consistent.
- **Actors are not automatically eventually consistent**. But that is not automatically the case. You need to work to make the behavior eventually consistent. Looking back at the distributed store which we have just seen that had a few flaws. For example, if updates come within the same millisecond then the merge was not properly resolved.
- **Event consistency requires eventual dissemination of all updates**. Another problem is that message delivery is not guaranteed, but there was no resend mechanism. And this would also be problematic because eventual consistency requires that eventually all updates are disseminated to all interested parties. And this usually implies that there needs to be a resend mechanism. Another way to do it has been shown in the cluster which does not resend because of failures. It just resends pessimistically so to speak. The gossip messages are always sent no matter whether we know that the other party, for example, needs the update or it might all be old news.
- **Need to employ suitable data structures, for example Commutative Replicated Data Types**

#### An example CRDT
The cluster membership state is a convergent data type:
- DAG of states
- Conflicts can always be resolved locally
- Conflict resolution is commutative

Let's look at the example:

![crdt example](./resources/crdt.png)

When cluster nodes exchange gossip messages, these messages contain the state of all nodes currently known to the cluster. So for each of the members, there will be one of these values. For example, we had a cluster here with a few nodes. And this one, learns something about say, node four. And this one learns something else about node four, then the information might spread. And let's say, node two learned the green thing first and then the red thing. These are two new informations about node four which need to be merged. It'll say that the red information was, that it was down, and the green one was, that it was leaving.

Because this is such a nice graph, we can give an order to all the possible pairs of states, saying that down takes precedence over leaving, because you can go from leaving to down, but not the other way.
This gives us the property that a conflict, for example, as here, can be resolved locally, first. And the second one is that conflict resolution is commutative. So it does not matter whether you learn this first, or that. The merge result will always be down in this case. And this is the property which makes the cluster communication eventually converge, even if conflicting information was injected at several different points in the cluster.

### Actors composition

#### The type of an Actor
The interface of an Actor is defined by its accepted message types, the type of an Actor is structural. This structure may change over time defined by a protocol. Superficially current Actor implementations are unityped:
 - sending a message is (Any => Unit)
 - behavior is PartialFunction[Any, Unit]

These types do not restrict what you can send. In a sense, you could say that actors are not usefully typed.

This limitation is no a funamental. While it is true that an actor can change its behavior arbitrarily in principle, most Actors you write will not do that. They will either always accept the same set of messages, at least the types or it might change. But then the total sets of all messages ever accepted will be bounded. In all actors we have seen so far, we defined the complete set of messages they accept in their companion object for example. This makes it possible to statically verify that when you send a message to an actor, it could in principal at some point, have a chance to handle it. Means you could reject a large number of errors.

However, the type system will never allow us to express which state an actor currently is in. So we can deal with the complete set of messages, but we cannot statically determine whether an actor will process a given message for example. There are many reasons for that. One of them is that **delivery is not guaranteed**, for example. 

![alice and bob example](./resources/alice_bob.png)
Another one is that actors may interact with several different parties and if we have here, for example, Alice and Bob both talking to Charlie. Then Bob may send a message changing the state, so that when Alice sends a message, that might find Charlie in a state which is not suitable for that message anymore. And since this can happen concurrently, it is not known statically at compile time. So the compiler will not be able to catch those kinds of mistakes. However, having fully typed Actors is something which we do not have yet, so let us use what we have right now.

Given that sending a message to an Actor does not return anything and the types currently are not really restricted. It is obvious that Actors cannot be composed like functions. Instead we have seen that actors collaborate like human organizations. You can split down a task into sub tasks and have each actor perform its own part, and then the results are taken together, they are composed, and reply to the client which asks for the operation to be performed.

This means that actors are not composed like functions, they're composed at the protocol level. This gives the actor the possibility to do everything you can think of conceptually using function composition.
So you can translate the values passed into a function. You can translate its return value. And the same thing you can do with messages flying back and forth. You can also call multiple actors and
then aggregate the results.

**An Actor can**
- translate and forward requests
- translate and forward replies
- split up requests and aggregate replies

But what protocol composition can do which, function composition cannot do, is, for example, to alter the frequency with which an actor is called, or the precise timing of the calls. Or perhaps dropping messages, injecting new messages. There is a lot more freedom here but this freedom is bought by least type safety.

#### Composition patterns
##### The Customer pattern

- fundamental request-reply pattern
- customer address included in the original request
- allows dynamic composition of actor systems

The most fundamental pattern is request-reply. It is so common that in Akka the sender reference is kept exactly for that purpose. Because you so often need to reply and for that you need to know to whom to reply. 

![customer pattern](./resources/customer_pattern.png)

Let us say that Alice wants to know something of Bob, and sends a message. Together with the sender information, this enables Bob to reply. But we have seen other uses, for example Bob could forward the message to Charlie for example. So it's basically this. And forwarding means that the sender reference stays the same. So the sender of that message here, will not be Bob, it will be Alice. And then the reply from Charlie, goes directly back to Alice. This means the fact that simply putting the sender reference in the message and letting it travel with it allows dynamic composition of actors. Bob can decide dynamically whom shall handle the request. Then the reply will go back to the one requesting it.