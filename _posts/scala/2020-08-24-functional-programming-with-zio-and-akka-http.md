---
layout: post
title:  "Functional Programming with ZIO and Akka HTTP"
date:   "2020-08-24 12:37:00 +0100"
tags: akka akka-http fp functional-programming scala ZIO
---

![](/assets/images/1*3nl072xy4JWgfqiSlBRjvA.png)

Huge disclaimer, I’m not yet some FP expert who can boldly explain category theory and all them really fancy terms, but this article explains how I was able to build a pretty basic and functional Todo CRUD app using ZIO and Akka Http.

Granted, there are some functional Http clients such as [http4s](https://http4s.org/), but I’ve been using Akka Http for ages and I just needed some FP library that I could integrate with Akka Http, and since there is so much buzz around ZIO, I decided to use this pretty awesome library. So here goes 😄.

Before we get into some code, I just wanna explain what I think an effect is as this is one term that will be used a lot in this article. As easily explained in this [blogpost](https://alvinalexander.com/scala/what-effects-effectful-mean-in-functional-programming/), `an effect is what a Monad models, or handles`, when you see an Option, you immediately think of a value that may or may not exist, similarly, a future represents some async computation that may or may not succeed, a List represents zero or more items of some type. What you’ll notice is that all these data structures, encapsulate some sort of data model and have some well-defined API’s that help you manipulate their contents or effects :).

To understand monads, [check out this article](https://samuelorji.com/understanding-monads-in-scala/)

Another thing to note about effects is that it’s easily composable such that you can build a whole program using just effects. These effects describe how your program is run but does not start computation until you force it to. You can build the next billion-dollar software as an effect composed of other effects, but until you run the effect, nothing is done, I believe this helps make your code a little deterministic as well as easy to test.

> One analogy I developed for understanding effects is an electronic circuit that may be a large or small circuit that has been properly connected and designed with light bulbs and other electrical apparatus. The resulting circuit is the effect, and with electric circuits, until an energy source is added to the circuit, no bulb will be lit up or an electrical appliance powered on. Similarly, large or small effects need to be run in order to see the result of the effect. Pretty cool analogy right :)

ZIO is a pretty cool FP library to use, it helps you think of your whole program as composed effects, where the output of one effect solely depends on the input it was given, from the environment or another effect. The beauty of this is that similar to how when designing the blueprints of a house, it’s mandatory that lines are connected to form a circuit, ZIO ensures that your program is sealed tight, forcing you to deal with errors that may be thrown during execution. This for me was really awesome, a little stressful, but it ensured that I was able to write a pretty bulletproof application.

Now in ZIO, the mother effect, is `ZIO[R,E,A]`, which is a functional data structure that describes an effect which may or may not depend on an environment of type `R` , produce a value of type`A` or fail with an error of type`E`. If I want to write a program that fetches an Item from a database, First thing is that I need a database to search from, and once I have access to that database, I can either get the result which is the `Item` or something horrific may happen in the process of fetching that item which will result in an `Error` . If I model this as an effect using ZIO I’ll have,

> `ZIO[Database,Error,Item]`

This basically easily explains to the reader that for this effect to run, you need to supply it a database, and when you do, you’ll either get the item you were searching for, or you may get an error, and the error and item are a package deal, in the sense that you must deal with both results at some point in the program. Another example is saying, I wanna print something to a console, This may sound weird, but you need a console to print to first, it may be your own local machine or on some remote terminal. The remote server may bring some added complexities, but let’s look at our own local machine. We can model this pretty simple effect as `ZIO[Console,Nothing,Unit]`, because, we need a `Console` to print to, and we really don’t expect that computation to fail, hence the `Nothing` error type, and the return type if printing to the console was successful is `Unit` . A type alias for an effect that cannot fail exists that’s called `URIO[R,A]`which is equivalent to `ZIO[R,Nothing,A]` if you’re not into typing too much :)

If I wanna create an effect that adds two small Integers, I’ll model it such, `ZIO[Any,Nothing,Int]`, because well we don’t need anything to add two integers and we don’t expect such small computation to fail, and if successful, it can only return an Int.

To really get the hang of it, I’ll highly suggest taking a look at the [ZIO docs](https://zio.dev/docs/overview/overview_index)

> Using ZIO, I actually got to see applications as just a combination of logic that may or may not require an input and produce some sort of result based on either the success or failure of a computation. A static HTML page doesn’t need input from the end-user to display it’s information, while a dynamic one sure does. Whenever you get a 40X or a 50X HTTP status code, it just goes to show that some error happened.

Enough talk, Let’s see some code. To begin, Let’s model our Todo, and the kind of errors that may be thrown, doing that we have

> P.S, I’m going to be overly explicit and break some single expressions into multiple expressions to drive some points home.

```scala
sealed trait Todo  
case class TodoName(name : String) extends Todo  
case class TodoItem(id :Int , name :String) extends Todo  
  
sealed trait TodoError extends Throwable  
//describes errors such as id not found, id exists  
case class ToDoItemError(errorMsg : String) extends TodoError  
  
//descibes invalid query errors  
case class QueryError(errorMsg : String) extends TodoError  
  
//describes errors such as lost database connections  
case class DbError(errorMsg : String) extends TodoError
```

We’ve described our data model for our application, what this implies is that our program must either return a success value of type `Todo` or an error type `TodoError`.

Now, since we are going to use a database let’s model our database. It’s a trait that describes an effect.

```scala
object Database {  
  trait Service {  
    def create(todo : TodoName) : IO[TodoError,Unit]  
    def findTodo(id : Int) : IO[TodoError, TodoItem]  
    def updateTodo(id : Int, todo : TodoName) : IO[TodoError, Unit]  
    def deleteTodo(id : Int) : IO[TodoError,Unit]  
    def getAll: IO[TodoError,List[TodoItem]]  
  }  
}  
trait Database {  
  def database: Database.Service  
}
```

If you didn’t check out the docs, the data type IO may seem weird, didn’t we say that all effects in `ZIO` start with `ZIO[_, _, _]`, well, `IO[E,A]` is a type alias which according to the docs represents an effect that has no requirements, and may fail with an `E`, or succeed with an `A`, because let’s face it, a database doesn’t need any requirement, you supply it a query and it either returns what you are looking for or returns an error. Now we’ve described our database operation as an effect.

> Just looking at the above, I can already see some ease in testing, I can easily pass a mock of this object without having to mock an actual database, If you’ve tried mocking dynamo, you’ll be so relieved 😅

The next thing which I believe will start tying things up is our API, which is also an effect. Remember I said, our program is composed of effects that are connected to each other in some way. We define an HTTP endpoint, which returns an effect that interacts with the database effect or some other effect.

```scala
val getTodoByIdRoute_: Route = path("todo" / IntNumber) { id =>  
  get {  
    complete {  
      val effect: ZIO[Database, TodoError, TodoItem] =   
                  ApplicationController.findTodoById(id)  
        
      val result: IO[TodoError, TodoItem] = effect.provide(database)  
      result  
  
    }  
  }  
}
```

This route is for `GET` requests and we see that an ID is supplied. What I want to draw your attention to is the fact that we have an Application Controller whose return type isn’t a Future as I’m used to, but it’s an effect and what it describes is some computation that requires an object of type Database, and will either return an error of type `TodoError` or a result of type `TodoItem`. We’ll take a look at the `findTodoById` method in a bit, but you’ll see that we transformed that effect into another effect via the method `provide`. What that `provide` method did was simply fulfill the requirement for this effect by providing it with a database from scope of type `Database` needed for it to work. The result of that is an `IO[TodoError,TodoItem]` which having described before is an effect that has no requirement. When this effect is run, it doesn’t depend on anything to produce a result. Our controller initially returned an effect that needed a database, and we provided a database to that effect returning an effect that doesn’t have any requirement.

Now let’s take a look at a snippet of the ApplicationController.

```scala
object ApplicationController {  
  
  def addTodo(todo : TodoName) : ZIO[Database, TodoError, Unit] = {  
    ZIO._accessM[Database](_.database.create(todo))  
  }  
  
  def findTodoById(id : Int): ZIO[Database, TodoError, TodoItem] = {  
    ZIO._accessM[Database](_.database.findTodo(id))  
  }

  def getAllTodo : ZIO[Database,TodoError, List[TodoItem]] = {  
    ZIO._accessM[Database](_.database.getAll)  
  }

 // other methods.   
}

```

Let’s look at the `findTodoById` method, let’s explain it’s implementation a bit, the first thing we see is the `ZIO.accessM` method, which according to the documentation _effectfully accesses the environment of the effect._ In other words, it takes the requirement that should ideally be passed into it and provides you with that requirement which you can then use to produce another effect. This database requirement was supplied in our case using the `provide` method in the HTTP Api, although there are other ways to do this. A more explicit way of looking at that piece of code is this.

```scala
def findTodoById(id : Int): ZIO[Database, TodoError, TodoItem] = {  
  val effectfullyAccessDatabase =  ZIO._accessM[Database]  
    
  val useDatabaseToFindTodo : Database => IO[TodoError,TodoItem] =   
    db => db.database.findTodo(id)  
  
  effectfullyAccessDatabase.apply(useDatabaseToFindTodo)  
    
}
```

The return type of the `accessM` method here, is a partially applied function of type `Database => ZIO[Database,TodoError,TodoItem]` but if you look closely you’ll see that the function I applied has a return type of `IO` not `ZIO`, If you remember that `IO[E,A]` is a type alias for `ZIO[Any, E, A]` and because the type environment type R is contravariant in its type parameter, we don’t get a compile error. You can clearly see how we have composed two effects, the ApplicationController effect, and the database effect.

Now, let us see a snippet of how the database effect is implemented.

<script src="https://gist.github.com/samuelorji/3add9aea6de3609dedd3aa710992f20b.js"></script>

<a href="https://gist.github.com/samuelorji/3add9aea6de3609dedd3aa710992f20b">View this gist on GitHub</a>

I’m pretty sure this is a lot especially if you’re new to ZIO.

The first thing we defined is our actual database as a trait, with the only public field being an effect that represents our database object. The reason why I chose to make it an effect is to be able to show another example where we would have needed some requirements to satisfy an effect. Of course, there is an easier way to do this which I’ll describe later. Our `db` object is represented as a `UIO` which is an effect that doesn’t need any environment or requirement, can’t fail, and return a value of type A, `type UIO[A] = ZIO[Any,Nothing,A].` This effect was lifted from a `UIO` object using the `fromFunction` method defined on the object, the parameter of the function is `Any => A` which makes sense as the environment of this effect is of type `Any`. I also defined a `DatabaseDriver` trait that holds the database object, and mixes in the trait into the `PostgresDatabase` trait too.

Now if you look at the first method in the `PostgresDatabase` trait, it’s called `runQuery` and this just basically defines an effect that requires a DatabaseDriver and runs the query supplied to it. Typically, when you run a query using the driver we supplied, you get a `Future[QueryResult]` , but since we are programming solely with ZIO effects, we have to lift that Future into an effect, and luckily enough, ZIO has a method to do that for us with `ZIO.fromFuture` , the beauty of this is it easily helps you convert what would have been a `Future[QueryResult]` which may fail with a throwable into the already evident `IO[Throwable,QueryResult]` which is an effect that we can compose …… YAAAAAAAAY ! . Now anyone that wants to use this effect and get the intended result has to supply a `DatabaseDriver`

Next, let’s look at how we defined our `findTodoById` function, it’s pretty easy to reason, we call the `runQuery` function and then provide self as a requirement since the trait already mixes in the `DatabaseDriver` trait, that makes sense, when we get our result `dbResult` in the for comprehension, we parse the result into an Option of a `TodoItem`, now, that could suffice, but since we are programming solely with effects, we need to convert the Option into an effect, and we do that using `ZIO.fromOption`, which converts an `Option[A]` into an `IO[Unit,A]`,which makes perfect sense as an option doesn’t have any meaningful error message or attribute, hence the `Unit` return type in the `IO`.

You may have seen that I called mapError on the effect returned from the option, and the reason why I did that is pretty simple. The for comprehension contains two composed effects. The first one from the future that fails with a type `Throwable`, and the second one from the option that fails with a type `Unit`, and because these effects are composed, the final resulting effect contains either one of the errors from either effect, and due to the covariant position of the error type E, the resulting type of the composed effect is Any since the nearest super type of both error types is `Any`. Now, to avoid losing any type information, at compile time, I just map the error type from the option effect to my custom error. You may also think, now, we have two error types `Throwable` and `TodoError`, and the same issue should apply in this case, but I made the `TodoError` type extend `Throwable`. This now ensures that the resulting error of both composed effects is of type `Throwable`, which I then exhaustively pattern match on and generate a more concrete error type depending on the kind of throwable we got.

Before we move on to the next part, I remember saying there was an easy way to write the `runQuery` method without having to make the database object an effect. Here is an easier version that just takes the database object and wraps the future returned from running a query on the object into an effect.

<script src="https://gist.github.com/samuelorji/5230baa2ea11bdb2d9eaad941694e898.js"></script>

<a href="https://gist.github.com/samuelorji/5230baa2ea11bdb2d9eaad941694e898">View this gist on GitHub</a>

In this simplified version, I avoided having to create an effect that needed some environment type into an `IO`, that doesn’t have any requirements.

Because I wanted to learn the capabilities of ZIO, I tried to build effects from almost every standard library data type I could think of :)

Now let’s go back to our API, if you’re familiar with Akka Http, you know that the complete function takes (by name) a `ToResponseMarshallable` type as an argument, but what we returned from our Application controller is an effect, so how on earth did this code compile. That actually involved a little magic 😃

To understand what I mean, Let’s see what the compiler says about this HTTP route, If I didn’t perform my so-called magic.

```scala
val getTodoByIdRoute: Route = path("todo" / IntNumber) { id =>  
  get {  
    complete {  
      val effect: ZIO[Database, TodoError, TodoItem] =   
                  ApplicationController.findTodoById(id)  
        
      val result: IO[TodoError, TodoItem] = effect.provide(database)  
      result  
  
    }  
  }  
}
```

The simplified error message my IDE shows is this:

> Expected a ToResponseMarshallable but instead got an IO[TodoError,TodoItem]

This makes perfect sense as my effect simply returns an IO, but Akka Http doesn’t understand that data type. Now, for my magic trick, enter [Marshalling](https://doc.akka.io/docs/akka-http/current/common/marshalling.html).

Remember I said an effect is practically useless until it’s run, and when it’s run, it produces either the error value E or the actual value A, The first thing is to run the effect and deal with converting any of the result types into what Akka Http understands. In our case, AkkaHttp understands the response type `[HttpResponse](https://doc.akka.io/docs/akka-http/current/routing-dsl/directives/route-directives/complete.html)`. Here is a marshaller I wrote for two different effects that I’ll explain further.

<script src="https://gist.github.com/samuelorji/56aa034ed9d3ed6677514492410fee1a.js"></script>

<a href="https://gist.github.com/samuelorji/56aa034ed9d3ed6677514492410fee1a">View this gist on GitHub</a>

If you checked out the docs on marshalling, what I needed to do was define two marshallers, one from an error type `E` to an HttpResponse and another from a value type `A` to an HttpResponse also. If I tried to manually marshall my `IO` effect explicitly in the code, this is what I would have done in that GET request.

```scala
val result = ....

val httpResponse: Future[HttpResponse] =  
    Marshal_(result).to[HttpResponse]  
         
complete(httpResponse) 
``` 
  

But if I do this, the compiler throws this error, `No implicits found for parameter m : Marshaller[IO[TodoError,TodoItem],HttpResponse]` which makes sense since the compiler does not know how to marshall an `IO` effect into an `HttpReponse`. To fix that, I defined the generic `implicit def ioEffectToMarshallable`, First thing, I call `foldM` on the effect I want to run, to generate a new effect based on either of its return value, I then pass these new effects into implicitly passed marshallers of each return type depending on the resulting value of the effect. Now I’ve generated a new effect that will either fail with a type `Throwable` or pass with a `List[Marshalling[HttpResponse]]` which for the sake of this article we will just call an `HttpResponse` . Now that we have our effect, it’s time to run it and to do that, as usual, there are other ways I could have done this, but I decided to extend the `[Runtime](https://zio.dev/docs/overview/overview_running_effects)` ZIO trait, which is something I need to run my effect to generate concrete a value `E` or `A` (In my circuit analogy, the Runtime will be my battery or energy source) . If you pay attention, you’ll see that it takes a type parameter Unit, that just implies that it can only supply Unit to any effect that it runs, which is cool as the effect I want to run doesn’t need any other dependency since it’s an IO effect, This wouldn’t have been the case if I needed some other dependency as the effect won’t run if the runtime didn’t supply the requirement. With that in mind, I was able to run the effect using `unsafeRunAsync` because I want this effect to be run asynchronously, so for each incoming request, the effect is created and then run by the Runtime. Next, I attached more or less a callback where I passed the resulting value to the `Promise` I had initially created. You may be wondering where the `m1` and `m2` values are, well the success marshaller that converts my `TodoItem` into an `HttpResponse` is handled by this very handy line of code which just helps you marshall what’s in brackets to an HttpResponse.

```scala
implicit val todoItemFormatter = jsonFormat2(TodoItem)  
implicit val todoNameFormatter = jsonFormat1(TodoName)
```

The `m2` marshaller which marshalls my error of type `Throwable` into an HttpResponse is handled by this line of code, which basically transforms my `TodoError` into an HttpResponse, since `TodoError` extends `Throwable`, I can pass this marshaller to the where a `Throwable` is expected

```scala
implicit val errorMarshaller: Marshaller[TodoError, HttpResponse] = {  
  Marshaller { implicit ec => error   =>  
       val response = generateHttpResponseFromError(error)  
      PredefinedToResponseMarshallers.fromResponse(response)  
  
  }  
}
```

Now, writing this seems to fix the Http GET route, since we have now defined a marshaller for `IO` to `HttpResponse`, but if you pay attention, you’ll notice that we only have a marshaller of type `TodoItem` to `HttpResponse`, what if our effect when run, produces an `Int` or another random type, are we going to have to declare explicit marshallers for every type, surely, this is not a feasible solution. The effect below returns an `A` value of `Unit`, and I wasn’t ready to define a marshaller for Unit, here’s an example.

```scala
put {  
     entity(Directives.as[TodoName]){todo =>  
     val resultingEffect: IO[TodoError, Unit] =                                   - ApplicationController.updateTodoById(id,todo).provide(repo)

      complete(resultingEffect)  
        }  
      }
```

I had to find a pretty straight forward way to define a single marshaller that can handle all cases. Now, didn’t we just write a marshaller that could solve the `GET` issue, but for some reason, It doesn’t work, It tells me that there is no marshaller of type `IO[TodoError,Unit]` in scope, which makes perfect sense as we only have a marshaller of type `IO[TodoError,TodoItem]` in scope. I’m gonna admit, I got stuck here for a while, so I tried to figure out a way of combining all my effects into a single effect type and defining a ‘marshaller’ for that type, and the easiest way was to create an effect that also included the complete method like this,

```scala
put {  
  entity(Directives.as[TodoName]){todo =>  
    val resultingEffect: IO[TodoError, StandardRoute] =  
      ApplicationController.updateTodoById(id,todo)  
        .provide(repo).map{ _ =>  
      complete(HttpResponse(StatusCodes.OK))  
    }  
    resultingEffect  
  }  
}

```

Now, this way, if we can in someway generate a new effect that includes the call to `complete` , then all we have to do is define just one ‘marshaller’ for type `IO[TodoError,StandardResult]`, which we defined with this function.

implicit def standardRouteToRoute[E](effect: IO[E, StandardRoute])(implicit errToHttp: ErrorToHttpResponse[E]): Route

In that method, I just define a function that takes the effect that results in a `StandardRoute` and marshall the result into an HttpResponse. This way, I didn’t have to start defining marshallers for random types, as the singular result type of every effect I was expecting was a `StandardRoute`, because of the complete function. With this technique, I was able to design my routes with ease knowing that I had a marshaller in scope to avoid a compiler error.

Marshalling is a pretty huge topic and it’s gonna take way longer than a single post to explain everything I did.

Whew… that was a lot. With all of these, I was able to design a basic production-ready application composed purely of effects using the ZIO library…. This library is pretty straight forward to use, it properly explains errors and helps you think effectfully. I can’t wait to dig deeper and build more cool stuff with ZIO. I think I’m gonna try out ZIO streams next :).

Here are some resources I used to start with ZIO.

- [Tour of ZIO](https://www.youtube.com/watch?v=TWdC7DhvD8M)

- [wrapping impure code with ZIO](https://medium.com/@ghostdogpr/wrapping-impure-code-with-zio-9265c219e2e)

- [official docs](https://zio.dev/)

For the complete project, check it out [here](https://github.com/samuelorji/ZIO-AKKAHTTP)

P.S I went for something simple, and not correctness or perfect architecture 😃
