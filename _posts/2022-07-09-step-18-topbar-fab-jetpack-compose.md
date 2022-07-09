---
title: "Step 18: Add a TopBar and a Floating Action Button to the Jetpack Compose UI" 
layout: post
categories: jetpack compose navigation android topbar fab
--- 

Now we can navigate from one screen to another, but we can’t go back, because there’s no `TopBar` navigation.<br> 
Let’s set it up.

First, create a `TopBar` composable in the <b>utils</b> folder.

<img src="{{site.baseurl}}/assets/images/step-18/1.png" alt="create topbar" width="300"/>

It will receive an `upPress` callback and navigate back when the button is pressed:

{% highlight kotlin %} 
@Composable
fun TopBar(upPress: () -> Unit) {
   Surface(elevation = 8.dp) {
       Row(modifier = Modifier.fillMaxWidth()) {
           Icon(
               painter = painterResource(id = R.drawable.ic_back),
               modifier = Modifier.clickable(onClick = upPress)
                   .padding(16.dp),
               contentDescription = null
           )
       }
   }
}
{% endhighlight %} 

Then use it in the `RecipeDetailsScreen`. Wrap the `Column` composable that we already had there in a `Scaffold` with a `topBar` parameter:

{% highlight kotlin %} 
Scaffold(
   topBar = { TopBar(upPress = upPress) })
{
   Column( . . . ) { . . . }
}
{% endhighlight %} 

Build and run the app.<br>
Now we can navigate back from the recipe details to the main list.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-18/2.png" alt="create topbar" width="300"/></div>

Next we’ll add a floating button to our `RecipesScreen`, so that we can create a new recipe. 

To do so, first set up the navigation. <br>
We’ll reuse the `RecipeDetailsScreen` for editing and creating a new recipe.<br>
In the `MainScreen` add a new navigation destination like this:

{% highlight kotlin %} 
composable(
   Navigation.RecipeDetails.route
) {
   RecipeDetailsScreen( 
       upPress = { navController.popBackStack() })
}
{% endhighlight %} 

When the `recipeId` is `null`,  we’ll use the route `/recipeDetails` and create a new recipe. <br>
Otherwise we’ll view or edit the one with provided id, using the route that we defined before: 

{% highlight kotlin %} 
composable(
   "${Navigation.RecipeDetails.route}/{id}",
   arguments = listOf(navArgument("id") { type = NavType.LongType })
) {
   RecipeDetailsScreen(
       recipeId = it.arguments!!.getLong("id"),
       upPress = { navController.popBackStack() })
}
{% endhighlight %} 

Change the parameter to nullable in `RecipeDetailsScreen`:

{% highlight kotlin %} 
RecipeDetailsScreen(recipeId: Long? = null, upPress: () -> Unit)
{% endhighlight %} 

Finally, add a floating button to create new recipes.<br>
In `MainScreen` add the missing parameter to `RecipesScreen` destination:

{% highlight kotlin %} 
onAddRecipe = { navController.navigate("recipeDetails") }
{% endhighlight %} 

Change the `RecipesScreen` signature to accept `onAddRecipe` callback:
{% highlight kotlin %} 
public fun RecipesScreen(onRecipeClicked: (RecipeResponse) -> Unit, onAddRecipe: () -> Unit)
{% endhighlight %} 

And wrap the `Recipes` composable in a `Scaffold` with a floating action button:

{% highlight kotlin %} 
Scaffold(floatingActionButton = {
   FloatingActionButton(onClick = onAddRecipe) {
       Icon(
           painter = painterResource(id = R.drawable.ic_add),
           contentDescription = null
       )
   }
}) {
   Recipes(items = recipes, onRecipeClicked = onRecipeClicked)
}
{% endhighlight %} 

Build and run the app.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-18/3.gif" alt="create topbar" width="300"/></div>

Now you can navigate to an empty skeleton of the “Create new recipe” screen, but it’s not functional yet.<br>
<b>In the next step</b> we’ll <b>add a new shared ViewModel</b> for the RecipeDetailScreen.
