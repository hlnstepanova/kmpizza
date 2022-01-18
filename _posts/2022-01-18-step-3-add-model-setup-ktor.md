---
title:  "Step 3: Add a Recipe model + set up a Ktor server" 
layout: post
categories: setup ktor introductory
--- 
Now that we’ve made sure the app is alive, let’s start building up our backend.
We’ll have a recipe collection, so we’ll need a recipe, which consists of a list of ingredients and a list of instructions.

A simplified model of our recipes will look like this:

{% highlight kotlin %}
data class Recipe(
   val title: String,
   val ingredients: List<Ingredient>,
   val instructions: List<Instruction>
)

data class Ingredient( 
   val name: String,
   val amount: Double,
   val metric: String
)

data class Instruction( 
   val order: Int,
   val description: String
)
{% endhighlight %}

Let’s create a model directory in our backend module and add our data classes in a file called `Entity.kt`.

<img src="{{site.baseurl}}/assets/images/step-3/1.png" alt="add models" width="400"/>  

That's good enough of a model. Realistic, but not too complicated.<br>
What's next?<br>
<img src="https://www.icegif.com/wp-content/uploads/chef-pusheen-icegif.gif" alt="recipe" width="150"/>

Next we will set up a simple, but powerful server using the Ktor framework. Ktor is easy to use and has great multiplatform capabilities, which will benefit us a lot in the future when we get down to writing our networking client for the apps. First, let’s define Ktor dependencies in `Versions.kt`.

{% highlight kotlin %}
object Versions {
   const val KTOR_VERSION = "1.6.7"

   object Jvm {
       const val KTOR_AUTH = "io.ktor:ktor-auth:$KTOR_VERSION"
       const val KTOR_WEB_SOCKETS = "io.ktor:ktor-websockets:$KTOR_VERSION"
       const val KTOR_CLIENT_APACHE = "io.ktor:ktor-client-apache:$KTOR_VERSION"
       const val KTOR_SERVER_NETTY = "io.ktor:ktor-server-netty:$KTOR_VERSION"
       const val KTOR_SERIALIZATION = "io.ktor:ktor-serialization:$KTOR_VERSION"
   }
}
{% endhighlight %}

After syncing your gradle files, go to the backend module and if you have `build.gradle` instead of `build.gradle.kts`, change the extension, as we’re going to use Kotlin DSL throughout the project. Then add Ktor dependencies. Your `build.gradle.kts` should look like this:

{% highlight kotlin %}
plugins {
   kotlin("jvm")
}

dependencies {
   // Ktor
   implementation(Versions.Jvm.KTOR_CLIENT_APACHE)
   implementation(Versions.Jvm.KTOR_SERIALIZATION)
   implementation(Versions.Jvm.KTOR_SERVER_NETTY)
   implementation(Versions.Jvm.KTOR_AUTH)
   implementation(Versions.Jvm.KTOR_WEB_SOCKETS)
   implementation(Versions.Jvm.KTOR_CLIENT_APACHE)
}

{% endhighlight %}

Now we are ready to create our server. We’ll use embeddedServer and load configuration settings from the `application.conf` HOCON file, which we’ll place in the application resources. With Ktor you can configure various server parameters. When you create your server with embeddedServer you’ll need to specify the parameters in the constructor. Add `Server.kt` file to your backend and write the following:

{% highlight kotlin %}
import io.ktor.server.engine.commandLineEnvironment
import io.ktor.server.engine.embeddedServer
import io.ktor.server.netty.Netty

fun main(args: Array<String>) {
   embeddedServer(Netty, commandLineEnvironment(args)).apply {
       start()
   }
}
{% endhighlight %}

Now let’s add an `application.conf` file to resources directory in backend/main (you won’t need other resources in your backend module, so you can easily delete all other directories that were created automatically, like the drawables and mipmaps, as well as test, androidTest, AndroidManifest and proguard-rules):

<img src="{{site.baseurl}}/assets/images/step-3/2.png" alt="add application conf" width="400"/>  

Add the following to your `application.conf`:
```
ktor {
deployment {
  port = 8080
  port = ${?PORT}
}

application {
  modules = [dev.tutorial.kmpizza.backend.MainKt.module]
}
}
```

Here we configure our port: we’ll use 8080 by default for local development and in the future will be able to set the PORT variable from our environment with port = ${?PORT}.

Pay attention to the module path matching your package name!

Now add `Main.kt` to your backend, next to `Server.kt`.
This file will contain one function, corresponding to the module that we’ve just specified in our `application.conf`.

{% highlight kotlin %}
import io.ktor.application.*

internal fun Application.module() {
 
}
{% endhighlight %}

You will also need to specify your Server in the `build.gradle.kts`. Now your backend `build.gradle.kts` file should look as follows:
{% highlight kotlin %}
application {
   mainClass.set("dev.tutorial.backend.ServerKt")
}

plugins {
   kotlin("jvm")
   application
}

dependencies {
   // Ktor
   implementation(Versions.Jvm.KTOR_CLIENT_APACHE)
   implementation(Versions.Jvm.KTOR_SERIALIZATION)
   implementation(Versions.Jvm.KTOR_SERVER_NETTY)
   implementation(Versions.Jvm.KTOR_AUTH)
   implementation(Versions.Jvm.KTOR_WEB_SOCKETS)
   implementation(Versions.Jvm.KTOR_CLIENT_APACHE)
}
{% endhighlight %}

As always, sync the dependencies. 

Now we’re done with the boring part: this basic Ktor setup will be enough to begin with.
Soon we’ll be able to test our backend. 











