---
title: "Step 17: Add Navigation to Jetpack Compose UI" 
layout: post
categories: jetpack compose navigation android
--- 

Now that we have both `RecipesScreen` and `RecipeDetailScreen`, we can set up <b>navigation</b>. 

Clicking on one of the list items, the user will navigate from main screen to the details screen of the recipe with the chosen id.

First, we need a couple of routes that we’ve already discussed when designing our app.

Add `utils` package to `ui` package and create a `Navigation.kt` routing file with:

{% highlight kotlin %} 
sealed class Navigation(val route: String) {

   object Recipes : Navigation("recipes")
   object RecipeDetails : Navigation("recipeDetails")
}
{% endhighlight %} 


Now go to the `MainScreen` and add a `NavHost` there, replacing the `RecipesScreen`:
{% highlight kotlin %} 
@Composable
public fun MainScreen() {
   val navController = rememberNavController()
   val navBackStackEntry by navController.currentBackStackEntryAsState()

   NavHost(
       navController = navController,
       startDestination = Navigation.Recipes.route, [1]
       builder = {
           composable(Navigation.Recipes.route) {
               RecipesScreen(onRecipeClicked = { [2] navController.navigate("recipeDetail/${it.id}") })
           }
           composable(
               "recipeDetail/{id}",
               arguments = listOf(navArgument("id") { type = NavType.LongType })
           ) {
               RecipeDetailsScreen( [3]
                   recipeId = it.arguments!!.getLong("id"),
                   upPress = { navController.popBackStack() })
           }
       })
}
{% endhighlight %} 

[1] The `startDestination` will be the `RecipesScreen`<br>
[2] We define two destinations in this piece. One is `RecipesScreen`, which has an `onRecipeClicked` callback function. It is unresolved now, but it will define the behaviour when one of the items in our `RecipeList` gets clicked: the `navController` will take us to the `RecipeDetailsScreen` with the recipe’s id.<br>
[3] Another destination, `RecipeDetailsScreen` will receive the recipe id as an argument<br>

Also adjust the `RecipeScreen`, which now has a parameter `onRecipeClicked` and will pass it down to the responsible component:

{% highlight kotlin %} 
@Composable
public fun RecipesScreen(onRecipeClicked: (Recipe) -> Unit) {
  . . .
   Recipes (items = recipes, onRecipeClicked = onRecipeClicked)
}

@Composable
fun Recipes(
   items: List<Recipe>,
   onRecipeClicked: (Recipe) -> Unit
) {
   LazyColumn {
       itemsIndexed(items = items,
           itemContent = { _, item ->
               RecipeListItem(item, onRecipeClicked = onRecipeClicked)
           })

   }
}

@Composable
fun RecipeListItem(
recipe: RecipeResponse,
onRecipeClicked: (RecipeResponse) -> Unit){
. . .
   Row(
       verticalAlignment = Alignment.CenterVertically,
       modifier = Modifier
           .height(128.dp)
           .clickable { onRecipeClicked(recipe) }
   )
   . . .
}
{% endhighlight %} 

If needed, also adjust the `Preview` composable:

{% highlight kotlin %} 
@Preview
@Composable
fun RecipeListItemPreview(
) {
. . .
   RecipeListItem(recipe) {}
}
{% endhighlight %} 


Build and run the app. Click on a recipe list item. <br>
You’re in the `RecipeDetailScreen` now.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-17/1.png" alt="open ContentView" width="300"/></div>

Now let’s go back to the recipe screen.<br>
Wait, what? <br>
We can’t go back - we don’t have a top bar yet!<br>
<b>In the next step</b> we’ll learn how to set up <b>a top bar</b> with Jetpack Compose.
