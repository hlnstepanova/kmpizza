---
title: "Step 14: Glue everything together with Koin dependency injection" 
layout: post
categories: shared koin dependency injection android
--- 

This time we’re going to look into dependency injection with <b>Koin</b>. <br>
In a simple app you can get away with manually injecting every dependency you need.<br>
However, the bigger your app becomes, the more reasonable it is to use a dedicated dependency injection framework.<br>

First, add core dependency to the `Versions.kt`:

{% highlight kotlin %} 
object Common { 
. . .
const val KOIN_CORE = "io.insert-koin:koin-core:$KOIN_VERSION"
}
{% endhighlight %} 

And implement it in the shared `build.gradle.kts` in `commonMain` with:

{% highlight kotlin %} 
val commonMain by getting {
   dependencies {
       . . .
       api(Versions.Common.KOIN_CORE)
   }
}
{% endhighlight %} 

Sync your project files.

Now instead of using `RecipeRemoteSource` directly to fetch data let’s create a <b>repository</b> for our recipes.
In the future, it will manage data between our <b>remote source</b>, which we’ve already implemented and the <b>local source</b>, which we’ll implement later with SQLDelight. 


Add a `repository` package to your `commonMain`.<br>
Create a `RecipeRepository.kt` there. 

<img src="{{site.baseurl}}/assets/images/step-14/1.png" alt="create repository" width="250"/>

Now that we have Koin, we can create a recipe repository by making it a `KoinComponent` and injecting a `recipeRemoteSource`:

{% highlight kotlin %} 
class RecipeRepository : KoinComponent {
   private val recipeRemoteSource: RecipeRemoteSource by inject()

   suspend fun postRecipe(recipe: Recipe): Long = recipeRemoteSource.postRecipe(recipe)
   suspend fun getRecipes(): List<RecipeResponse> = recipeRemoteSource.getRecipes()
}
{% endhighlight %} 

This is a very simple repository, because we don’t have any other source but the backend now. 

Likewise, you can replace `RecipeRemoteSource` with `RecipeRepository` in your `RecipeViewModel` by simply injecting it:
{% highlight kotlin %} 
class RecipeViewModel : CoroutineViewModel(), KoinComponent {
   private val recipeRepository: RecipeRepository by inject()
. . .
}
{% endhighlight %} 

Don’t forget to change `getRecipes()`:
{% highlight kotlin %} 
fun getRecipes() {
   coroutineScope.launch {
       _recipes.value = recipeRepository.getRecipes()
   }
}
{% endhighlight %} 


Now you can remove all the manual injection of `RecipeRemoteSource` in `MainActivity`, `RecipesScreen` and `MainScreen`.

Use `RecipeViewModel` in `RecipesScreen` just like this:

{% highlight kotlin %} 
public fun RecipesScreen() {
   val viewModel = remember {
       RecipeViewModel()
   }
   val recipes by viewModel.recipes.collectAsState()

   Recipes(items = recipes)
}
{% endhighlight %} 

Your `MainActivity` will look much prettier now with no manual injection:

{% highlight kotlin %} 
class MainActivity : ComponentActivity() {
   override fun onCreate(savedInstanceState: Bundle?) {
       super.onCreate(savedInstanceState)
       setContent {
           MaterialTheme {
               MainScreen()
           }
       }
   }
}
{% endhighlight %} 


Try running the app now and you’ll get an error message:<br> `java.lang.IllegalStateException: KoinApplication has not been started`

That is because we have not applied our universal Koin glue yet!<br>
We used dependency injection everywhere, but our app doesn’t know about it.<br>
Therefore we first need to glue the shared and android platform code together.<br>

First create a `di` folder in the shared `commonMain` and add a `CommonModule.kt` there with the following code:

{% highlight kotlin %} 
fun initKoin(appDeclaration: KoinAppDeclaration) [1] = startKoin {
   modules( [2]
       apiModule
   )
}

private val apiModule = module {
   single<KtorApi> { KtorApiImpl() } [3]
   factory { RecipesApi(get()) }
}
{% endhighlight %} 

[1] This will initialize Koin on the platform side. We’ll see how to use it later, when writing the platform-specific code.<br>
[2] Here we’ll keep all the modules that we need for dependency injection. For now it’s just our networking.<br>
[3] We create a singleton of KtorApi and then use a factory to create RecipesApi whenever we need one.<br>

Similarly, we’ll add a `viewModel` module and a `repositoryModule`.<br>
The final version has to look like this:
{% highlight kotlin %} 
fun initKoin(appDeclaration: KoinAppDeclaration) = startKoin {
   appDeclaration()
   modules(
       apiModule,
       repositoryModule,
       viewModelModule
   )
}


private val apiModule = module {
   single<KtorApi> { KtorApiImpl() }
   factory { RecipesApi(get()) }
}

private val viewModelModule = module {
   single{ RecipeViewModel() }
}

private val repositoryModule = module {
   factory { RecipeRemoteSource(get()) }
   single { RecipeRepository() }
}
{% endhighlight %} 

Now we’re only missing platform-related Koin dependencies.<br>
Add this to your `Versions.kt`:
{% highlight kotlin %} 
object Android {
const val KOIN_ANDROID_MAIN = "io.insert-koin:koin-android:$KOIN_VERSION"
. . .
}
{% endhighlight %} 

In Android app `build.gradle`:
{% highlight kotlin %} 
implementation(Versions.Android.KOIN_ANDROID_MAIN)
{% endhighlight %} 

In the `androidApp` create a `MainApp` class which will extend the `Application` class 
<img src="{{site.baseurl}}/assets/images/step-14/2.png" alt="create MainApp" width="290"/>

Start the Koin application in `MainApp.kt`:

{% highlight kotlin %} 
@Suppress("unused")
class MainApp : Application() {

   override fun onCreate() {
       super.onCreate()
       initKoin { 
       }
   }
}
{% endhighlight %} 

Finally, we need to inform the android app about our starting point, which we do in the `Manifest` by adding this line to `application`:
{% highlight xml %} 
   android:name=".MainApp"
{% endhighlight %} 

Run the app again and you’ll see “Pizza dough” on the main screen. <br>
With the help of Koin our project looks neater and it’s still working just like before!

Now that our android app is up and running, in the <b>next step</b> we’ll do the same for the iOS app by <b>binding iOS UI to the shared KMM ViewModel</b>.