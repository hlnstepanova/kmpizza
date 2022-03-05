---
title:  "Step 7: Add serialization and useful endpoints to save Recipes in the database" 
layout: post
categories: database exposed postgres
--- 

Last time we tested the local data source with a simple request.<br>
Now let’s add some real power to it.<br>
First, declare a bunch of useful functions in the `LocalSource` interface. These have to do with our future app’s functionality like adding recipes and corresponding ingredients and instructions.

{% highlight kotlin %}
internal interface LocalSource {
   suspend fun getPizza(): String
   suspend fun addIngredient(recipeId: Long, ingredient: Ingredient): Long
   suspend fun addInstruction(recipeId: Long, instruction: Instruction): Long
   suspend fun addRecipe(recipe: Recipe): Long
   suspend fun getRecipes() : List<Recipe>
   suspend fun getRecipe(recipeId: Long) : Recipe
}
{% endhighlight %}

Some of them return `Long`, because we’ll add new entries to the database, which will assign an `id` to each of them and return that `id` as `Long`.

Now implement these functions in `LocalSourceImpl`:
{% highlight kotlin %}
override suspend fun addIngredient(recipeId: Long, ingredient: Ingredient) = withContext(dispatcher) {
   transaction {
       val recipe = RecipeEntity[recipeId.toInt()] [1]
       IngredientEntity.new { [2]
           name = ingredient.name
           amount = ingredient.amount.toBigDecimal()
           metric = ingredient.metric
           this.recipe = recipe
       }.id.value.toLong()
   }
}

override suspend fun addInstruction(recipeId: Long, instruction: Instruction) = withContext(dispatcher) {
   transaction {
       val recipe = RecipeEntity[recipeId.toInt()]
       InstructionEntity.new {
           order = instruction.order
           description = instruction.description
           this.recipe = recipe
       }.id.value.toLong()
   }
}

override suspend fun addRecipe(recipe: Recipe) = withContext(dispatcher) {
   withContext(dispatcher) {
       val recipeId = transaction {
           RecipeEntity.new {
               title = recipe.title
           }.id.value.toLong()
       } [3]

       recipe.ingredients.forEach{
           addIngredient(recipeId, it)
       } [4]

       recipe.instructions.forEach{
           addInstruction(recipeId, it)
       }
       recipeId
   }
}

override suspend fun getRecipes() : List<Recipe> = withContext(dispatcher) {
   transaction {
       RecipeEntity.all().map { it.toRecipe() }
   }
}

override suspend fun getRecipe(recipeId: Long): RecipeResponse = withContext(dispatcher) {
   transaction {
       RecipeEntity[recipeId.toInt()].toRecipeResponse()
   }
}
{% endhighlight %}

These are basic functions that will fill up our database with recipes, ingredients and instructions.

[1] When creating an ingredient or instruction, we have to specify recipeId which this item belongs to and get the appropriate recipe <br>
[2] Then we’ll create a new ingredient or instruction entity and attach it to our recipe<br>
[3] When adding a new recipe to the database we first create a new recipe entry <br>
[4] Then add all the instructions and ingredients to it<br>
 
Now let’s bind those functions to our backend. Just like we did with getPizza, we’ll add some more routes for adding and getting recipes in `Api.kt`:

{% highlight kotlin %}
internal fun Routing.api(localSource: LocalSource) {
   pizza(localSource)
   addRecipe(localSource)
   getRecipes(localSource)
   getRecipe(localSource)
}
{% endhighlight %}

Implement them in the same file:
{% highlight kotlin %}
private fun Routing.addRecipe(localSource: LocalSource) {
    post("/recipes") {
        val recipeId = localSource.addRecipe(call.receive())
        call.respond(recipeId)
    }
}

private fun Routing.getRecipes(localSource: LocalSource) {
    get("/recipes") {
        localSource.getRecipes().let {
            call.respond(it)
        }
    }
}

private fun Routing.getRecipe(localSource: LocalSource) {
    get("/recipes/{recipeId}") {
        val recipeId = call.parameters["recipeId"]!!.toLong()
        localSource.getRecipe(recipeId).let {
            call.respond(it)
        }
    }
}
{% endhighlight %}

[1] When adding a Recipe we specify the Recipe data class as the body input and get a newly created recipe entity id from the database <br>
[2] To get a recipe by id we need to lookup for the recipe with the id, that we specified in API route<br>

One last thing before we test it: we want to be able to serialize our classes between the client and the server. To do so, we’ll need `kotlinx.serialization` plugin. Let’s adjust our dependencies.

First, add this to the `Versions.kt`:
{% highlight kotlin %}
const val KOTLIN_VERSION = "1.6.10"
{% endhighlight %}

In the `backend` module add the serialization plugin to `build.gradle.kts`:
{% highlight kotlin %}
plugins {
   kotlin("jvm")
   application
   id("com.github.johnrengelman.shadow") version Versions.SHADOW_JAR_VERSION
   kotlin("plugin.serialization") version Versions.KOTLIN_VERSION
}
{% endhighlight %}

You may also need to adjust the project `build.gradle.kts` so that it matches the latest Kotlin version you specified, for example like so:
{% highlight kotlin %}
dependencies {
        classpath("org.jetbrains.kotlin:kotlin-gradle-plugin:${Versions.KOTLIN_VERSION}")
        classpath("com.android.tools.build:gradle:7.1.0-alpha10")
    }
{% endhighlight %}

Sync the project.
Then add `@Serializable` annotation to all our model data classes like this:
{% highlight kotlin %}
@Serializable
data class Recipe(
   val id: Long = 0,
   val title: String,
   val ingredients: List<Ingredient>,
   val instructions: List<Instruction>

)
{% endhighlight %}

You may need to do a manual import:
{% highlight kotlin %}
import kotlinx.serialization.Serializable
{% endhighlight %}

Now let’s test this in action. Use Postman for it. Rebuild and make the backend again and run it locally as before:
```
export JDBC_DATABASE_URL=jdbc:postgresql:recipecollection?user=postgres
java -jar ./backend/build/libs/backend.jar -port=9000 
```

Then try adding a Pizza Dough recipe with Postman. Send a `POST` request to our API endpoint `http://localhost:9000/recipes`. You can use the following body:
```json
{
   "title":"Pizza dough",
   "ingredients": [
       {
           "name": "flour",
           "amount": 2.0,
           "metric": "cup"
 
       },
       {
           "name": "water",
           "amount": 1.0,
           "metric": "cup"
 
       },
       {
           "name": "flour",
           "amount": 2.5,
           "metric": "cup"
 
       },
       {
           "name": "salt",
           "amount": 1.0,
           "metric": "teaspoon"
 
       },
       {
           "name": "sugar",
           "amount": 1.0,
           "metric": "teaspoon"
 
       },
       {
           "name": "dry yeast",
           "amount": 7.0,
           "metric": "gram"
 
       },
       {
           "name": "olive oil",
           "amount": 3.0,
           "metric": "tablespoons"
 
       }
 
   ],
   "instructions":[
       {
           "order": 1,
           "description": "Mix water, sugar and yeast. Let it froth for 10 minutes"
       },
       {
           "order": 2,
           "description": "Mix dough, salt, olive oil. Add water mixture from previous step."
 
       },
       {
           "order": 3,
           "description": "Knead the dough for 10 minutes"
 
       }
 
   ]
}
```

We get `500 Internal Server Error` 😱
Don’t be upset! We’ll find out why.

To help us investigate the cause we can install `StatusPages` in `Application.module()` and let it return the cause of the error like this :
{% highlight kotlin %}
install(StatusPages) {
   exception<Throwable> { cause ->
       call.respond(cause.toString())
   }
}
{% endhighlight %}

Now if you rebuild the backend and try the same request again you’ll see 
```
io.ktor.features.CannotTransformContentToTypeException: Cannot transform this request's content to com.example.recipecollection.backend.model.Recipe
```

This happens because we forgot to install `ContentNegotiation` between the client and the server to help us serialize objects. Let’s do it now, add this to you application:
{% highlight kotlin %}
install(ContentNegotiation) {
   json()
}
{% endhighlight %}

Build the backend and try sending the request again.<br>
Voilà!<br>
We get the recipe `id` in return.

Now try posting the `Recipe` again.
Then try the `GET /recipes`endpoint and you’ll get our Pizza Dough recipe in return.<br>
Time to start cooking! 🧑‍🍳





