---
title: "Step 20: Connect shared RecipeDetails ViewModel to Android Compose UI" 
layout: post
categories: kmm shared viewmodel ktor koin
--- 

Now let’s create a proper `ViewModel` for the `RecipeDetails` screen. Keep in mind that we’ll also reuse this `ViewModel` to add new recipes afterwards.

First in the `shared` module in the `model` folder add an `RecipeUiModel`, which we’ll use as an interface between <b>shared</b> and <b>UI</b> layer:
{% highlight kotlin %} 
data class RecipeUiModel (
   val id: Long = 0,
   val title: String,
   val ingredients: List<Ingredient> = listOf(),
   val instructions: List<Instruction> = listOf()
)
{% endhighlight %} 


Go to `shared` -> `remote` -> `RecipesRemoteSource.kt` and add `getRecipe()`:

{% highlight kotlin %} 
suspend fun getRecipe(id: Long) = recipesApi.getRecipe(id)
{% endhighlight %} 

And use it in `RecipeRepository`:

{% highlight kotlin %} 
suspend fun getRecipe(id: Long) : RecipeResponse = recipeRemoteSource.getRecipe(id)
{% endhighlight %} 

Next, we will adjust our models so that we can use them for our UI as well.<br>
In the best case scenario you want to create UiModels and use them to display data, but in this tutorial we’ll just make `id` nullable for our shared models (except `RecipeResponse`, because it cannot arrive without id).<br>
Also rename `Recipe` to `RecipeRequest` for clarity:

{% highlight kotlin %} 
@Serializable
data class RecipeRequest (
   val id: Long? = 0,
   val title: String,
   val ingredients: List<Ingredient>,
   val instructions: List<Instruction>
)

@Serializable
data class Ingredient(
   val id: Long? = 0,
   val name: String,
   val amount: Double,
   val metric: String

)
@Serializable
data class Instruction(
   val id: Long? = 0,
   val order: Int,
   var description: String,
)
{% endhighlight %} 

Then create helper functions to transform your `RecipeResponse` to `RecipeUiModel` and back:

{% highlight kotlin %} 
fun RecipeResponse.toRecipeUiModel() = RecipeUiModel(
   id = id,
   title = title,
   ingredients = ingredients,
   instructions = instructions
)

fun RecipeUiModel.toRecipeRequest() = RecipeRequest(
   id = id,
   title = title,
   ingredients = ingredients,
   instructions = instructions
)
{% endhighlight %} 


Now in `shared`->`viewmodel` folder add `RecipeDetailsViewModel.kt`.<br>
Here create an `EditRecipeChangeListener` to help the `ViewModel` observe the changes in the user interface:

{% highlight kotlin %} 
interface EditRecipeChangeListener { [1]
   fun onTitleChanged(title: String)
   fun onIngredientsChanged(ingredient: Ingredient)
   fun onInstructionsChanged(instruction: Instruction)
}
{% endhighlight %} 

And finally add the `RecipeDetailsViewModel` itself:

{% highlight kotlin %} 
class RecipeDetailsViewModel(private val id: Long?) : CoroutineViewModel(), KoinComponent, EditRecipeChangeListener { [2]
   private val recipeRepository: RecipeRepository by inject()

   private val _recipe = MutableStateFlow(EditRecipeUiModel()) [3]
   val recipe: StateFlow<EditRecipeUiModel> = _recipe

   init {
      id?.let { getRecipe(it) } [4]
   }

   fun getRecipe(id: Long) {
   coroutineScope.launch {
       _recipe.value = recipeRepository.getRecipe(id).toRecipeUiModel()[5]
   }
}


   @Suppress("unused")
fun observeRecipe(onChange: (RecipeUiModel?) -> Unit) { [6]
   recipe.onEach {
       onChange(it)
   }.launchIn(coroutineScope)
}


  override fun onTitleChanged(title: String) { [7]
   _recipe.value = _recipe.value?.copy(title = title)

}

override fun onIngredientsChanged(ingredient: Ingredient) {
   val ingredients = _recipe.value?.ingredients
   _recipe.value = _recipe.value?.copy(ingredients = ingredients?.plus(ingredient) ?: listOf(ingredient))

}

override fun onInstructionsChanged(instruction: Instruction) {
   val instructions = _recipe.value?.instructions
   _recipe.value = _recipe.value?.copy(instructions = instructions?.plus(instruction) ?: listOf(instruction))

}
{% endhighlight %} 


<b>[1]</b> We created an interface to react to UI changes<br>
<b>[2]</b> `RecipeDetailsViewModel` implements this interface<br>
<b>[3]</b> The `ViewModel` also holds the mutable state flow of the `recipe`, which we’ll display on the platform side<br>
<b>[4]</b> If we have a `recipeId`, we’ll load it from our backend, otherwise start with an empty new recipe<br>
<b>[5]</b> Transform `RecipeResponse` to `RecipeUiModel`<br>
<b>[6]</b> This method just like in `RecipesViewModel` will be used to observe the `recipe` flow from iOS<br>
<b>[7]</b> These overridden methods belong to `EditRecipeChangeListener` interface. Here you modify the data in the Recipe `StateFlow`, which passes the newest data to the UI<br>

To edit or create a recipe add an appropriate function in `shared` -> `RecipesApi`:
{% highlight kotlin %} 
suspend fun postRecipe(recipe: Recipe): Long? {
   try{
       return client.post {
           json()
           apiUrl(RECIPES_BASE_URL)
           setBody(body)
       }.body()
   } catch (e: Exception){
       return null
   }
}
{% endhighlight %} 


Use it in `RecipesRemoteSource`:
{% highlight kotlin %} 
suspend fun postRecipe(recipe: Recipe) = recipesApi.postRecipe(recipe)
{% endhighlight %} 

Add this to `RecipeRepository`:
{% highlight kotlin %} 
suspend fun postRecipe(recipe: Recipe): Long?  = recipeRemoteSource.postRecipe(recipe)
{% endhighlight %} 

In `RecipeDetailsViewModel`:
{% highlight kotlin %} 
fun saveRecipe() {
   coroutineScope.launch {
       recipe.value?.let {
           if (it.title.isNotEmpty() && it.ingredients.isNotEmpty() && it.instructions.isNotEmpty()){
               val result = recipeRepository.postRecipe(it.toRecipeRequest())[1]
               result?.let { _upload.value = true } [2]
           }
       }
   }
}
{% endhighlight %} 

<b>[1]</b> Transform `RecipeUiModel` to `RecipeRequest`<br>
<b>[2]</b> Use the result value to send a signal back to the UI<br>

Here we also add the `upload` lever to know when the new recipe was successfully uploaded, so that we can go back to the recipe list.<br>
Add the following to the `RecipeDetailsViewModel`:

{% highlight kotlin %} 
private val _upload = MutableStateFlow<Boolean>(false)
val upload: StateFlow<Boolean> = _upload

@Suppress("unused")
fun observeUpload(onChange: (Boolean?) -> Unit) {
   upload.onEach {
       onChange(it)
   }.launchIn(coroutineScope)
}

fun resetUpload(){
   _upload.value = false
}
{% endhighlight %} 


The Recipe Details View Model is ready!<br>
 Now we can add new recipes to the database!<br>
<b>In the next step</b> we’ll <b>connect</b> this <b>ViewModel</b> to our Android <b>ompose UI</b>.
