---
title: "Step 21: Add images to iOS app with Kingfisher image library" 
layout: post
categories: kmm ios kingfisher
--- 

In <b>Step 15</b> we started creating our iOS App <b>Swift UI</b>.<br>
So far it has only a main view with the list of recipes. <br>
However, if you try running the app, it’ll crash with an error.
 
That’s because we need to use a special version of coroutines library <b>(native-mt)</b>.<br>
Add the following dependency to your project.
In `Versions.kt`:

{% highlight kotlin %}  
const val COROUTINES_MT = "1.6.3-native-mt"
{% endhighlight %} 
 
In `Common` object:
{% highlight kotlin %} 
const val COROUTINES = "org.jetbrains.kotlinx:kotlinx-coroutines-core:$COROUTINES_MT"
{% endhighlight %} 
 
In shared `build.gradle.kts` within the `commonMain` sourceSet:
{% highlight kotlin %} 
// Coroutines
implementation(Versions.Common.COROUTINES) {
   version {
       strictly(Versions.COROUTINES_MT)
   }
}
{% endhighlight %} 
 
Also add the following to your project `gradle.properties`:
{% highlight kotlin %} 
kotlin.native.binary.memoryModel=experimental
kotlin.native.binary.freezing=disabled
{% endhighlight %} 
 
These settings will enable the new memory model management.<br>
Run the app again and you’ll see a list of recipe titles.

Now let’s make our iOS app look more like our Android app by adding other cool features.
 
First, add an image loading library.<br>
- Go to `File > Add Packages`

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-21/1.png" alt="add packages menu" width="300"/></div>

- Search for `https://github.com/onevcat/Kingfisher.git`

- Select "Up to Next Major" with "7.0.0"

- Add the package to iosApp

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-21/2.png" alt="add package to iosApp" width="800"/></div>

You have a new package dependency now

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-21/3.png" alt="new package dependency" width="200"/></div>

Add a new file to the recipes folder: `RecipeView`.<br>
Here we’ll describe how we want one `Recipe` to look like in the list of recipes.

{% highlight kotlin %} 
import SwiftUI
import shared
import struct Kingfisher.KFImage [1]
 
struct RecipeView: View {
    
    var item: RecipeResponse
    
    var body: some View {
        HStack(alignment: .center) {
            KFImage(URL(string: "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"))
                .resizable()
                .frame(width: 64, height: 64, alignment: .center)
                .cornerRadius(8)
                .padding(.trailing)
            VStack(alignment: .leading) {
                Text(item.title)
                    .font(.body)
            }
        }
    }
}
{% endhighlight %} 
 
<b>[1]</b> Don’t forget to import `KFImage` from the `Kingfisher` imaging library, for now we’ll use it with a hardcoded placeholder image<br>
<b>[2]</b> The recipe view will hold `Recipe` as an entity<br>
<b>[3]</b> `HStack` is like a `Row` and `VStack` is like a `Column` in Jetpack Compose.<br>
 
Now you can replace the `Text(recipe.title)` in `RecipesView` with the following:
{% highlight kotlin %} 
   RecipeView(item: recipe)
{% endhighlight %} 
 
As a result, you should be able to see a nice list with image placeholders for every recipe.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-21/4.png" alt="ios recipes list with images" width="300"/></div>

<b>In the next step</b> we’ll proceed with <b>RecipeDetailView</b> and add <b>Navigation</b> to our iOS App.
