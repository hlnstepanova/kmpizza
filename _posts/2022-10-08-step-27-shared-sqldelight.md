---
title: "Step 27: Add a local database to KMM Android and iOS app with SQLDelight"
layout: post
categories: kmm ios shared android sqldelight
--- 

Now that we basically have a functional app, we can adapt it to network emergencies - what if you’re baking pizza in a remote mountain hut and there’s no connection? <br>

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-27/pizza_view.png" alt="pizza" width="400"/></div>

To make the app more persistent, we’ll equip it with a local database. For this purpose we’ll use <b>SQLDelight</b> library, which is based on SQLLite.
 
The following setup follows the <a href="https://kotlinlang.org/docs/kmm-configure-sqldelight-for-data-storage.html#gradle-plugin">official documentation</a>:
 
<i>“SQLDelight generates type-safe Kotlin APIs from SQL statements. It also provides a multiplatform implementation of the SQLite driver.”</i>
 
Define the latest stable version of the plugin in `Versions.kt`:

{% highlight kotlin %} 
const val SQL_DELIGHT = "1.5.3"
{% endhighlight %} 
 
In `object Common` add the dependency:
{% highlight kotlin %} 
const val SQLDELIGHT_PLUGIN = "com.squareup.sqldelight:gradle-plugin:$SQL_DELIGHT"
{% endhighlight %} 
 
Then use it in <b>project level</b> `build.gradle.kts`: 
{% highlight kotlin %} 
   classpath(Versions.Common.SQLDELIGHT_PLUGIN)
{% endhighlight %} 
 
Then add this plugin to other `plugins` in <b>shared</b> `build.gradle.kts`:
{% highlight kotlin %} 
plugins {
   kotlin("multiplatform")
   kotlin("native.cocoapods")
   id("com.android.library")
   kotlin("plugin.serialization") version Versions.KOTLIN_VERSION
   id("com.squareup.sqldelight")
}
{% endhighlight %} 
 
Now we need to setup database drivers, therefore we’ll add the following dependencies to `Versions.kt`:
 
In `object Common`: 
{% highlight kotlin %} 
const val SQLDELIGHT_DRIVER = "com.squareup.sqldelight:runtime:$SQL_DELIGHT"
{% endhighlight %} 
 
In `object Android`: 
{% highlight kotlin %} 
const val SQLDELIGHT_DRIVER = "com.squareup.sqldelight:android-driver:$SQL_DELIGHT"
{% endhighlight %} 
 
In `object iOS`: 
{% highlight kotlin %} 
const val SQLDELIGHT_DRIVER = "com.squareup.sqldelight:native-driver:$SQL_DELIGHT"
{% endhighlight %} 
 
Go to <b>shared</b> `build.gradle.kts`.
Add the dependencies in appropriate `sourceSets`:
 
<b>commonMain</b>:
{% highlight kotlin %} 
implementation(Versions.Common.SQLDELIGHT_DRIVER)
{% endhighlight %} 
 
<b>androidMain</b>:
{% highlight kotlin %} 
implementation(Versions.Android.SQLDELIGHT_DRIVER)
{% endhighlight %} 
 
<b>iosMain</b>:
{% highlight kotlin %} 
implementation(Versions.iOS.SQLDELIGHT_DRIVER)
{% endhighlight %} 
 
Sync your project.
 
Now we need to configure our database. We will create a database named `PizzaDatabase` with the package name `dev.tutorial.kmpizza.db` for the generated Kotlin classes. At the bottom of <b>shared</b> `build.gradle.kts` add:

{% highlight kotlin %} 
sqldelight {
   database("KmpizzaDatabase") { // This will be the name of the generated database class.
       packageName = "dev.tutorial.kmpizza.db"
       sourceFolders = listOf("sqldelight")
    }
}
{% endhighlight %} 
 
<i>“SQLDelight provides multiple platform-specific implementations of the SQLite driver, so you should create it for each platform separately.”</i> <br>
So let’s use a suitable implementation for our platforms.<br>
Go to <b>shared</b> `CommonModule.kt` and add a `platformModule`:

{% highlight kotlin %} 
modules(
   apiModule,
   repositoryModule,
   viewModelModule,
   platformModule
)
{% endhighlight %} 

Here we’ll specify the SqlDelight Driver.
In <b>shared</b> <b>di</b> package create a new file `PlatformModule.kt` and add the following:

{% highlight kotlin %} 
expect val platformModule: Module
{% endhighlight %} 
 
Now add `PlatformModule.kt` to <b>androidMain/dev/tutorial/kmpizza/di</b> and `PlatformModule.kt` to <b>iosMain/dev/tutorial/kmpizza/di</b>, and write actual platform modules.

<b>androidMain</b>:

{% highlight kotlin %} 
actual val platformModule = module {
   single<SqlDriver> {
       AndroidSqliteDriver(
           KmpizzaDatabase.Schema,
           get(),
           "kmpizza.db"
       )
   }
}
{% endhighlight %} 

<b>iosMain</b>:

{% highlight kotlin %} 
actual val platformModule = module {
   single<SqlDriver> { NativeSqliteDriver(KmpizzaDatabase.Schema, "kmpizza.db") }
}
{% endhighlight %} 

Also add linking to `sqlite` in the ios project:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-27/5.png" alt="link sqlite ios" width="1000"/></div>
 
Add 
{% highlight kotlin %} 
@Suppress("NO_ACTUAL_FOR_EXPECT")
{% endhighlight %} 

to suppress the warning for JVM module.
 
You may notice that `KmpizzaDatabase` is unresolved. That’s because it doesn’t exist yet - we need to create it first. 
 
Create a package `sqldelight` in <b>shared/src/commonMain</b>.<br>
There, create a package that corresponds to the package you specified earlier: <b>dev/tutorial/kmpizza/db</b><br>
In that package, create `KmpizzaDatabase.sq` file. <br>
Here you’ll write down all your SQL queries.<br>
In the end you’ll have the following structure:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-27/1.png" alt="db structure" width="300"/></div>
 
Now let’s create our tables for recipes.<br>
In `KmpizzaDatabase.sq` write the following:

{% highlight kotlin %} 
import com.example.recipecollection.model.Ingredient;
import com.example.recipecollection.model.Instruction;
import com.example.recipecollection.model.RecipeImage;
import kotlin.collections.List;
 
CREATE TABLE Recipe (
id INTEGER NOT NULL PRIMARY KEY AUTOINCREMENT,
title TEXT NOT NULL,
ingredients TEXT AS List<Ingredient> NOT NULL,
instructions TEXT AS List<Instruction> NOT NULL,
images TEXT AS List<RecipeImage> NOT NULL
);
{% endhighlight %} 
 
You’ll see the IDE suggesting you to <b>install the SQLDelight plugin</b> - it’s a good idea.<br>
The plugin will help you read SQL statements.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-27/2.png" alt="sql ide plugin" width="600"/></div>
 
You still may see `List` as unresolved, but it’s just an IDE bug and will not affect the build process.
 
Sync and rebuild the app. Now you can import `KmpizzaDatabase` where it was previously unresolved.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-27/3.png" alt="import kmpizza database" width="600"/></div>
 
For simplicity of the tutorial, we won’t create separate tables for `Ingredients` and `Instructions`. Instead, we’ll use custom `types` and `columnAdapters` to convert objects to text. In `commonMain` create a new folder `local` and add `RecipesLocalSource.kt` there. Don’t write anything into the class yet, just add these adapters to repositories on top and add the necessary imports from your project and `kotlinx.serialization`:

{% highlight kotlin %} 
val listOfInstructionsAdapter = object : ColumnAdapter<List<Instruction>, String> {
   override fun decode(databaseValue: String) = if (databaseValue.isEmpty()) emptyList() else databaseValue.split("|").map { Json.decodeFromString<Instruction>(it) }
   override fun encode(value: List<Instruction>) = if (value.isEmpty()) "" else value.joinToString(separator = "|") { Json.encodeToString(it) }
}
 
val listOfIngredientsAdapter = object : ColumnAdapter<List<Ingredient>, String> {
   override fun decode(databaseValue: String) = if (databaseValue.isEmpty()) emptyList() else databaseValue.split("|").map { Json.decodeFromString<Ingredient>(it) }
   override fun encode(value: List<Ingredient>) = if (value.isEmpty()) "" else value.joinToString(separator = "|") { Json.encodeToString(it) }
}
 
val listOfRecipeImagesAdapter = object : ColumnAdapter<List<RecipeImage>, String> {
   override fun decode(databaseValue: String) = if (databaseValue.isEmpty()) emptyList() else databaseValue.split("|").map { Json.decodeFromString<RecipeImage>(it) }
   override fun encode(value: List<RecipeImage>) = if (value.isEmpty()) "" else value.joinToString(separator = "|") { Json.encodeToString(it) }
}
 
class RecipeLocalSource {
}
{% endhighlight %} 
 
Now we can rebuild the project and you’ll find generated sqldelight files.<br>
To use them, we’ll return to our `CommonModule.kt` and add a new module:

{% highlight kotlin %} 
private val coreModule = module {
   single {
       KmpizzaDatabase(
           get(),
           Recipe.Adapter(
               instructionsAdapter = listOfInstructionsAdapter,
               ingredientsAdapter = listOfIngredientsAdapter,
               imagesAdapter = listOfRecipeImagesAdapter
           )
       )
   }
}
{% endhighlight %} 
 
Of course, we’ll add it to modules too:

{% highlight kotlin %} 
modules(
   apiModule,
   repositoryModule,
   viewModelModule,
   platformModule,
   coreModule
)
{% endhighlight %} 

Remember about the `RecipeLocalSource`. Use the recipe `local` database there as parameter:

{% highlight kotlin %} 
class RecipeLocalSource (private val dbRef: KmpizzaDatabase) {
}
{% endhighlight %} 
 
And then in `CommonModule.kt` add the following to the top of `repositoryModule`:

{% highlight kotlin %} 
factory { RecipeLocalSource(get()) }
{% endhighlight %} 
 
Now it’s time to write some queries and see how we can use them in `RecipeLocalSource`.<br>
 Go back to `RecipeDatabase.sq` and add following SQL queries:
{% highlight kotlin %} 
getAllRecipes:
SELECT * FROM Recipe;
 
getRecipeById:
SELECT * FROM Recipe WHERE id = ?;
 
insertOrReplaceRecipe:
INSERT OR REPLACE INTO Recipe (
                            id,
                            title,
                            ingredients,
                            instructions,
                            images)
                            VALUES(?, ?, ?, ?, ?);
{% endhighlight %} 

Now let’s use them to retrieve data.<br>
In `RecipesLocalSource` add following functions:

{% highlight kotlin %} 
fun Recipe.mapToRecipeUiModel(): RecipeUiModel{ [1]
   return RecipeUiModel(
       id, title, ingredients, instructions, images
   )
}
 
fun getAllRecipes() : List<RecipeUiModel> = 
       dbRef.recipeDatabaseQueries [2]
           .getAllRecipes()
           .executeAsList()
           .map { it.mapToRecipeUiModel() }
 
 
fun getRecipeById(id: Long) : RecipeUiModel? =
   dbRef.recipeDatabaseQueries
           .getRecipeById(id)
           .executeAsOneOrNull()?.mapToRecipeUiModel()
 
 
fun insertOrReplaceRecipe(recipe: RecipeUiModel) {
   dbRef.recipeDatabaseQueries
       .insertOrReplaceRecipe(recipe.id, recipe.title, recipe.ingredients, recipe.instructions, recipe.images)
}
{% endhighlight %} 
 
<b>[1]</b> This mapper function that will help you convert database Recipe to RecipeUiModel<br>
<b>[2]</b> Use the generated query to get all recipes from the local database and convert them to RecipeUiModel <br>
<b>[3]</b>, <b>[4]</b> Similarly we can extract a recipe by id from the local database or insert a new recipe
 
 
Now we need to adjust the repository. The idea is to use remote source and back up with local source data. <br>
Go to `RecipeRepository` and add `recipeLocalSource` below `private val recipeRemoteSource`

{% highlight kotlin %} 
private val recipeLocalSource: RecipeLocalSource by inject()
{% endhighlight %} 

Then adjust all the methods to use `recipeLocalSource`:
{% highlight kotlin %} 
suspend fun postRecipe(recipe: RecipeUiModel): Long? {
   return try {
       val id = recipeRemoteSource.postRecipe(recipe.toRecipeRequest()) [1]
       id?.let {
           val entry = recipe.copy(id = it)
           recipeLocalSource.insertOrReplaceRecipe(entry) [2]
       }
       id
   } catch (e: Exception) {
       null
   }
}
{% endhighlight %} 
 
<b>[1]</b> Here we try to save our new recipe remotely<br>
<b>[2]</b> If we succeed, we should receive a recipe `id` from the api, then we can save the recipe locally<br>
 
{% highlight kotlin %} 
suspend fun getRecipes(): List<RecipeUiModel> {
   return try {
       val recipes = recipeRemoteSource.getRecipes().map {
           it.toRecipeUiModel()
       } [1]
       recipes.forEach {
           recipeLocalSource.insertOrReplaceRecipe(it)
       } [2]
       recipes
   } catch (e: Exception) {
       recipeLocalSource.getAllRecipes() [3]
   }
}
{% endhighlight %} 
 
<b>[1]</b> Here we try to get all recipes from backend<br>
<b>[2]</b> If we succeed, we want to save all the recipes that we received locally<br>
<b>[3]</b> Otherwise we’ll just get what we have saved locally<br>
 
{% highlight kotlin %} 
suspend fun getRecipe(id: Long): RecipeUiModel? {
   return try {
       recipeRemoteSource.getRecipe(id).toRecipeUiModel()[1]
   } catch (e: Exception) {
       recipeLocalSource.getRecipeById(id)[2]
   }
}
{% endhighlight %} 

<b>[1]</b> Try getting the recipe from remote source<br>
<b>[2]</b> If it doesn’t work, then get the local backup<br>
 
{% highlight kotlin %} 
suspend fun postRecipeImage(recipeId: Long, imageFile: ImageFile) = recipeRemoteSource.postImage(recipeId, imageFile) [1]
{% endhighlight %} 
 
<b>[1]</b> This request remains unchanged<br>
 
Build and run the app. You’ll get an error:<br>
`Caused by: org.koin.core.error.NoBeanDefFoundException: |- No definition found for class:'android.content.Context'. Check your definitions!`
 
This is because we forgot to add the `androidContext` when starting the app. Go to `MainApp.kt` and change `initKoin {}` to this:
{% highlight kotlin %} 
initKoin {
   androidContext(this@MainApp)
}
{% endhighlight %} 
 
Now <b>turn off your internet connection</b> and run the Android app again. <b>You will still see that recipe</b> in the recipe list. 
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-27/4.gif" alt="android no connection" width="300"/></div>

However, the app has just the basic database functionality now. You won’t be able to create a new recipe offline. If you’re curious, try adjusting `RecipeRepository` so that allows you more operations when offline.
 
Now build and run your iOS app. It behaves similarly. 
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-27/6.gif" alt="ios no connection" width="300"/></div>
 
See, we <b>didn’t have to change anything in iOS at all</b> - it all came with the changes in the shared code. <br>
Isn’t it awesome?

 
<i><b>Here’s a task for you</b>: if you try opening one of the locally saved recipes without internet connection, you’ll notice that there is no image. Fix it by adding a “No Internet” placeholder instead of an empty space when the user can’t download the image.</i>