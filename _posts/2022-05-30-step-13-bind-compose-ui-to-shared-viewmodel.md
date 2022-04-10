---
title: "Step 13: Bind Jetpack Compose UI to the shared KMM ViewModel" 
layout: post
categories: shared KMM viemwodel compose android
--- 

We’ve been working on the shared module for a long time.<br> Let’s see if it brought us anything. 

For a moment let’s step back from the shared module and work on our apps. 

If you run the Android app now, you’ll still see the template <i>“Hello, Android…”</i> text. <br>We won’t need it, so you can remove `Greeting` and `Platform` files from your `commonMain`, as well as `androidMain` and `iosMain`. 

We’ll also use Jetpack Compose for our UI, so we’ll rebuild the `MainActivity` and eliminate the `layout` folder.

But first import <b>Jetpack Compose</b> dependencies.<br>
In your `Versions.kt` add:
{% highlight kotlin %} 
const val COMPOSE = "1.1.1"
const val COMPOSE_ACT = "1.6.0-alpha01"
const val COMPOSE_NAV = "2.5.0-alpha03"
{% endhighlight %} 

Then add to your Android object: 

{% highlight kotlin %} 
// Compose
const val COMPOSE_UI = "androidx.compose.ui:ui:$COMPOSE"
const val COMPOSE_GRAPHICS = "androidx.compose.ui:ui-graphics:$COMPOSE"
const val COMPOSE_TOOLING = "androidx.compose.ui:ui-tooling:$COMPOSE"
const val COMPOSE_FOUNDATION = "androidx.compose.foundation:foundation-layout:$COMPOSE"
const val COMPOSE_MATERIAL = "androidx.compose.material:material:$COMPOSE"
const val COMPOSE_NAVIGATION = "androidx.navigation:navigation-compose:$COMPOSE_NAV"
const val COMPOSE_ACTIVITY = "androidx.activity:activity-compose:$COMPOSE_ACT"
{% endhighlight %} 


And implement them in your Android app `build.gradle.kts`:
{% highlight kotlin %} 
 dependencies {
   implementation(project(":shared"))

   implementation(Versions.Android.COMPOSE_UI)
   implementation(Versions.Android.COMPOSE_GRAPHICS)
   implementation(Versions.Android.COMPOSE_TOOLING)
   implementation(Versions.Android.COMPOSE_FOUNDATION)
   implementation(Versions.Android.COMPOSE_MATERIAL)
   implementation(Versions.Android.COMPOSE_NAVIGATION)
   implementation(Versions.Android.COMPOSE_ACTIVITY)
}
{% endhighlight %} 


Also add a material components dependency to your Android app.<br>
As before, add these lines to `Versions.kt`:
{% highlight kotlin %} 
const val MATERIAL = "1.5.0"
const val MATERIAL_COMPONENTS = "com.google.android.material:material:$MATERIAL"
{% endhighlight %} 

And implement it in your Android app `build.gradle.kts`:
{% highlight kotlin %} 
implementation(Versions.Android.MATERIAL_COMPONENTS)
{% endhighlight %} 

Moreover,  add the following configurations to the android section of android app `build.gradle`:
{% highlight kotlin %} 
buildFeatures {
   compose = true
}

composeOptions {
   kotlinCompilerExtensionVersion = "1.1.1"
}
{% endhighlight %} 

These will just turn on the Jetpack Compose functionality.<br> Unluckily, it doesn’t come automatically with the Android Studio Wizard KMM Template, which is weird.<br> I think Jetpack Compose and KMM are meant for each other.

Sync your project files.

Now open `MainActivity`, which is in the `androidApp` module in the `src` folder. 

<img src="{{site.baseurl}}/assets/images/step-13/1.png" alt="open MainActivity" width="290"/>

We don’t need the `greet()` anymore. <br>We’ll also make the MainActivity extend `ComponentActivity` instead of `AppCompatActivity`:

{% highlight kotlin %} 
class MainActivity : ComponentActivity() {

   override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       setContent { [1]
               MaterialTheme { [2]
           }
       }
   }
}
{% endhighlight %} 

<b>[1]</b> Set the content with setContent function. It basically tells Jetpack Compose where to start rendering the UI.<br>
<b>[2]</b> Apply the basic MaterialTheme<br>

Build and run the app. <br>
You’ll see an empty screen. <br>
Good. 

Let’s add our first screen where we’ll display a list of recipes.<br>
Add a `ui` package with a `RecipesScreen.kt` inside. This will contain our first composables. <b>Composable</b> is a UI component in Jetpack Compose that can be recomposed if necessary. Jetpack Compose observes the states that the composable depends on and recomposes automatically.

Add the following to `RecipesScreen.kt`:
{% highlight kotlin %} 
@Composable
public fun RecipesScreen(recipeRemoteSource: RecipeRemoteSource) {
   val viewModel = remember {
       RecipeViewModel(recipeRemoteSource) [1]
   }
   val recipes by viewModel.recipes.collectAsState() [2]

  Recipes (items = recipes)[3]
}

@Composable
fun Recipes(
   items: List<RecipeResponse>
) {
   LazyColumn { [4]
           itemsIndexed(items = items,
               itemContent = { _, item ->
                   Text(text = item.title)
               })
      
   }
}
{% endhighlight %} 

<b>[1]</b> We will use the shared `RecipeViewModel` as a source of data. Here we use the `remember` function so that the viewModel stays the same within recompositions. In other words, it allows you to remember state from previous recompose invocation, because we want to use the same viewModel in between the recompositions. We also need to inject the `RecipeRemoteSource`, which is used for network requests. Later we’ll learn how to simplify the dependency injection with a library.<br>
<b>[2]</b> We’ll collect the recipes flow from our ViewModel as state. This will make Jetpack Compose observe all the changes to the recipes list and recompose the ui accordingly<br>
<b>[3]</b> We’ll use the state of our recipes list from the viewModel in the `Recipes` composable, where we display a list of recipes<br>
<b>[4]</b> `LazyColumn` in Jetpack compose is basically your recyclerView. For starters we’ll show a list of recipe titles. We specify the recipes as `items` and the `content` of each item will be represented by a simple `Text` composable.

Also add a `MainScreen.kt` to the `ui` package.<br> 

<img src="{{site.baseurl}}/assets/images/step-13/2.png" alt="open MainActivity" width="290"/>

We’ll use it later more intensively for navigation and other stuff.<br> For now it'll be our entry point for `RecipesScreen`: 
{% highlight kotlin %} 
@Composable
public fun MainScreen(recipeRemoteSource: RecipeRemoteSource) {
   RecipesScreen(recipeRemoteSource)
}
{% endhighlight %} 

Now add the `MainScreen` to the `MainActivity` as the entry point for your <b>Composition</b>:
{% highlight kotlin %} 
class MainActivity : ComponentActivity() {
   val recipeRemoteSource = RecipeRemoteSource(RecipesApi(KtorApiImpl()))

   override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       setContent {
           MaterialTheme {
               MainScreen(recipeRemoteSource = recipeRemoteSource)
           }
       }
   }
}
{% endhighlight %} 

Finally, add something we always forget to add to the `AndroidManifest`:
{% highlight kotlin %} 
<uses-permission android:name="android.permission.INTERNET" />
{% endhighlight %} 

Build and run the app.<br>
If everything goes well, you’ll see <i>“Pizza dough”</i> on your screen.<br>
Yay! It worked! <br>

<img src="https://jf-staeulalia.pt/img/other/41/collection-images-pizza-22.jpg" alt="Yay" width="300"/>

Feel free to add more recipes to your backend if you want to see a longer list.

Well, to be honest, this architecture <i><b>kind of</b></i> works.<br>
But just look at this chain of injections:
{% highlight kotlin %} 
val recipeRemoteSource = RecipeRemoteSource(RecipesApi(KtorApiImpl()))
{% endhighlight %} 
Isn’t it hideous?<br>
It also implies we’ll have to do something as ugly in the iOS app.<br>
Luckily, we can avoid this and make everything look much better.<br>
<b>In the next step</b> we’ll learn how to do it, using <b>Koin dependency injection framework</b>.