---
layout: post
title:  "Essential Effects 07: Managing Resources"
date:   2021-04-12 12:37:00 +0100
tags: cats fp functional-programming scala essential-effects
---


### Managing Resources

In Cats Effect, the Resource data type represents this acquire-use-release pattern to  
manage state. In other words, a Resource represents acquisition of an entity with its release function already implemented, which will be called once that resource has been used.

To acquire and use a resource, we call `Resource.make` which has this type signature:

`def make[F[_], A](acquire: F[A])(release: A => F[Unit])(implicit F: Functor[F]): Resource[F, A]`

We see it takes the entity to acquire wrapped in a type constructor and the release function. The result of this is a Resource, that we can then call use on. Once it's done being used, the release function kicks in

Let's take a simple and contrived example where we acquire and eventually release simple file api
{% highlight scala %}
object ResourcesExample extends IOApp {

  trait FileApi {
    def getContents : Array[Byte]
    def close : Unit
  }

  def getFileApi = new FileApi {
    override def getContents: Array[Byte] = {
      "hello there".getBytes
    }

    override def close: Unit =  {
      println("closing this file")
    }
  }

  val fileResource: Resource[IO, FileApi] = Resource.make(IO(getFileApi))(api => IO.pure(api.close))

  override def run(args: List[String]): IO[ExitCode] =
    for {
    _               <- IO(println("Let's check out file for your welcome message"))
    welcomeMessage  <- fileResource.use { fileApi =>
                        IO(fileApi.getContents)
                      }
    _               <- IO(println(s"Your welcome message is [${new String(welcomeMessage)}]"))
  } yield ExitCode.Success
}
{% endhighlight %}

The result of this program prints:

Let's check out file for your welcome message
closing this file
Your welcome message is [hello there]

We easily see that our mock file api is closed immediately after use, even before the next line in in the for comprehension.

It's important to know that the release function of a resource is called even if it throws an exception while being used

We could change our fil api exampe to throw an error:
{% highlight scala %}
 for {
    _               <- IO(println("Let's check out file for your welcome message"))
    welcomeMessage  <- fileResource.use { fileApi =>
      IO.raiseError[String](new Exception("are we gonna be released ????"))
    }
    _               <- IO(println(s"Your welcome message is [${new String(welcomeMessage)}]"))
  } yield ExitCode.Success
{% endhighlight %}
We still see that the resource is closed despite the exception being thrown

Let's check out file for your welcome message
closing this file
java.lang.Exception: are we gonna be released ????

##### Resource Composition

Resources also compose. since they are functional constructs, we can map or flatMap over them. Which means we can construct a new resource from a previous resource.

We can also use a resource within another resource:
{% highlight scala %}
  val intResource: Resource[IO, Int] = Resource.make(IO(42))(x => IO(println(s"releasing $x ")) *> IO.unit)
  val stringResource: Resource[IO, String] = Resource.make(IO("thor"))(x => IO(println(s"releasing $x ")) *>  IO.unit)

  val result : IO[Unit]= for {
    result <- intResource.use { age =>
      stringResource.use {name =>
        IO(s"name is $name, and age is $age")
      }
    }
    _ <- IO(println(s"result is $result"))
  } yield ()
{% endhighlight %}