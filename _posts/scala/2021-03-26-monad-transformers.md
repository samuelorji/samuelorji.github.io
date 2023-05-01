---
layout: post
title:  "Monad Transformers"
date:   2021-03-26 12:37:00 +0100
tags: cats fp functional-programming scala
---

Before we start talking about monad transformers, let's talk about why we may need them.  
Let's say we define the following classes and functions in this contrived example:

{% highlight scala %}
  case class User(name : String) extends AnyVal
  case class Order(order : String) extends AnyVal
  case class DeliveryDetails(detail : String) extends AnyVal

  def getUser(name : String) : Future[Option[User]] = ???
  def getOrder(user : User) : Future[Option[Order]] = ???
  def getDeliveryDetails(order : Order) : Future[Option[DeliveryDetails]] = ???
{% endhighlight %}

If we want to get the delivery details of a particular user, we can easily do this via a for-comprehension:

{% highlight scala %}
def deliveryDetails : Future[Option[DeliveryDetails]] =  {
    for {
      maybeUser <- getUser("user")
      maybeOrder <- maybeUser match {
        case Some(user) =>
          getOrder(user)
        case None => Future.successful(None)
      }
      maybeDeliveryDetails <- maybeOrder match {
        case Some(order) => 
          getDeliveryDetails(order)
        case None => Future.successful(None)
      }
    }  yield maybeDeliveryDetails
  }
{% endhighlight %}

It's easy to see that there's a pattern of pattern matching (excuse my pun) on the optional values, and as the for-comprehension gets complicated, we soon start seeing nested pattern matching on type constructors.

Now, in our case, we are only concerned about the success case of these type constructors. If we didn't have nested type constructors, It would have made our for-comprehension a lot easier without having to do so much pattern matching.

Our life could also have been made easier, if the scala compiler allowed for this kinda syntax:

{% highlight scala %}
def deliveryDetailsOpt : Future[Option[DeliveryDetails]] = {
    for {
      user <- maybeUser <- getUser("person_1") 
      order <- maybeOrder <- getOrder(user)
      deliveryDetails <- maybeDeliveryDetails <- getDeliveryDetails(order)
    } yield deliveryDetails 
  }
{% endhighlight %}

But Sadly, the scala compiler doesn't really permit that :)

What we're actually looking for is a way to work with nested type constructors. In our case, we have `Future[Option[A]]` and we want to abstract over the Future and `map` or `flatMap` over the optional value.

Here's where monad transformers can be really helpful as they help us combine operations of several monads into one single monad.

_Typically, a monad transformer is created by generalising an existing monad. Applying the resulting monad transformer to the identity monad yields a monad which is equivalent to the original monad (ignoring any necessary boxing and unboxing)._

In our example, we have two "Monads" (not entirely true, but for the sake of this article), a `Future` and an `Option` and because monads don't compose, we have to find a better way to achieve composability.

In our example, we are dealing with an `Option`, that is wrapped in another type constructor. We need to find a way to abstract over that outer type constructor and deal directly with the optional value in a composable manner.

Let's define a simple type that will help achieve this composability:

{% highlight scala %}
 case class FutOpt[A](value : Future[Option[A]]){
    def map[B](f : A => B) : FutOpt[B] = {
      FutOpt(value.map(_.map(f)))
    }
    
    def flatMap[B](f : A => FutOpt[B]) : FutOpt[B] = {
      // if the option is empty, return a Future.successful(None)
      // else call the flatMap function on the non empty option
      FutOpt(value.flatMap(_.fold(Future.successful(None : Option[B]))(f(_).value)))
    }
  }
{% endhighlight %}

What we've done is quite simple, we've extended the map function to apply to both the `Option` contained within the `Future` and the Future itself, similarly, the flatMap function is applied to both type constructors.

A slightly more expressive way to write the flatMap is shown below:

{% highlight scala %}
def flatMap[B](f : A => FutOpt[B]) : FutOpt[B] = {
  // if the option is empty, return a Future.successful
  // else call the flatMap function on the non empty option
  def next : Future[Option[B]] = value.flatMap {
    case Some(result) => f(result).value
    case None => Future.successful(None : Option[B])
  }
  FutOpt(next)
}
{% endhighlight %}

With this class, we have abstracted over the outer type constructor which is a `Future` in our case and provided a way to functionally interact directly with the inner type constructor without having to pattern match as seen using our previous example:

{% highlight scala %}
def op : FutOpt[DeliveryDetails] = {
    for {
      user <- FutOpt(getUser("person_1"))
      order <- FutOpt(getOrder(user))
      deliveryDetails <- FutOpt(getDeliveryDetails(order))
    } yield deliveryDetails
  }

  val deliveryDetails = op.value
{% endhighlight %}

Voila, we have simplified our for-comprehension by defining a simple monad transformer which is just another bigger monad "composed" of other monads.

We could also define a simple monad transformer for List of options too:

{% highlight scala %}
case class ListOpt[A](value : List[Option[A]]){
    def map[B](f : A => B) : ListOpt[B] = {
      ListOpt(value.map(_.map(f)))
    }

    def flatMap[B](f : A => ListOpt[B]) : ListOpt[B] = {
      ListOpt(value.flatMap(_.fold(Nil : List[Option[B]])(f(_).value)))
    }
  }
{% endhighlight %}

If we really look at the function signatures of these toy monad transformers, we can see that they all have a map and flatMap method that enables them being used in for-comprehensions. We could easily abstract that functionality into a monad trait:

{% highlight scala %}
 trait Monad[F[_]] {
    def pure[A](a: => A): F[A]
    def map[A, B](fa: F[A])(f: A => B): F[B]
    def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
  }
{% endhighlight %}

We can then define our generic optional monad transformer thus:

{% highlight scala %}
  case class MonadFOpt[F[_],A](value : F[Option[A]])(implicit monad : Monad[F]){
    def map[B](f : A => B) : MonadFOpt[F,B] = {
      MonadFOpt(monad.map(value)(_.map(f)))
    }

    def flatMap[B](f : A => MonadFOpt[F,B]) : MonadFOpt[F,B] = {
      MonadFOpt(monad.flatMap(value)(_.fold(monad.pure[Option[B]](None))(f(_).value)))
    }
  }
{% endhighlight %}

We have defined a generic optional monad transformer, that can abstract over any F[_] type provided there's a monad instance in scope.

We can rewrite our previous examples like this:

{% highlight scala %}
  implicit val futMonad = new Monad[Future] {
      override def pure[A](a: => A): Future[A] = Future.successful(a)

      override def map[A, B](fa: Future[A])(f: A => B): Future[B] = fa.map(f)

      override def flatMap[A, B](fa: Future[A])(f: A => Future[B]): Future[B] = fa.flatMap(f)
    }
  
  
  val op : MonadFOpt[Future, DeliveryDetails] = {
    for {
      user <- MonadFOpt(getUser("person_1"))
      order <- MonadFOpt(getOrder(user))
      deliveryDetails <- MonadFOpt(getDeliveryDetails(order))
    } yield deliveryDetails
  }
  
  val deliveryDetails: Future[Option[DeliveryDetails]] = op.value
  {% endhighlight %}

If you understand all of this, congratulations, you understand the basics of Monad transformers, We could also define transformer for other type constructors such as Either, Try e.t.c.

In Practice, you may not need to write your own monad transformers, most FP libraries around offer these transformers such as the [cats library](https://github.com/typelevel/cats/blob/1d99ec77e4a42d0b2682ca31c7a7dde1bf6fe0fb/core/src/main/scala/cats/data/OptionT.scala).
