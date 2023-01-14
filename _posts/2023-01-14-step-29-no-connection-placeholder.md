---
title: "Step 29: Add assets and a placeholder for no internet connection"
layout: post
categories: kmm ios android swift ui compose
--- 

Remember the task from Step 27?

<i>If you try opening one of the locally saved recipes without internet connection, you’ll notice that there is no image. Fix it by adding a “No Internet” placeholder instead of an empty space when the user can’t download the image.</i>

Here’s my solution.<br>
First, find an image you’d like to show when image libraries fail to load a placeholder image from the Internet.<br>
In Android place this image in <b>res/drawable</b>:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/0.png" alt="drawable folder" width="300"/></div>

In iOS first create an <b>Asset Catalog</b> by clicking on <b>“New File…”</b> and adding an Asset catalog.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/1.png" alt="add asset catalog" width="500"/></div>

You’ll see it in your app structure:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/2.png" alt="ios structure" width="300"/></div>

Then just drag and drop the chosen image to this catalog:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/3.png" alt="asset image ios" width="300"/></div>
 
Now we can use this image in our apps.<br>

In Android simply extend the `HeaderImage` loading with `error`:

{% highlight kotlin %} 
AsyncImage(
   model = image,
   modifier = Modifier.size(200.dp),
   contentDescription = null,
   error = painterResource(id = R.drawable.no_pizza)
)
{% endhighlight %} 
 
Turn off the connection. Build and run the Android app. You’ll see if the recipe has an image, but it couldn’t be downloaded, there’ll be a no-connection placeholder instead:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/4.gif" alt="android no-connection" width="300"/></div>
 
Similarly, in iOS we can extend `KFImage` logics in `RecipeDetailView` with `.placeholder`:

{% highlight swift %} 
if (state.recipe?.images.isEmpty == false){
                KFImage(URL(string: state.recipe?.images.first?.image ?? ""))
                    .resizable()
                    .placeholder { Image("no_pizza").resizable() }
                    .frame(width: 180, height: 180)
            }
{% endhighlight %} 
 
However, if you try running the app you’ll get an error about a missing AppIcon. <br>
Looks like we accidentally deleted something 😱 <br>
But no worries, it’s a good excuse to add an AppIcon to our app!

First, go to https://appicon.co and create a set of icons for you Android and iOS apps.
 
Then simply take <b>Assets.xcassets</b> and drag it to your iOS project.<br>
Then remove the old `Assets` catalog and place your no-connection image to the new one.
 
Also change the name of the iOS app here:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/10.png" alt="ios app change name" width="500"/></div>
 
Build and run the app again. Turn off the internet connection and you’ll see a no-connection placeholder image.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/6.gif" alt="ios no-cnnection" width="300"/></div>
 
And the app has a new icon and name now as well:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/11.png" alt="ios ap icon" width="300"/></div>
 
Finally, let’s also add an app icon and change the name in the Android app.<br>
Go to <b>File -> New -> Image Asset</b>

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/7.png" alt="android image asset" width="500"/></div>
 
Then choose your image
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/8.png" alt="choose image asset" width="500"/></div>
 
Finally, in <b>Android Manifest</b> specify the app icon (and also the app name):

{% highlight kotlin %} 
android:label="@string/app_name"
android:icon="@mipmap/ic_launcher"
{% endhighlight %} 
 
Build and run the app again. In your Android apps you’ll see the new icon and a new name:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-29/9.png" alt="android app icon" width="300"/></div>
 
Now, KMPizza has grown to be a <b>rather functional</b> KMP app. It has a Ktor backend and a shared business layer. What’s more, it also shares the ViewModel layer between iOS and Android. 
 
But in software development there’s always <b>room for growth</b>. Since I started with KMP a lot of <b>new libraries</b> appeared, which make the development easier. There are also different approaches to a shared local database - for example, using Realm. There are <b>other Kotlin backend frameworks</b> as well, like Kotlin Spring, Micronaut, Vert.X. In general, KMP offers a great playground for your future experiments and projects. 

Meanwhile, there are already teams that are using KMP in prouduciton to gain a <b>competitive edge</b>. Once KMP goes stable, there will be even more reasons to adopt it. 

May be your KMP project will be the next big thing? 
 
<b>Use your imagination and never stop learning!</b>