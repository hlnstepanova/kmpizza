---
title: "Step 20: Connect shared RecipeDetails ViewModel to Android Compose UI" 
layout: post
categories: kmm shared viewmodel ktor koin
--- 
Now that we have a shared `RecipeDetailsViewModel` it's time to build the UI, and we'll start with Android.

First, add some values to strings.xml in `res/values` folder:
{% highlight xml %} 
<resources>
   <string name="name">Name</string>
   <string name="amount">Amount</string>
   <string name="metric">Metric</string>
   <string name="description">Description</string>
   <string name="save">Save</string>
   <string name="title">What is the name of this dish?</string>
</resources>
{% endhighlight %} 

In <b>RecipeDetailsScreen.kt</b> use the new `RecipeDetailsViewModel`:

{% highlight kotlin %} 
   val viewModel = remember { RecipeDetailsViewModel(recipeId) }
   val recipe by viewModel.recipe.collectAsState()
{% endhighlight %}

Add an `upload` state and react to its changes:

{% highlight kotlin %} 
val upload by viewModel.upload.collectAsState()

if (upload){
   upPress()
   viewModel.resetUpload()
}
{% endhighlight %} 

Adjust the `Scaffold` to have a scrolling state and add `Edit` fields in case when the `recipeId` is `null` (i.e. we want to add a new recipe):

{% highlight kotlin %} 
val scrolling = Modifier.verticalScroll(rememberScrollState())

Scaffold(
   topBar = { TopBar(upPress = upPress) })
{
   Column(
       modifier = scrolling.padding(8.dp),
       horizontalAlignment = Alignment.CenterHorizontally
   ) {
       HeaderImage(image = placeholder)
       recipeId?.let { recipe?.title?.let { Title(it) } } ?: EditTitle(changeListener = viewModel, title = recipe?.title)
       SectionHeader(title = "Ingredients")
       recipeId?.let { recipe?.ingredients?.let { Ingredients(it)} } ?: EditIngredients(changeListener = viewModel, items = recipe?.ingredients)
       SectionHeader(title = "Instructions")
       recipeId?.let { recipe?.instructions?.let { Instructions(it) }} ?: EditInstructions(changeListener = viewModel, items = recipe?.instructions)
       if (recipeId == null){
           SubmitButton {
               viewModel.saveRecipe()
           }
       }
   }
}
{% endhighlight %} 

With the <b>Submit</b> button we’ll send the recipe data to backend:

{% highlight kotlin %} 
@Composable
fun SubmitButton(
   onClick: () -> Unit){
   ExtendedFloatingActionButton(
       text = { Text(text = stringResource(id = R.string.save))},
       shape = RoundedCornerShape(51),
       onClick = onClick,
       modifier = Modifier
           .fillMaxWidth()
           .padding(vertical = 4.dp)
   )
}
{% endhighlight %} 

These new "edit recipe" components control user interactions with the input fields and pass the appropriate changes to the view model. <br>
Here’s an example of `EditInstructions`:

{% highlight kotlin %} 
@Composable
private fun EditInstructions(changeListener: EditRecipeChangeListener, items: List<Instruction>?) {
   var isEdited by remember { mutableStateOf(false) } [1]
   var description by remember { mutableStateOf("") } [2]

   val onDescriptionChanged = { value: String ->
       description = value
   }

   val onInstructionAdded = {
       isEdited = true
       if (description.isNotEmpty()){
           changeListener.onInstructionsChanged(Instruction(order = items?.size?.plus(1) ?: 1, description = description))
           description = ""
       }
   } [3]

  Instructions(items = items) [4]
   if (isEdited){
       NewInstruction(description = description, onDescriptionChanged = onDescriptionChanged)
   } [5]
   AddItemButton(onAddInstruction = onInstructionAdded) [6]
}
{% endhighlight %} 

<b>[1]</b> Keep track of whether the `Instructions` section is being edited<br>
<b>[2]</b> Holds the current state of the instruction description, which is manipulated by the user<br>
<b>[3]</b> The `onInstructionAdded` callback follows the `AddItemButton` click and activates a new description field tor user input. When the user clicks the button again it adds the previous instruction to the list of instructions for this recipe via `viewModel.onInstructionsChanged()`<br>
<b>[4]</b> Shows all the instructions that have already been added<br>
<b>[5]</b> Under all the added instructions we have a field for user input, which is shown when `isEdited` was triggered<br>
<b>[6]</b> At the bottom of the section there’s an `AddItemButton` which allows the user to add a new instruction<br>

{% highlight kotlin %} 
@Composable
private fun AddItemButton(onAddInstruction: () -> Unit = {}) {
   IconButton(onClick = onAddInstruction) {
       Icon(
           painter = rememberAsyncImagePainter(R.drawable.ic_add),
           contentDescription = null,
           modifier = Modifier.clip(CircleShape))
   }
}
{% endhighlight %} 

<u><b>Check the repository for the full code and other components like EditTitle, EditIngredients and others.</b></u><br> [KMPizza Repo](https://github.com/hlnstepanova/kmpizza-repo)

Once your UI is ready, run the project.<br>
You’ll encounter an error saying that you received an unexpected variable in your JSON Response. That’s because you have images in the json `RecipeResponse`, but the `Recipe` entity on the app side doesn’t. <br>

Let’s fix it by temporarily changing the types. This way we’ll avoid errors with images, which we’ll add later on to the backend.

{% highlight kotlin %} 
internal class RecipeRemoteSource(
   private val recipesApi: RecipesApi
) {

   suspend fun getRecipes() = recipesApi.getRecipes().map { it.toRecipe() }
   suspend fun getRecipe(id: Long) = recipesApi.getRecipe(id).toRecipe()
   suspend fun postRecipe(recipe: Recipe) = recipesApi.postRecipe(recipe)
}

fun RecipeResponse.toRecipe() = Recipe (id = id, title = title, ingredients = ingredients, instructions = instructions)
{% endhighlight %} 

Now it's fixed and the Android UI is ready. <br>
<b>In the next</b> step we'll move on to playing with <b>Swift UI</b>.
