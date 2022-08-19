---
title: "Step 24: Add a shared Log and navigate back on upload success in the iOS app"
layout: post
categories: kmm ios swift ui shared viewmodel
--- 

To investigate what’s happening in iOS when we want to save a new recipe let’s add a logging function to the `shared` module.
 
In `commonMain` `util` folder add a new file `Log.kt` with the following expect function:
{% highlight swift %} 
expect val log: (String) -> Unit
{% endhighlight %} 
 
Add corresponding actual functions to `androidMain` and `iosMain`, like we did before with `CoroutineViewModel`.
 
<b>androidMain:</b>
{% highlight swift %} 
actual val log: (String) -> Unit = {
   Log.d("RecipesLog", it)
}
{% endhighlight %} 
 
<b>iosMain:</b>
{% highlight swift %} 
actual val log: (String) -> Unit = { println(it) }
{% endhighlight %} 
 
Also add a missing actual function to `jvmMain`:
{% highlight swift %} 
actual val log: (String) -> Unit = { }
{% endhighlight %} 
 
Now add some log output to the `saveRecipe` function:
{% highlight swift %} 
fun saveRecipe() {
   coroutineScope.launch {
           recipe.value?.let {
               if (it.title.isNotEmpty() && it.ingredients.isNotEmpty() && it.instructions.isNotEmpty()){
                   log(it.toString())
                   val result = recipeRepository.postRecipe(it)
                   log(result.toString())
                   result?.let { _upload.value = true }
               }
           }
   }
}
{% endhighlight %} 
 
Make sure you imported the `shared` log function.<br>
<b>Build and run</b> the Android app.<br>
Try saving a new recipe, you’ll see:<br>
{% highlight bash %} 
D/RecipesLog: RecipeUiModel(id=0, title=Pie, ingredients=[Ingredient(id=0, name=Flour, amount=1.0, metric=Kg)], instructions=[Instruction(id=0, order=1, description=Mix all ingredients)])
D/RecipesLog: 10
{% endhighlight %} 
 
Now do the same for iOS, you’ll see the following log output:
{% highlight bash %} 
RecipeUiModel(id=0, title=Pie, ingredients=[Ingredient(id=null, name=Flour, amount=0.0, metric=Kg)], instructions=[Instruction(id=null, order=1, description=Mix all together)])
null
{% endhighlight %} 
 
See the difference? <br>
Right! We forgot that we use `Long` for our backend and shouldn’t be setting them to `null`, so let’s change it to `0` in `AddButtons`:
{% highlight swift %} 
    viewModel.onIngredientsChanged(ingredient: Ingredient(id: 0, name: name, amount: amount ?? 0.0, metric: metric))
    viewModel.onInstructionsChanged(instruction: Instruction(id: 0, order: Int32((instructions?.count ?? 0) + 1), description: description))
{% endhighlight %} 
 
Now <b>run the app</b> again...and it kind of works: if you start the app again and refresh the recipes list, you’ll see the new recipe! 
 
However, we want to see the new recipe in the list once we save it.<br>
We will fix it up later.<br> 

First let’s handle the `upload` value from the shared `RecipeDetailViewModel`, because we want to close the recipe detail view only after the request succeeded.<br>

Change the `upload` variable in `RecipeDetailState` to `KotlinBoolean?`:

{% highlight swift %} 
@Published private(set) var upload: KotlinBoolean? = nil
{% endhighlight %} 
 
Add the following observe function in `init()`:

{% highlight swift %} 
viewModel.observeUpload{ upload in
            self.upload = upload
        }
{% endhighlight %} 
 
Now we need to adjust the app’s behavior when Save Recipe button in clicked.
Remove  `presentationMode.wrappedValue.dismiss()` from the Save Button in `RecipeDetailView`, as well as `Environment variable @Environment(\.presentationMode) var presentationModel`.
 
In `RecipesView` add
{% highlight swift %} 
@State var isRecipeDetailsShown: Bool = false
{% endhighlight %} 
 
We’ll bind the `isRecipeDetailsShown` with the `RecipeDetailsView`, so that `isRecipeDetailsShown` is manipulated within the `RecipeDetailsScreen` and receives the result of the `upload` variable from the shared view model.
 
Add to the `RecipeDetailState` the following variable:
{% highlight swift %} 
@Binding var isPresented: Bool
{% endhighlight %} 
 
Change `init()`:
{% highlight swift %} 
 init(recipeId: KotlinLong?, isPresented: Binding<Bool>) {
        self.recipeId = recipeId
        self._isPresented = isPresented
        
        viewModel = RecipeDetailsViewModel(id: recipeId)
        
        viewModel.observeRecipe { recipe in
            self.recipe = recipe
        }
    }
{% endhighlight %} 
 
 
And move the `observeUpload()` to `saveRecipe()`:
{% highlight swift %} 
 func saveRecipe(){
        viewModel.saveRecipe()
        
        viewModel.observeUpload { upload in
            self.upload = upload
            if ((self.upload?.boolValue ?? false) == true){
                self.isPresented = false
            }
        }
    }
{% endhighlight %} 
 
Also change `RecipeDetailView` by passing the `isPresented` binding further to the state:
{% highlight swift %} 
  init(id: KotlinLong?, isPresented: Binding<Bool>) {
        self.recipeId = id
        state = RecipeDetailState(recipeId: id, isPresented: isPresented)
    }
{% endhighlight %} 
 
The `body` in `RecipeDetailView` should trigger `saveRecipe()`:
{% highlight swift %} 
 if (recipeId == nil) {
                Button("Save recipe") {
                    state.saveRecipe()
                }
                .buttonStyle(FilledButtonStyle())
                .padding()
            }
{% endhighlight %} 
 
 
Then in `RecipesView` we’ll pass this state variable to our `RecipeDetailsView` like this. 
{% highlight swift %} 
 NavigationLink(destination: RecipeDetailView (id: recipe.id.toKotlinLong(), isPresented: self.$isRecipeDetailsShown)) {
                    RecipeView(item: recipe)
                }
{% endhighlight %} 
 
Finally change the `FloatingActionButton`, so that we can signal when it’s pressed and the `RecipeDetailsView` is shown or not.
 
~~~ swift
struct FloatingActionButton: View {
    @Binding var isPresented: Bool
    
    var body: some View {
        NavigationLink(destination: RecipeDetailView (id: nil, isPresented: $isPresented), isActive: $isPresented) {
            Image(systemName: "plus.circle.fill")
                .resizable()
                .frame(width: 56, height: 56)
                .foregroundColor(Color.accentColor)
                .shadow(color: .gray, radius: 0.2, x: 1, y: 1)
        }
        .simultaneousGesture(
            TapGesture().onEnded {
                isPresented = true
            })
    }
}
~~~
 
 
Don’t forget to pass the value to `FloatingActionButton` in `RecipesView` body:
{% highlight swift %} 
FloatingActionButton(isPresented: self.$isRecipeDetailsShown)
{% endhighlight %} 
 
 
Now if you test the app your `RecipeDetailView` will close only after it receives `upload ==  true` from the shared view model.
 
However, the recipe list still doesn’t update when we go back. <br>Let’s solve this problem now.<br>
Open `RecipeDetailState` and add another variable just like you did with `isPresented`:
{% highlight swift %} 
    @Binding var uploadSuccess: Bool
{% endhighlight %} 
 
In `init` add `uploadSuccess: Binding<Bool>` to the signature
and initiate with
{% highlight swift %} 
        self._uploadSuccess = uploadSuccess
{% endhighlight %} 
 
In `saveRecipe` below `self.isPresented = false add self.uploadSuccess = true`
 
In `RecipeDetailView` change the init section:
{% highlight swift %} 
init(id: KotlinLong?, isPresented: Binding<Bool>, uploadSuccess: Binding<Bool> ) {
        self.recipeId = id
        state = RecipeDetailState(recipeId: id, isPresented: isPresented, uploadSuccess: uploadSuccess)
    }
{% endhighlight %} 
 
Finally, in `RecipesView` add 
{% highlight swift %} 
@State var uploadSuccess: Bool = false 
{% endhighlight %} 

and pass it to `RecipeDetailView` where necessary:
 
 {% highlight swift %} 
NavigationLink(destination: RecipeDetailView (id: recipe.id as? KotlinLong, isPresented: self.$isRecipeDetailsShown, uploadSuccess: self.$uploadSuccess)) {
        RecipeView(item: recipe)
    }
 FloatingActionButton(isPresented: self.$isRecipeDetailsShown, uploadSuccess: self.$uploadSuccess)
 {% endhighlight %} 
 
Also adjust `FloatingActionButton` struct:
 {% highlight swift %} 
NavigationLink(destination: RecipeDetailView (id: nil, isPresented: $isPresented, uploadSuccess: self.$uploadSuccess), isActive: $isPresented)
 {% endhighlight %} 
 
Add now an `onAppear` callback to `NavigationLink` in `RecipesView:`
 {% highlight swift %} 
    .onAppear {
        if uploadSuccess {
            state.viewModel.getRecipes()
        }
{% endhighlight %} 

This will make sure that the shared view model gets the latest recipe list from the api and saves it in `Kotlin Flow`, which in turn passes it to `Swift UI`.

<b>In the next step</b> we'll learn how to show <b>images from our backend</b> in the KMM Android and iOS apps.

