### Notes on async

Async computations allows to run operations in parallel. It runs the computation on another unit, which means it does not block anything.

#### Callbacks

```scala
def makeCoffee(coffeDone: Coffee => Unit): Unit = {
    val coffee = ...
    coffeDone(coffe)
}

def coffeeBreak(): Unit = {
    makeCoffe { coffee => 
        drink(coffee)
    }
    chatWithColleagues()
}
```

A synchronous type signature can be turned into an asynchronous type signature by:
- returning `Unit`
- taking as parameter a continuation defining what to do after the return value has been computed

Simpliest example of such change:
```scala
def program(a: A): B //sync

def program(a: A, k: B => Unit): Unit //async
```
If we want to run an async computations to run in some order, we must just pass callbacks into callbacks. But this could lead to so-called callback hell...

```scala
def coffeBreak(breakDone: Unit => Unit): Unit = ...

def workRoutine(workDone: Work => Unit): Unit = {
    work { work1 =>
        coffeBreak { _ =>
            work { work2 =>
                workDonw(work1 + work2)
            }
        }
    }
}
```

#### Exceptions in async
We cannot just throw exceptions, because we could not be aware of exceptions happening on other thread of execution or on a remote machines. Therefore, we need a way to signal failures to call sides.
One way of obtaining this is wrapping an argument of passed function in `Try`

```scala
def workRoutine(work: Try[Work] => Unit)
```
#### Summary for callbacks
- Allow us to sequence asynchronous computations
- Have complex type signatures
- CPS is good.

### Futures
Callback may be confusing because result type is passed as a function parameter, and overall function return type is `Unit`. 
Migrating from CPS to Future:
```scala
def program(a: A, k: B => Unit): Unit
```
1. Currying the continuation parameter
```scala
def program(a: A): (B => Unit) => Unit
```
2. Introducing type alias
```scala
type Future[+T] = (T => Unit) => Unit

def program(a: A): Future[B]
```
3. Add failure handling
```scala
type Future[+T] = (Try[T] => Unit) => Unit
```
4. Reify type alias into a proper trait
```scala
trait Future[+T] extends ((Try[T] => Unit) => Unit) {
    def apply(k: Try[T] => Unit): Unit
}
```
5. Rename `apply` to `onComplete`
```scala
trait Future[+T] extends ((Try[T] => Unit) => Unit) {
    def onComplete(k: Try[T] => Unit): Unit
}
```

Simplified API of Future:
```scala
trait Future[+A] {
    def onComplete(k: Try[A] => Unit): Unit
    def map[B](f: A => B): Future[B]
    def flatMap[B](f: A => Future[B]): Future[B]
    def zip[B](fb: Future[B]): Future[(A, B)]
    def recover(f: Exception => A): Future[A]
    def recoverWith(f: Exception => Future[A]): Future[A]
}
```
Important note here is that `zip` does not creates dependency between the two Future values (unlike `flatMap`). Therefore, two zipped futures are evaluated concurrently.

#### Comparing zip and flatMap
```scala
def makeTwoCoffees(): Future[(Coffee, Coffee)] = makeCoffee() zip makeCoffee()

def makeTwoCoffees(): Future[(Coffee, Coffee)] = makeCoffee().flatMap { fst =>
    makeCoffee().map(snd => (fst, snd))
}
```
With `zip` the second call to makeCoffee is performed not after first call, but with `flatMap` the second call is performed only after second. It is happens because `flatMap` introduces sequentiality between its operands: the right hand side is always evaluated after left hand side. However, it is still possible to emulate `zip` throught `flatMap`:
```scala
def makeTwoCoffees(): Future[(Coffee, Coffee)] = {
    val fstEval = makeCoffee()
    val sndEval = makeCoffee()
    fstEval.flatMap { fst => sndEval.map(snd => (fst, snd))}
}
```

Because `Future` has `map` and `flatMap` methods, the `for-comprehension` syntax could be used. 

coffeeBreak with Futures:
```scala
def coffeeBreak(): Future[Unit] = {
    val fstCoffeeDrunk = makeCoffee().flatMap(drink)
    val eventuallyChatted = chatWithColleagues()

    fstCoffeeDrunk.zip(eventuallyChatted).map(_ => ())
}
```
Returning `Future[Unit]` instead of just `Unit` is useful, because it allows callers of the method to be notified when computation is finished or it has failed.

#### recover and recoverWith

Allow us to turn failed `Future` in a successfull one. Signatures are:
```scala
trait Future[+A] {
    def recover[B >: A](pf: PartialFunction[Throwable, B]): Future[B]
    def recoverWith[B >: A](pf: PartialFunction[Throwable, Future[B]]): Future[B]
}
```

#### ExecutionContext

Continuations are executed phisically on threadpool. API of `Future` allows users to supply a context of execution. For instance, user can choose to allocate single thread to execute all continuations, which means they couldn't be executed in parallel, or user can choose to allocate a thread pool in order to get parallelism. 

In practice the `ec` is passed via an implicit parameter. Therefore, at the beginning of your program you usually import the `ec` you want to use:
```scala
import scala.concurrent.ExecutionContext.Implicits.global
```

It is good idea to use a threadpool with as many threads as the underlying machine has physical processors and this is exactly what this default `ex` provides.

#### Turning a Callback-API to Future:
Callback API:
```scala
def makeCoffee(
    coffeeDone: Coffee => Unit,
    onFailure: Exception => Unit
):Unit
```
Future API:
```scala
def makeCoffeeFuture():Future[Coffee] = {
    val p = Promise[Coffee]()
    makeCoffee(
        coffee => p.trySuccess(cofeee),
        reason => p.tryFailure(reason)
    )
    p.future
}
```
`Promise` is a write-once container for `Future`

### Summary on `Future`
- Future type is an equivalent alternative to CPS
- Offers convenient transformation and failure crecovering operations
- map and flatMap allows sequentiality