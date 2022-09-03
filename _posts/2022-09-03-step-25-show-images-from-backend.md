---
title: "Step 25: Show images from the KMP backend in the KMM Android and iOS apps"
layout: post
categories: kmm ios swift ui shared viewmodel compose android
--- 

Let’s make our app more colourful by <b>adding some pictures</b>. <br>
We’ll start with getting the UI ready to show the images from the backend.<br>
Later we’ll take care of uploading images.
 
We already have <b>Coil</b> and <b>Kingfisher</b> as image loading libraries in Android and iOS. <br>
So far we’ve been using only a static placeholder image for the recipes. Let’s change it. <br>
The backend already can send us a pizza image, so let’s use it. 
 
Go to `RecipeUiModel.kt` file in `commonMain/model`.<br>
Add a list of images to this model, so that it looks like this:
 
{% highlight kotlin %} 
data class RecipeUiModel (
   val id: Long = 0,
   val title: String,
   val ingredients: List<Ingredient> = listOf(),
   val instructions: List<Instruction> = listOf(),
   val images: List<RecipeImage> = listOf()
 )
{% endhighlight %} 
 
Adjust the `toRecipeRequest()`:

{% highlight kotlin %} 
fun RecipeResponse.toRecipeUiModel() = RecipeUiModel(
   id = id,
   title = title,
   ingredients = ingredients,
   instructions = instructions,
   images = images
)
{% endhighlight %} 
 
Now, in `RecipesRemoteSource` add a function `postImage()` that will later be responsible for sending a new image to the backend:

{% highlight kotlin %} 
   suspend fun postImage(recipeId: Long, imageFile: ImageFile) = recipesApi.postImage(recipeId, imageFile)
{% endhighlight %} 
 
In `RecipeRepository` change the methods as well so that we operate with `RecipeUiModel` here:

{% highlight kotlin %} 
class RecipeRepository : KoinComponent {
   private val recipeRemoteSource: RecipeRemoteSource by inject()
 
   suspend fun postRecipe(recipe: RecipeUiModel): Long? =
       recipeRemoteSource.postRecipe(recipe.toRecipeRequest())
 
   suspend fun getRecipes(): List<RecipeUiModel> =
       recipeRemoteSource.getRecipes().map { it.toRecipeUiModel() }
 
   suspend fun getRecipe(id: Long): RecipeUiModel =
       recipeRemoteSource.getRecipe(id).toRecipeUiModel()
 
}
{% endhighlight %} 
 
 
Modify `RecipeDetailsViewModel`. First remove redundant `toRecipeUiModel()` and `toRecipeRequest()`, because we moved them to `RecipeRepository`.
 
Also add an empty list of images to the initial value of recipe variable:
 
{% highlight kotlin %} 
private val _recipe = MutableStateFlow<RecipeUiModel?>(
   RecipeUiModel(
       title = "",
       ingredients = listOf(),
       instructions = listOf(),
       images = listOf()
   )
)
{% endhighlight %} 

As it’s more reasonable to use `RecipeUiModel` in our UI we’ll refactor all the occurrences of `RecipeResponse` to `RecipeUiModel`. This will also help us later to handle `postImage()` to the backend.
 
Go to `RecipeViewModel` and change all occurrences of `RecipeResponse` to `RecipeUiModel` to stay consistent. Check out the repository if you feel confused about how to do it. 
 
Now let’s switch to the UI, starting with Android.
 
In `RecipesScreen` change all occurrence of `RecipeResponse` to `RecipeUiModel` to stay consistent.
 
Also you’ll need to adjust `RecipeListItemPreview` in `RecipesScreen` to use `RecipeUiModel`:

{% highlight kotlin %} 
val recipe = RecipeUiModel(id = 0, title = title, listOf(), listOf(), listOf())
{% endhighlight %} 
 
Then change the `HeaderImage(image = placeholder, padding)` in `RecipeDetailsScreen` to:

{% highlight kotlin %} 
recipe?.images?.let {
    if (it.isNotEmpty()){
        HeaderImage(it[0].image) [1]
    } else {
        HeaderImage(placeholder) [2]
    }
} 
{% endhighlight %} 

<b>[1]</b> If we receive any images from backend, we’ll display the first one for now.<br>
<b>[2]</b> Otherwise we’ll stick to our placeholder<br>
 
If you run the app and open an existing recipe, you’ll see the pizza image that you uploaded, otherwise there will still be a placeholder image.
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-25/1.gif" alt="pizza on android" width="300"/></div>
 
Now let’s add a button to the placeholder image, which you’ll use to change the placeholder to a picture from your phone gallery. 
 
In `RecipeDetailsScreen` add a new composable:

{% highlight kotlin %}  
@Composable
private fun PlaceholderImage(padding: PaddingValues) {
   val placeholder = "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"
   Box(contentAlignment = Alignment.Center) {
       HeaderImage(image = placeholder, padding = padding)
       IconButton(
           onClick = {},
           modifier = Modifier.clip(CircleShape).background(Color.Cyan)
       ) {
           Icon(
               imageVector = Icons.Default.Add,
               contentDescription = "edit image",
               modifier = Modifier
                   .size(64.dp),
               tint = Color.Black
           )
       }
   }
}
{% endhighlight %} 
 
In `RecipeDetailsScreen` `Scaffold` after `recipe?.images?.let...` change
{% highlight kotlin %}  
HeaderImage(placeholder, padding)
{% endhighlight %}  

to 

{% highlight kotlin %}  
PlaceholderImage(padding)
{% endhighlight %}  
 
Build and run the app. <br>
Open a recipe without an uploaded image. You’ll see:
 
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-25/2.png" alt="add image placeholder" width="300"/></div>
 
So far your `onClick` doesn’t do anything, and we’ll change it later. 
 
But now let’s adapt our Swift UI code to the changes in shared logic.<br>
Change `RecipeResponse` to `RecipeUiModel` everywhere in the iOS project.
 
Go to `RecipeView` and change item’s type from `RecipeResponse` to `RecipeUiModel`.<br>
Do the same with recipes list in `RecipesState`. <br>
`RecipeDetailState` is already using `RecipeUiModel`.
 
Now go to `recipeDetail` group and add a `RecipePlaceHolderView` there. Just like in Android it shows a placeholder image with a button to edit it:

{% highlight swift %}  
import SwiftUI
import shared
import Kingfisher
 
struct RecipePlaceholderView: View {
    var body: some View {
            ZStack{
                KFImage(URL(string: "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"))
                   .resizable()
                   .frame(width: 200, height: 200)
 
                
                Image(systemName: "camera.circle.fill")
                    .font(.system(size: 28, weight: .light))
                    .foregroundColor(Color(#colorLiteral(red: 0.4352535307, green: 0.4353201389, blue: 0.4352389574, alpha: 1)))
                    .frame(width: 50, height: 50)
                    .background(Color(#colorLiteral(red: 0.9386131763, green: 0.9536930919, blue: 0.9635006785, alpha: 1)))
                    .clipShape(Circle())
            }
    }
}
{% endhighlight %}  
 
 
Finally instead of just showing a placeholder (with `KFImage`) in `RecipeDetailView`, we will switch between an image from the backend and a placeholder view
{% highlight swift %}  
      if (state.recipe?.images.isEmpty == false){
                KFImage(URL(string: state.recipe?.images.first))
                    .resizable()
                    .frame(width: 200, height: 200)
 
            } else {
                RecipePlaceholderView().padding(16)
            }
{% endhighlight %}   

Run the app: <b>now you can change the image of your recipe not only on Android, but also on iOS! </b>

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-25/3.gif" alt="show images on ios" width="300"/></div>
 
And it took us only several minutes to adjust this iOS functionality to what we already had in the shared code!
 
<b>In the next step</b> we’ll extend our shared code to <b>send an image to the backend</b> from the Android or iOS app.