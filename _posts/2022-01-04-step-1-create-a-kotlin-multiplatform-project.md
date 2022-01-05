---
title:  "Step 1: Create a Kotlin Multiplatform project in Android Studio" 
layout: post
categories: setup introductory
--- 
Everyone likes pizza, so let’s create a recipe app, where you can save all the yummy pizza recipes that you’ve tried and would like to recreate sometime in the future!

Our first step will be to create a Kotlin Multiplatform project.

1. Open Android studio, go to File -> New -> New Project and create a KMM Application. 
<img src="{{site.baseurl}}/assets/images/step-1/1.png" alt="create a KMM applicarion" width="700"/>  

2. Choose a name for the application and the package, define your save location and click Next.
<img src="{{site.baseurl}}/assets/images/step-1/2.png" alt="choose a name" width="700"/>  

3. In the following screen you’ll have to name your Android Application, iOS Application and shared Module. You don’t need to customize anything here, so just leave it as is and click Finish.
<img src="{{site.baseurl}}/assets/images/step-1/3.png" alt="app structure" width="700"/>  

Now we’ve got a basic setup for our future KMM project. This way we can keep all our project related Kotlin code in the same place.  As you can see, it has an androidApp, where you can work on your Android App just as you usually would. You also have an iosApp that holds your ios application, which you can work with in XCode. Moreover, you have a shared module now, which is divided into androidMain, commonMain and iosMain. CommonMain will be the source of all your shared logic within the app, and the other two will hold platform specific implementations for expect functions and classes.



