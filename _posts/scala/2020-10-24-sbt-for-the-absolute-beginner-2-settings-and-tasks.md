---
layout: post
title:  "SBT for the absolute beginner 2 (Settings and Tasks)"
date:   2020-10-24 12:37:00 +0100
tags:  scala sbt
---

[**P.S. If you are a newbie, I highly encourage you to read part 1 of this post.**]({% post_url scala/2020-10-23-sbt-for-the-absolute-beginner-1-basics-and-dependency-management %})

You may be like me and may have wondered at some point how on earth you could run your Scala or Java programs from SBT with arguments, when all I did when running my program was pass the **run** command to SBT. So how do I pass JVM parameters or compiler parameters? Well, all of that can be done in your **build.sbt** config file as shown below

<script src="https://gist.github.com/samuelorji/5c77e97f908ee9a624938e19b8a7a11b.js"></script>

<a href="https://gist.github.com/samuelorji/5c77e97f908ee9a624938e19b8a7a11b">View this gist on GitHub</a>

With this configuration, we can pass arguments to the scala compiler and even the JVM. So whenever SBT is used to run your project, the commands added will be used to run your program.

I’m sure the `scalacOptions` is new as well as the `javaOptions`. They are actually just settings that define the build of your project that you can then tweak to your satisfaction. [A pretty exhaustive list of Scala compiler options can be seen here.](https://docs.scala-lang.org/overviews/compiler-options/index.html)

Both of these options are called **Settings** and before we explain what they are, let us try to understand some things.

1. SBT can be thought of as a really huge collection of key-value pairs represented as `key := value`
2. Everything in SBT is Scala code and commands like `run`, `compile`e.t.c are represented as functions (well, kinda) within SBT
3. Everything in SBT is either a Task or a Setting

Now, let us look at two very fundamental concepts in SBT called **Tasks** and **Settings.**

**SETTINGS**

These are similar to functions in Scala in that they have return values, but don’t take any input arguments. They are similar to `lazy vals` in Scala. So the scala code inside a setting is computed once, and will always give the same return value for every other computation. They are defined by a setting expression `SettingKey[A]` where **A** _is_ the return type of the setting. Some already familiar settings include `name`, `organization`, `version`, e.t.c in our `build.sbt`

**TASKS**

Tasks are also similar to functions, and quite similar to Settings in that they have a return value, but they differ in the sense that any code inside a Task is calculated each time the task is called, similar to a `def` in scala. Another difference is that there are some kinds of tasks that take input arguments called InputTasks, but we won’t deal with that here. They are defined by a task expression with `TaskKey[A]`where **_A_** is the return type of the task. Some already familiar tasks include `run`, `compile`, `clean` e.t.c

Now that we have a pretty faint idea of tasks and settings, It’s now good to know that SBT defines a DAG( directed acyclic graph ) of tasks and settings during project builds. What that means is that settings and tasks can be made to depend on each other to represent your build and SBT understands and respects how you define your dependencies :)

At this point, I bet you are already bored …..show me the code right?

> To make this post relatable as usual, we are going to define our own SBT command or task just like `clean` or `compile` called `zip` which will simply zip the **_build.sbt_** file and place it in the root of our project. Pretty easy, but it should cover both Settings and Tasks.

let’s see how to define a setting in SBT. Add this to your `build.sbt` file

`lazy val srcFile = settingKey`[java.io.File`](“Path to build.sbt”)`

What did we just do?. We just defined (well not technically) the key of a setting by calling the function `settingKey`. This is basically how keys are defined. This function takes a return type in the `[]` brackets (which can be gotten by calling `.value` on the key) and also a short description in the `()` bracket. So with this, we defined the key of a setting which along with other settings constitutes of:

1. Name: ours is **srcFile** in this case
2. Return Type: ours is **java.io.File**
3. Description: ours is `Path to build.sbt`

Settings and Tasks both have a `.value` method that returns the value of either the setting or the task key. In a setting, it is always the same result as computation is done once, but in a task, it causes a re-computation of the task.

Now, If we run the SBT shell in interactive mode, we will get an error simply saying that `srcFile` is uninitialized which makes perfect sense as we just defined a setting without assigning it a value or supplying an implementation. We can do that easily by assigning it a value using the now known operator `:=`

`srcFile := new java.io.File("Path/to/build.sbt")`

What did we just do here, remember that we defined our `srcFile` setting key to return a `File` as a value, so we just assigned a `File` as a value to the setting key, so when the setting key is called, it will just return the value that was assigned to it.

> One pretty easy way to understand the concept of defining and assigning settings is to imagine the call to `settingKey[T](“”)` as defining the key of the setting which represents the type of the key and using `:=` to represent the actual value of the setting `[assigning it a value`], similar to how functions are defined in scala.

For all we care, we could have done something like this

```scala
srcFile := {  //some other computation   
  val currentTime: String = new java.util.Date().toString  
  println(time)  
  new java.io.File("Path/to/build.sbt") //return type : File   
}
```

The only thing in this case is that when `srcFile` is called the first time, it will run the computation first and print the current time also but not on other calls to `srcFile`. So not really what we want right :)

Another way we can assign a `File` as a value to the `srcFile` key is

```scala
//key         //value    
srcFile := baseDirectory.value / "src"
```

Woah Woah Woah… what on earth did we just do, err …. you said we could only assign a File to the `srcFile` setting, then why am I not seeing any call to java’s File class. Well, do you remember us saying that calling `.value` on a setting returns the actual value of the setting which is a concrete type?. Well in SBT, there are some predefined settings and one of them is `baseDirectory` which defines the root (or base ….duurh ) directory of your project, which is a java File, so calling `.value` on the setting returns a File, now the `/` function is just syntactic sugar for constructing file paths, returning new files when called. So in reality, what we did is similar to `new File("base_directory/src")` You can therefore see that what we did is actually valid. We actually did assign a file as a value to the `srcFile` setting.

> [**See other predefined SBT settings here**](https://github.com/sbt/sbt/blob/develop/main/src/main/scala/sbt/Keys.scala)

Now, provided we are in our sbt shell, if we type in `srcDir` , it should actually show you the path of the file assigned to it, in my case it shows this.

```bash
> srcFile  
[info]/tmp/phony/build.sbt
```

We have defined our setting which is basically a variable that points to the `build.sbt` file we want to Zip, that’s not all as all this command is a setting, pretty similar to a variable and all it does is return a file path, now let us create the task that will do the actual work of Zipping the file for us.

To do that, similar to settings, to define a task, we define a task key in our `build.sbt`with its return type and a description message, we then assign it a value to the task key that definitely has a return value of some type.

<script src="https://gist.githubusercontent.com/samuelorji/76a5918d66cb3d2ddb9f25b5426036fc.js"></script>

<a href="https://gist.githubusercontent.com/samuelorji/76a5918d66cb3d2ddb9f25b5426036fc">View this gist on GitHub</a>

Now, let us look at what we did, first we defined the key of our task in the first line with a return type of Unit and a really short description. Next, we wrote some scala code, describing the actual task and then assigned it to the `zip`task key with `:=`as its value

You may see that similar to the setting, we actually called another predefined sbt setting called `slog` whose return value is the logger defined for the sbt shell, we also called our own defined setting `srcFile` (yaaay … ) and got its value, as well as sbt’s predefined `baseDirectory` setting. The actual implementation of the

Now, start a new sbt shell to capture the changes made to the`build.sbt`file or enter `reload`in the previous shell to reload sbt and thus capture the recent changes in the project. Now, if we type in the command`zip` into sbt, it should zip the files for us. My own console output is

```bash
> zip  
[info]zipping files from /tmp/phony/build.sbt  
[info]zipping files to /tmp/phony/build.sbt.zip  
[success] Total time: 0 s, completed Dec 15, 2019 3:42:04 PM
```

![](/assets/images/zipper.png)

We can actually see the file that was zipped to our base directory with the name **build.sbt.zip** as defined in **build.sbt** file.

> Settings should be seen as defining variables, while tasks should be seen as defining functions or commands to SBT.

With this, we have both defined settings and tasks. We have more or less created our own sbt command called `zip` just as how other commands like `clean` and `compile` exist. So now, we can get SBT to run our own custom command that we created :)

# **Dependency in tasks and settings**

Remember we said settings can depend only on other settings, while tasks can depend on both other settings and other tasks. Tasks are lazily evaluated, while settings are evaluated at project load. So what this means is that when SBT starts, all settings are evaluated, but tasks are only evaluated when they are called. This makes sense as it is pointless to automatically compile my code once I start SBT. That said, tasks can be made to depend on other tasks.

> Let’s imagine another zip implementation that requires fetching data from a remote source and adding to the src directory before zipping. To the end user, all he/she has to call is `zip` and not be bothered about how the file is to be fetched. In our case, we will define a task that will simulate fetching a config file from a remote source and zip it along with the `build.sbt` file

This is a really simple implementation:

<script src="https://gist.githubusercontent.com/samuelorji/58681595768e5ad37597479462ad4d6d.js"></script>

<a href="https://gist.githubusercontent.com/samuelorji/58681595768e5ad37597479462ad4d6d">View this gist on GitHub</a>

Let’s explain what we just did here.

First, we defined a task `fetchConfFile` that simulates fetching data from a remote source, in our case we just slept for a second and created a new file named `application.conf` . Then we kinda modified our `zip` task to actually zip the file gotten from the `fetchConfFile`task along with the `build.sbt` file.

Now, if we run `zip`, I get this console output
```bash
> zip  
[info] fetching file ......   
[info] done fetching file ....   
[info] zipping files from /tmp/phony/build.sbt  
[info] zipped files to /tmp/phony/build_and_config.zip  
[success] Total time: 0 s, completed Dec 15, 2019 5:01:40 PM
```

If you’re attentive, you’ll notice a difference in the console output, this for some reason runs the `fetchConfFile` task before the actual zip task, you should expect that it should at least log `zipping files from /tmp/phony/build.sbt` before running the `fetchConfFile`, but in reality, the `fetchConfFile` task was actually executed first despite the fact that the call to `logger.info` was written before it. This is in line with the DAG that we previously said SBT created on each project build. [**See the docs for a little more clarity**](https://www.scala-sbt.org/1.x/docs/Tasks.html)**.**

> As we should know by now, tasks can be made to depend on other settings and tasks as well as settings depending only on other settings. This thus means that if lets say a task depends on another task, then the task it depends on will be executed first before the actual task itself.

To see dependencies of a task or setting, there is a handy command in sbt called `inspect` that actually outlines the name of the task, its description, dependencies e.t.c
```bash
> inspect zip  
[info] Task: Unit    
[info] Description  
[info]  zip files  
[info] Provided by:  
[info]  ProjectRef(uri("file:/tmp/phony/"), "playground") / zip  
[info] Defined at:  
[info]  /tmp/phony/build.sbt:52  
[info] Dependencies:  
[info]  fetchConfFile  
[info]  baseDirectory  
[info]  srcFile  
[info]  sLog  
[info] Delegates:  
[info]  zip  
[info]  ThisBuild / zip  
[info]  Global / zip
```
The inspect command shows that `zip` is a Task with return type of Unit, it also shows the description supplied in the taskKey function. You can see the Dependencies for the `zip` task, how SBT knows that it depends on the `fetchConfFile` task, as well as `baseDirectory`, `srcFile` and `sLog` too.

Another pretty cool command is `show` which executes and prints the return value of a task or setting in sbt.
```bash
> show zip  
[info]fetching file ......   
[info]done fetching file ....   
[info]zipping files from /tmp/phony/build.sbt  
[info]zipped files to /tmp/phony/build_and_config.zip  
[info]()
```
You can see it actually executed the task and then printed out the return value `()` which is just Unit in scala.

Tasks and settings can also be modified, Imagine that we want to delete the existing zipped folder everytime we run `zip` , remember there is an sbt command called `clean` which basically deletes files produced by the build, such as generated sources, compiled classes, and task caches resulting from the `compile` task. Running `inspect clean` in your sbt console shows that it depends on a setting called `cleanFiles` which is simply a list of files to be deleted whenever the clean command is called. So what we can do is append our zipped file `build.sbt.zip` to the list of files to be cleaned, so that whenever we are cleaning the project, we also clean the zipped file we generated from the `zip` command and generate a new zipped file everytime as shown below.
```scala
cleanFiles := (baseDirectory.value / "build.sbt.zip")                                           +: cleanFiles.value
```
## **FINAL THOUGHTS**

We have but just scratched the surface of sbt, there are still other dimensions that I didn’t touch. My main goal was to provide a very basic understanding of SBT, as well as provide a good foundation for reading and writing **build.sbt** files.
