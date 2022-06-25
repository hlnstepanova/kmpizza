---
title: "Step 16: Use Coil for Images in Jetpack Compose UI" 
layout: post
categories: jetpck compose coil images android
--- 

Let’s decorate the recipe list with images!

We haven’t implemented any logic to get images from backend and handle them in our shared business logic, so we’ll use placeholders at first.

To use images you need the <b>Coil</b> image library. Add these to `Versions.kt`:
{% highlight kotlin %} 
const val COIL  = "2.0.0-rc03"

object Android {
. . .
const val COMPOSE_COIL = "io.coil-kt:coil-compose:$COIL"
}
{% endhighlight %} 

Add this to the androidApp `build.gradle.kts`:
{% highlight kotlin %} 
implementation (Versions.Android.COMPOSE_COIL)
{% endhighlight %} 

Sync your project files.<br>
Add a `RecipeListItem` composable to `RecipesScreen.kt`.<br>
Now we can use Coil to load an image for the `RecipeListItem`:
{% highlight kotlin %} 
@Composable
fun RecipeListItem(
   recipe: RecipeResponse
) {
   val placeholder = "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"
   Row(
       verticalAlignment = Alignment.CenterVertically,
       modifier = Modifier.height(128.dp)
   ) {
       AsyncImage(
           modifier = Modifier.padding(8.dp),
           model = placeholder,
           contentDescription = null
       )
       Text(
           text = recipe.title)
   }
}
{% endhighlight %} 

If you want to see a preview of the UI Item, it can be easily done with Compose, just add this to the bottom of your file:

{% highlight kotlin %} 
@Preview
@Composable
fun RecipeListItemPreview(
) {
   val title = "Best Dish in the World"
   val recipe = RecipeResponse(id = 0, title = title, listOf(), listOf(), listOf() )
   RecipeListItem(recipe)
}
{% endhighlight %} 


Finally, you can use your recipe list item in the recipe list from before. Change `Recipes` composable:
{% highlight kotlin %} 
@Composable
fun Recipes(
   items: List<Recipe> 
) {
   LazyColumn {
       itemsIndexed(items = items,
           itemContent = { _, item ->
               RecipeListItem(item)
           })

   }
}
{% endhighlight %} 

If you run the app now, you will see placeholder image next to the Pizza dough recipe.

<img src="{{site.baseurl}}/assets/images/step-16/1.png" alt="open ContentView" width="400"/>

We want our app not only to show a list of recipe titles, but also some other information like ingredients and instructions. When the recipe is clicked we want to move to the recipe details screen. 
 
Add a `RecipeDetailsScreen.kt` to `ui` package.

{% highlight kotlin %} 
@Composable
public fun RecipeDetailsScreen(recipeId: Long, upPress: () -> Unit) [1] { 
   val viewModel = remember { RecipeViewModel() } [2]
   val recipes by viewModel.recipes.collectAsState()
   val recipe = recipes.find { it.id == recipeId } [3]
   val placeholder = "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"

   Column(
       modifier = Modifier.padding(8.dp),
       horizontalAlignment = Alignment.CenterHorizontally
   ) {
       HeaderImage(image = placeholder)
       recipe?.let { Title(it.title) }
       SectionHeader(title = "Ingredients")
       recipe?.let { Ingredients(it.ingredients) }
       SectionHeader(title = "Instructions")
       recipe?.let { Instructions(it.instructions) }
   }
}
{% endhighlight %} 

[1] As parameters pass the recipe id and a function that will let us go back to the recipes list.<br>
[2] For now just like in `RecipesScreen` we use the `RecipeViewModel`.<br>
[3] We get all the recipes from `RecipeViewModel` and find the one we want by the id that we’ve passed from `RecipesScreen`. It’s not the perfect solution, but we’ll touch it up later.<br>

<b><i>Check out the project repo for full code.</i></b>
 
Look at the `SectionHeader` composable:
{% highlight kotlin %} 
@Composable
private fun SectionHeader(title: String) {
   Column(
       modifier = Modifier
           .fillMaxWidth()
           .padding(top = 8.dp),
       horizontalAlignment = Alignment.CenterHorizontally
   ) {
       Text(text = title)
       Image(
           painter = rememberAsyncImagePainter(R.drawable.ornament),
           contentDescription = null,
       ) [1]
   }
}
{% endhighlight %} 

[1] Unlike `AsyncImage` that we used before to make an image request and load an external resource, `Image` with `rememberAsyncImagePainter` is used to load a drawable from local resources.

This is how simple it is to use Coil image library!<br>
<b>In the next step</b> we’ll set up <b>navigation</b> to move from recipes list to recipe details screen.