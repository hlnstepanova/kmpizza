---
title:  "Step 4: Build and test Ktor application with shadowJar" 
layout: post
categories: setup ktor shadowjar introductory
--- 
By now you must be very excited to try it out. At least I am. Teaching yourself software development always leaves room for surprise. What if it doesn't work? 😰

To build and test our backend we’ll use a jar file, which we will create with the help of [shadowJar](https://github.com/johnrengelman/shadow) library.

First, let’s add the Shadow Jar version to the `Versions.kt`:

{% highlight kotlin %}
const val SHADOW_JAR_VERSION = "7.1.1"
{% endhighlight %}

And then to configure it in backend `build.gradle.kts` add the following to the `plugins` section:
{% highlight kotlin %}
id("com.github.johnrengelman.shadow") version Versions.SHADOW_JAR_VERSION
{% endhighlight %}

This will automatically add a shadowJar task to our project and configure it to include the sources from the main sourceSet. We will also configure the task to set the name and output extension. To do so, first add an import to the top of your `build.gradle.kts`:
{% highlight kotlin %}
import com.github.jengelman.gradle.plugins.shadow.tasks.ShadowJar
{% endhighlight %}

And then add the configuration after your dependencies in `build.gradle.kts`:

{% highlight kotlin %}
tasks.withType<ShadowJar> {
   archiveBaseName.set("Backend")
   archiveClassifier.set("")
   archiveVersion.set("")
   isZip64 = true
}
{% endhighlight %}

Before we can test our backend let’s implement simple routing.
In the Application function install the routing:
{% highlight kotlin %}
import io.ktor.application.*
import io.ktor.routing.*

internal fun Application.module() {
   install(Routing) {
       api()
   }
}
{% endhighlight %}

Immediately create a separate `Api.kt` file, where we’ll list all the routes like this:

{% highlight kotlin %}
import io.ktor.application.*
import io.ktor.response.*
import io.ktor.routing.*

internal fun Routing.api() {
   hello()
}

private fun Routing.hello() {
   get("/hello") {
       call.respondText("Hello!")
   }
}
{% endhighlight %}

Now <br>
(1) rebuild <br>
(2) make the project<br>

<img src="{{site.baseurl}}/assets/images/step-4/1.png" alt="rebuild and make" width="400"/> 

If you get a junit error, then just delete the test folder from the backend module - we will not need it. And try again.

Then check the build folder: here it is, our Backend.jar 🤩 

<img src="{{site.baseurl}}/assets/images/step-4/2.png" alt="backend jar" width="300"/> 

Let’s try to start the backend from the command line.
First, open the terminal window and go to your KMP project.
As mentioned above, we can always pass required parameters to our server application through environment variables.
For example, we can override the port that we specified in `application.conf`:

`java -jar ./backend/build/libs/backend.jar -port=9000`

Then open your web browser at `localhost:9000/hello`.
If it greets you, then it’s a good sign!

<img src="{{site.baseurl}}/assets/images/step-4/3.gif" alt="hello" width="200"/> 

Congrats, we’ve created a simple backend REST API with our first get function!<br>
You can reward yourself with a slice of pizza now.














