---
layout: post
title:  "Typeclasses for Beginners"
date:   2020-10-25 12:37:00 +0100
tags:  scala fp typeclasses
---

**P.S This post is for beginners, experienced developers can skip this :)**

In this post, I will try my best to avoid using conventions so as not to confuse the beginner, I just want to solely pass the message across without confusing the reader with general conventions.

As a Scala developer, I had always heard about Typeclasses, and it seemed really hard to understand, but I’ll try to simplify it so much so other beginners can easily understand this great feature in Scala.

Before I define what TypeClasses are, let’s start with a small problem and hope that by trying to solve the problem will help you understand the concept better.

Say you have a class, possibly from a library and you want to add some functionality to the class knowing you cannot define a method in the class. How will you be able to add functionality to that class ?. See the code sample below for a simple example of the internal workings of a sample library.

<script src="https://gist.githubusercontent.com/samuelorji/049d353d437fd0a08730bbba504c0828.js"></script>

<a href="https://gist.githubusercontent.com/samuelorji/049d353d437fd0a08730bbba504c0828">View this gist on GitHub</a>

Say all you can do is create objects of type Cat and Parrot, but then it hits you, a parrot can also talk like a human, but the writers of the library did not put that into consideration, so how do I add that functionality ( to speak like a human ) to objects of type Parrot or any animal of my choice but not to all animals without having to modify the source code the library ?

This is where TypeClasses can help out, **it’s basically extensibility of a closed Model or adding functionality to a closed Model.** So now, how do I add my method to the type Parrot when I can’t modify the source code.

To begin, what’s the best way to represent our functionality, it’s very obvious that a trait will be best as we can easily mix it into classes that we want to add functionality to. Let us define a trait ( a behaviour or functionality ) that we want to be able to mix into classes of our choice.

<script src="https://gist.githubusercontent.com/samuelorji/bc649e7713e5abdab4414d079eac0bb4.js"></script>

<a href="https://gist.githubusercontent.com/samuelorji/bc649e7713e5abdab4414d079eac0bb4">View this gist on GitHub</a>

Now, this is a trait which when mixed in with type **A**, will make ‘**A**’ HumanLike ( or have human tendencies ). **AN INSTANCE OF THIS TRAIT IS OUR TYPECLASS** \[ You see the trait takes in a type \].

For us to use this typeclass on the object of our type, and this must be remembered, **there must be a concrete definition of our typeclass for that particular type in scope for it to work and its preferable that it’s implicit.** If you don’t understand implicit parameters, for now, just imagine it one way as defining a global parameter marked as implicit that the compiler injects into multiple places in your code where that parameter is needed, so you don’t have to keep passing that same parameter as long as it’s marked as implicit in its signature. It’s way more complicated than that but that’s the basic idea.

Now to create our type class for the parrot object, we need to create an object of type HumanTendency ( our typeclass ) which will take a parrot as a type parameter, since that is the class we want to extend its functionality. Implementing this gives us a typeclass for parrots only.

<script src="https://gist.githubusercontent.com/samuelorji/fd001ed520d59ac1efb11e753679200d.js"></script>

<a href="https://gist.githubusercontent.com/samuelorji/fd001ed520d59ac1efb11e753679200d">View this gist on GitHub</a>

for beginners, the above is a short way of creating a class that extends HumanTendency and then creating an object of that class. This is just an easier method.

There are two ways to actually use this typeclass, and I’ll start will my least favourite. Which basically defines a function that takes a parrot as a parameter and its type class as another argument(preferably implicit), it then calls the method **makeHumanLike** already defined by the typeclass with the parrot object as a parameter.

<script src="https://gist.githubusercontent.com/samuelorji/9a4efadc9a5208b5d5b49e3ee45008bc.js"></script>

<a href="https://gist.githubusercontent.com/samuelorji/9a4efadc9a5208b5d5b49e3ee45008bc">View this gist on GitHub</a>

Voila, we have subtly added functionality to type parrot with this function, now if we pass in a parrot object into this function, with its typeclass in scope, we can call makeHumanLike on the parrot object.

The Second method which is my favourite involves some implicits :)

First, we define an implicit class that implicitly converts a type Parrot into an arbitrary type ( ToParrot ) that already has a predefined method (let us call it speakLikeHuman) that takes a type class as a parameter, similar to the previous method, but what makes this method really cool is that now, instead of calling a function, we can now call **parrot.speakLikeHuman** as if the speakLikeHuman method were already a part of the Parrot Class, whereas it wasn’t.

So, to understand the implicit class, when we call **parrot.speakLikeHuman,** the compiler checks if the **speakLikeHuman** method is defined for type parrot. When this fails, it then checks if there’s an implicit conversion from type parrot to an arbitrarily defined class in scope that has the method **speakLikeHuman** defined. That explains the implicit class that takes a type Parrot and behind the scenes, converts it to type ‘**ToParrot**’ that already has the method **speakLikeHuman** already defined.

<script src="https://gist.githubusercontent.com/samuelorji/2644f93cdfdfb3784358594d98717153.js"></script>

<a href="https://gist.githubusercontent.com/samuelorji/2644f93cdfdfb3784358594d98717153">View this gist on GitHub</a>

From above, both will do the same thing, but with the second method, it feels a little more natural to call a method on an object. But if we call the `speakLikeHuman` method on a Cat, it will not work, **unless there is a Cat typeclass in scope.**

Now, we can make JSON representations of custom classes using typeclasses without having to add the toJson method to the classes directly. The possibilities are endless.

**Conclusion:** This article is supposed to be a very gentle intro to typeclasses, I hope beginning Scala developers will have a better understanding of this topic.
