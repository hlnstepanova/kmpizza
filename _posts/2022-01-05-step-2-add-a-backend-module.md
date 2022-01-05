---
title:  "Step 2: Add a Backend module + buildSrc for dependency management" 
layout: post
categories: setup introductory
--- 
What do we need to serve our pizza?

Right! A backend.

And you know what, we can do that with Kotlin. So let’s start.

Add another module to our basic setup: backend. Click on the project and choose New -> Module. Name it “backend”. Choose “No Activity” when asked to create one.
 
<img src="{{site.baseurl}}/assets/images/step-2/1.png" alt="add backend module" width="650"/> 
<img src="{{site.baseurl}}/assets/images/step-2/2.png" alt="name backend module" width="650"/>   

If you check out the project settings.gradle.kts, you will see that the backend module is included in the project now.
{% highlight kotlin %}
rootProject.name = "kmpizza"
include(":androidApp")
include(":shared")
include(":backend")
{% endhighlight %}

As our recipe collection is going to grow into a big project for multiple platforms, we’ll need to keep our dependencies clean and organised. To do so, we’ll setup buildSrc for dependency management. Add a new directory to the root folder and name it  “buildSrc”. 

<img src="{{site.baseurl}}/assets/images/step-2/3.png" alt="add buildSrc" width="350"/>   

In this module create Versions.kt and build.gradle.kts to match the following file structure:
<img src="{{site.baseurl}}/assets/images/step-2/4.png" alt="add versions" width="300"/>  

In Versions.kt  file you wil keep all the dependencies for different modules in this project.

Add the following to the build.gradle.kts file:
{% highlight kotlin %}
plugins {
   `kotlin-dsl`
}

repositories {
   gradlePluginPortal()
   maven("https://dl.bintray.com/kotlin/kotlinx/")
}
{% endhighlight %}

To make sure we haven’t broken anything yet, sync the project, choose androidApp configuration and run the app.

<img src="{{site.baseurl}}/assets/images/step-2/5.png" alt="test androidApp" width="200"/>  

You’ll see a “Hello, Android!” on your device, which is an initial setup that comes with the KMM Application Android Studio Project.

If you want to try it in XCode, open iosApp.xcworkspace. 

<img src="{{site.baseurl}}/assets/images/step-2/6.png" alt="test iosApp" width="400"/> 

Then run it on a simulator and you’ll see a “Hello, iOS!” screen. Great, everything is still working 😉 

Or is it? If you try running it on a real device you may get an error `Signing for "iosApp" requires a development team. Select a development team in the Signing & Capabilities editor.` 
Let’s quickly fix it.
Select the project name in XCode and go to Signing&Capabilites in the project editor. There change your team to Personal Team, something like this:

![signing and capabilities]({{site.baseurl}}/assets/images/step-2/7.png)

Now you may see another error if you try running the app again:
`Command PhaseScriptExecution failed with a nonzero exit code`
😱

No worries, we’ll fix it too. First, you need to sync the gradle project in Android Studio, then go to the iosApp directory in your terminal and run `pod install` there. 

Now try running again. 
If you get a warning like this

<img src="{{site.baseurl}}/assets/images/step-2/8.png" alt="trust developer ios" width="200"/>

Then go to Settings -> General -> Device Management and choose to trust your developer account.

Run the app again. Finally we see the greeting on a real device!

And just like this, one step at a time we’ll continue building our multiplatform app. 🍕








