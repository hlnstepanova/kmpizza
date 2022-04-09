---
title: "Step 12: Add a ViewModel layer to the shared KMM module" 
layout: post
categories: shared KMM viemwodel
--- 

To build UI we’ll use <b>Jetpack Compose</b> on Android and <b>SwiftUi</b> on iOS. To provide our UIs with data we’ll use <b>shared ViewModels</b>. First we’ll have to prepare our shared code by creating a sort of a view model interface, which will allow us to use viewmodels on both platforms.

Create a `util` package in `commonMain` and add the following file there:

<img src="{{site.baseurl}}/assets/images/step-12/1.png" alt="create KtorApi interface" width="350"/>

Create an `expect` class inside:

{% highlight kotlin %}
expect abstract class CoroutineViewModel() {
   val coroutineScope: CoroutineScope

   fun dispose()
  
   protected open fun onCleared()
}
{% endhighlight %}


As it’s an `expect` class we’ll have to write actual implementations on both platforms.

First go to `androidMain` package and add a `CoroutineViewModel` to the `util` package. 

<img src="{{site.baseurl}}/assets/images/step-12/2.png" alt="create KtorApi interface" width="290"/>

Our `CoroutineViewModel` in Android will just extend the regular `ViewModel` and inherit its scope and functionalities.

{% highlight kotlin %}
actual abstract class CoroutineViewModel : ViewModel() {

   actual val coroutineScope = viewModelScope
 
   actual fun dispose() {
       coroutineScope.cancel()
       onCleared()
   }
 
   actual override fun onCleared() {
       super.onCleared()
   }
}
{% endhighlight %}

You’ll see that some references are unresolved. This is because we haven’t imported necessary dependencies yet. Let’s add these to `Versions.kt`:

{% highlight kotlin %}
const val LIFECYCLE_VERSION = "2.4.1"
{% endhighlight %}

And to the Android object:
{% highlight kotlin %} 
const val VIEW_MODEL = "androidx.lifecycle:lifecycle-viewmodel-ktx:$LIFECYCLE_VERSION"
{% endhighlight %}

Then implement it in android source set in shared `build.gradle.kts` under `androidMain`:

{% highlight kotlin %}
implementation(Versions.Android.VIEW_MODEL)
{% endhighlight %}

Sync your project files.

Now we can resolve all references in the actual Android `CoroutineViewModel` implementation.

In `iosMain` create a new coroutine scope for the actual `CoroutineViewModel`:

{% highlight kotlin %}
actual abstract class CoroutineViewModel {
   actual val coroutineScope = CoroutineScope(Dispatchers.Main + SupervisorJob())

   actual fun dispose() {
       coroutineScope.cancel()
       onCleared()
   }

   protected actual open fun onCleared() {
   }
}
{% endhighlight %}

Now that we have these abstractions, we can easily use them to create shared ViewModels. When they will be used on the Android side, they will be handled as ViewModels and for ios we’ll use coroutines to dispatch jobs.

You may see a warning in CoroutineViewModel: `Expected class 'CoroutineViewModel' has no actual declaration in module kmpizza.shared.jvmMain for JVM`. This is there because we specified `jvm()` as target in the shared `build.gradle.kts`. However, for now we only need `CoroutineViewModel` for iOS and Android. So just choose the suggested action and Android Studio will create a template actual CoroutineViewModel for you in `jvmMain`. This will make sure the warning is gone.

<img src="{{site.baseurl}}/assets/images/step-12/3.png" alt="create KtorApi interface" width="550"/>

Now let’s add a `RecipeViewModel` to a `viewmodel` package in the `shared` module:

<img src="{{site.baseurl}}/assets/images/step-12/4.png" alt="create KtorApi interface" width="350"/>

Let’s take a look at our `RecipeViewModel`:
{% highlight kotlin %}
class RecipeViewModel (private val recipeRemoteSource: RecipeRemoteSource)
 : CoroutineViewModel() [1] {

   private val _recipes = MutableStateFlow<List<RecipeResponse>>(emptyList())
   val recipes: StateFlow<List<RecipeResponse>> = _recipes [2]

   init {
       getRecipes()
   }

   fun getRecipes() {
       coroutineScope.launch {
           _recipes.value = recipeRemoteSource.getRecipes() [3]
       }
   }
}
{% endhighlight %}

[1] It extends `CoroutineViewModel` <br>
[2] We’ll need a  variable to hold the mutable state flow of a list fo recipes. We’ll use it to display a list of recipes with Jetpack Compose in Android and SwiftUi in iOS apps. <br>
[3] Here we simply launch our `getRecipes` function from the remote source which fetches data from our backend. And we’ll do it right when we init the `RecipeViewModel`. <br>

To make this viewModel complete and compatible with ios we’ll need to add a function which will help us <b>observe the recipes from the iOS side</b>. We’ll use this function later in SwiftUi and pass an appropriate callback, which refresh the UI accordingly every time when the recipes variable is changed. 

{% highlight kotlin %} 
fun observeRecipes(onChange: (List<RecipeResponse>) -> Unit) {
   recipes.onEach {
       onChange(it)
   }.launchIn(coroutineScope)
}
{% endhighlight %}

Notice how we’re using the coroutineScope to observe the flow of recipes, just like we used it here in `getRecipes` to fetch them.

<i>Awesome, we finished our first shared ViewModel!</i>

 Next we’ll move to Android, <b>start building our UI</b> with Jetpack Compose and see how to <b>bind it to the shared RecipeViewModel</b>.