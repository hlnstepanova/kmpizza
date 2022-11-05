---
title: "Step 28: Refactoring the iOS KMM App: NavigationLink and Binding"
layout: post
categories: kmm ios shared
--- 

There’s always room for perfection, so let’s brush up our iOS app a little bit. <br>
Go to `RecipesState` and add

{% highlight swift %} 
func getRecipes() {
    viewModel.getRecipes()
}
{% endhighlight %} 
 
Then in `RecipesView` make some small adjustments. <br>
Change

{% highlight swift %} 
    NavigationLink(destination: RecipeDetailView (
        id: recipe.id.toKotlinLong(), 
        isPresented: self.$isRecipeDetailsShown, 
        uploadSuccess: self.$uploadSuccess)
        ) 
{% endhighlight %} 

to

{% highlight swift %} 
    NavigationLink(destination: RecipeDetailView (
        id: recipe.id [1], 
        isPresented: self.$isRecipeDetailsShown, 
        uploadSuccess: self.$uploadSuccess)
        ) 
{% endhighlight %} 
 
<b>[1]</b> We’ll pass to `RecipeDetailView` a `recipeId` as `Int64?` and convert to `KotlinLong?` later.
 
Also in `onAppear()` change

{% highlight swift %} 
if uploadSuccess {
    state.viewModel.getRecipes()
}
{% endhighlight %} 

to 

{% highlight swift %} 
if uploadSuccess {
    state.getRecipes()
}
{% endhighlight %} 
 
Then in `FloatingActionButton` remove:

{% highlight swift %} 
.simultaneousGesture(
    TapGesture().onEnded {
        isPresented = true
    }
)
{% endhighlight %} 
 
Now go to `RecipeDetailsState`.<br>
Remove

{% highlight swift %} 
    @Published private(set) var upload: KotlinBoolean? = nil
{% endhighlight %} 
 
And replace 

{% highlight swift %} 
self.upload = upload
    if ((self.upload?.boolValue ?? false) == true){
        self.uploadSuccess = true
        self.isPresented = false
    }
{% endhighlight %} 
 
with just

{% highlight swift %} 
if ((upload?.boolValue ?? false) == true){
    self.uploadSuccess = true
    self.isPresented = false
}
{% endhighlight %} 
 
Also in `RecipeDetailState` add several published values, which we’ll use in `RecipeDetailView` after:

{% highlight swift %} 
    @Published var title: String = "" {
        didSet {
            viewModel.onTitleChanged(title: title)
        }
    }
    
    @Published var ingredients: [Ingredient] = [] {
        didSet {
            if let ingredient = ingredients.last {
            viewModel.onIngredientsChanged(ingredient: ingredient)
            }
        }
    }
    
    @Published var instructions: [Instruction] = [] {
        didSet {
            if let instruction = instructions.last {
            viewModel.onInstructionsChanged(instruction: instruction)
            }
        }
    }
    
    @Published var image: UIImage? = nil {
        didSet {
            viewModel.onImageChanged(image: image ?? UIImage())
        }
    }
{% endhighlight %} 

Now go to `RecipeDetailView`<br>
First change `init` to

{% highlight swift %} 
    init(id: Int64?, isPresented: Binding<Bool>, uploadSuccess: Binding<Bool> ) { [1]
        self.recipeId = id?.toKotlinLong()
        state = RecipeDetailState(recipeId: id?.toKotlinLong(), isPresented: isPresented, uploadSuccess: uploadSuccess)
    }
{% endhighlight %} 
 
<b>[1]</b> I think it’s better to pass `id` as `Int64?` here and then convert it to `KotlinLong?` for the `RecipeDetailState`<br>

Now in `EditIngredients` replace

{% highlight swift %} 
    var ingredients: [Ingredient]?
    var viewModel: RecipeDetailsViewModel
{% endhighlight %} 
 
with

{% highlight swift %} 
    @Binding var ingredients: [Ingredient]
{% endhighlight %} 
 
And change the `AddButton` to

{% highlight swift %} 
AddButton {
        ingredients.append(Ingredient(id: 0, name: name, amount: amount, metric: metric))
        name = ""
        amountString = ""
        metric = ""
    }.disabled(!isValid)
     .padding()
{% endhighlight %} 
 
Do the same for `EditInstrucitons`:<br>
Change
{% highlight swift %} 
    var instructions: [Instruction]?
    var viewModel: RecipeDetailsViewModel
{% endhighlight %} 
 
to
{% highlight swift %} 
    @Binding var instructions: [Instruction]
{% endhighlight %} 
 
Add
{% highlight swift %} 
private var isValid: Bool {
    return description != ""
}
{% endhighlight %} 
 
So that the add instruciton plus button is active only when the user types an instruction. Use it in body:
{% highlight swift %} 
 var body: some View {
        
        Instructions(instructions: instructions)
        
        HStack {
            Text ("\(instructions.count  + 1). ")
            TextField("Description", text: $description)
                .frame(maxWidth: .infinity, alignment: .leading)
        }
        .font(.body)
        
        AddButton {
            instructions.append(Instruction(id: 0, order: Int32(instructions.count + 1), description: description))
            description = ""
        }.disabled(!isValid)
            .padding()
    }
{% endhighlight %} 
 
As you can see, we no longer directly call `onChanged` methods in `RecipeDetailView`. Instead, we hid those in `RecipeDetailState` and used `@Published` to manipulate them from `RecipeDetailView`.
 
Now in `RecipeDetailView` replace

{% highlight swift %} 
                RecipePlaceholderView(image: Binding(
                    get: { state.recipe?.localImage},
                    set: {
                        if let image = $0 {
                            state.viewModel.onImageChanged(image: image)
                        }
                    }
                )).padding(16)
{% endhighlight %} 
with just

{% highlight swift %} 
RecipePlaceholderView(image: $state.image).padding(16)
{% endhighlight %} 
 
Replace

{% highlight swift %} 
TextField("What is the name of this dish?", text: Binding(
    get: { state.recipe?.title ?? "" },
    set: { state.viewModel.onTitleChanged(title: $0) }
    ))
    .multilineTextAlignment(.center)
    .padding()
{% endhighlight %} 
 
with

{% highlight swift %} 
TextField("What is the name of this dish?", text: $state.title)
    .multilineTextAlignment(.center)
    .padding()
{% endhighlight %} 
 
Then replace
{% highlight swift %} 
EditIngredients(ingredients: state.recipe?.ingredients, viewModel: state.viewModel )
{% endhighlight %} 
 
with
{% highlight swift %} 
EditIngredients(ingredients: $state.ingredients)
{% endhighlight %} 

And finally replace
{% highlight swift %} 
EditInstructions(instructions: state.recipe?.instructions, viewModel: state.viewModel )
{% endhighlight %} 
 
with
{% highlight swift %} 
EditInstructions(instructions: $state.instructions)
{% endhighlight %} 
 
As you can see, now the `body` looks leaner and you use binding in `RecipeDetailState` to save changes in the shared view model:
{% highlight swift %} 
 var body: some View {
        ScrollView {
            if (state.recipe?.images.isEmpty == false){
                KFImage(URL(string: state.recipe?.images.first?.image ?? ""))
                    .resizable()
                    .frame(width: 200, height: 150)
            } else {
                RecipePlaceholderView(image: $state.image).padding(16)
            }
            
            if (recipeId != nil) {
                Text(state.recipe?.title ?? "")
                    .font(.headline)
                    .padding()
            } else {
                TextField("What is the name of this dish?", text: $state.title)
                    .multilineTextAlignment(.center)
                    .padding()
            }
            
            Text("Ingredients")
                .font(.subheadline)
            if (recipeId != nil) {
                Ingredients(ingredients: state.recipe?.ingredients)
            } else {
                EditIngredients(ingredients: $state.ingredients)
            }
            
            Text("Instructions")
                .font(.subheadline)
            if (recipeId != nil) {
                Instructions(instructions: state.recipe?.instructions)
            } else {
                EditInstructions(instructions: $state.instructions)
            }
            
            if (recipeId == nil) {
                Button("Save recipe") {
                    state.saveRecipe()
                }
                .buttonStyle(FilledButtonStyle())
                .padding()
            }
            
        }.padding()
    }
{% endhighlight %} 

Build and run the app. It still works just like before:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-28/1.gif" alt="pizza app" width="300"/></div>