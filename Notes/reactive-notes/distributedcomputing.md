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
** Actor communication is async, one-way and not guaranteed **
Actors are distributed by their nature, they model exactly what the network gives us. Instead of takin the local model and trying to extend it to the network, actors tooke the inverse approach: looking at the network model and using that on the local machine. 

** Actor encapsulation makes them look the same, regardless where they live **

** Actors are "Location transparent", hidden behind `ActorRef`.
Regardless where they live it will be the same sending a message to them, this is what we call "Location transparency". The current semantics in feature set offered by Akka Actors have been reached by rigorously treating everything as if it were remote. And all features  which cannot be modeled in this world, were removed.

** As a result, writing distributed program on Actors is the same way as writing the local program. The code itself will not look much different. **

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