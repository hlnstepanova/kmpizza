---
title: "Step 23: Create editable Recipe Details View in Swift UI and bind to the shared View Model" 
layout: post
categories: kmm ios swift ui shared viewmodel
--- 

For a `Floating Action Button` add the following `struct` in `RecipesView`:
{% highlight swift %} 
struct FloatingActionButton: View {
    var body: some View {
        Button(action: {
            print("Add new recipe") [1]
        }) {
            Image(systemName: "plus.circle.fill") [2]
                .resizable()
                .frame(width: 56, height: 56)
                .foregroundColor(Color.accentColor)
                .shadow(color: .gray, radius: 0.2, x: 1, y: 1)
        }
    } 
}
{% endhighlight %} 
 
<b>[1]</b> This closure defines an action that gets executed when the user clicks the button, for now it’s just printing in log output<br>
<b>[2]</b> Here we describe how the button looks<br>
 
 
Then use it in `RecipesView` like this:
{% highlight swift %} 
var body: some View {
        ZStack(alignment: .bottomTrailing) { [1]
            List(state.recipes, id: \.id) { recipe in
                NavigationLink(destination: RecipeDetailView (id: recipe.id.toKotlinLong())) {
                    RecipeView(item: recipe)
                    
                }
            }
            .listStyle(PlainListStyle())
            FloatingActionButton() [2]
                .padding()
                .frame(alignment: .bottomTrailing)
        }
    }
{% endhighlight %} 
 
<b>[1]</b> We’re using `ZStack` to place the `FloatingActionButton` over the `Recipe List`<br>
<b>[2]</b> Use the `FloatingActionButton` `struct` we created before<br>
 
Build and run the app.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-23/1.png" alt="Big blue add recipe button" width="300"/></div>

Click on the button and you’ll see `Add new recipe` printed in the `log`.<br>
We need to change this to actually be able to add a recipe.<br>
To do so, replace the `Button` with `NavigationLink`
{% highlight swift %} 
struct FloatingActionButton: View {
    var body: some View {
        NavigationLink(destination: RecipeDetailView (id: nil)) {
            Image(systemName: "plus.circle.fill")
                .resizable()
                .frame(width: 70, height: 70)
                .foregroundColor(Color.accentColor)
                .shadow(color: .gray, radius: 0.2, x: 1, y: 1)
        }
    }
}
{% endhighlight %} 
 
<b>[1]</b> Instead of using a `Button` that prints a log we’ll navigate to `RecipeDetailView`
 
In the `RecipeDetailView` change the title item so that if we already have a recipe (with `recipeId`), it displays a title, otherwise it displays a text field:
{% highlight swift %} 
ScrollView {
            KFImage(URL(string: "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"))
                .resizable()
                .frame(width: 200, height: 150)
            
            if (recipeId != nil) {
                Text(state.recipe?.title ?? "")
                    .font(.headline)
                    .padding()
            } else {
                TextField("What is the name of this dish?", text: Binding(
                    get: { state.recipe?.title ?? "" },
                    set: { state.viewModel.onTitleChanged(title: $0) }
                ))
                .multilineTextAlignment(.center)
                .padding()
            }
. . .
}
{% endhighlight %} 
 
<b>[1]</b> If it’s an old recipe with `recipeId`, display the title<br>
<b>[2]</b> Otherwise create an input field<br>
<b>[3]</b> The title gets updated according to the recipe title from the shared `RecipeDetailsViewModel`<br>
<b>[4]</b> when the user enters a new title the value in the shared model is updated via `onTitleChanged()`<br>
 
You’ll also need a plus button to add more ingredients or instructions like the one we had in Android.<br>
Add this `struct` below the `RecipeDetailView` `struct`:
{% highlight swift %} 
struct AddButton: View {
    let action: () -> Void
    
    var body: some View {
        Button(action: action) {
            Image(systemName: "plus.circle.fill")
                .resizable()
                .frame(width: 30, height: 30)
                .foregroundColor(Color.accentColor)
                .shadow(color: .gray, radius: 0.2, x: 1, y: 1)
        }.padding()
    }  
}
{% endhighlight %} 
 
Add other structs for the UI, starting with the `Ingredients` section.<br>
This `View` shows a simple list of ingredients with respective names, amounts and metric and arranges them in a similar way like in `Android`:

{% highlight swift %} 
struct Ingredients: View {
    
    var ingredients: [Ingredient]?
    
    var body: some View {
        LazyVStack {
            ForEach(Array(ingredients?.enumerated() ?? [].enumerated()), id: \.offset) { index, ingredient in
                HStack {
                    Text(ingredient.name)
                        .font(.body)
                        .frame(maxWidth: .infinity, alignment: .leading)
                    
                    HStack {
                        Text("\(ingredient.amount, specifier: "%.1f")")
                        Text(ingredient.metric)
                            .font(.body)
                    }
                }
                
            })
        }
        .padding()
    }
}
{% endhighlight %} 

<b>[1]</b> Here we can't use `id` to build a `LazyVStack`, because all of them has the same `id = nil`, that's why we are using `enumerated()` as workaround to display the ingredients. Otherwise you'll get an error: `LazyVStackLayout: the ID nil is used by multiple child views, this will give undefined results!`
 
The `Instructions` section is similar to the `Ingredients` section:

{% highlight swift %} 
struct Instructions: View {
    
    var instructions: [Instruction]? [1]
    
    var body: some View {
        LazyVStack {
            ForEach(instructions ?? [], id: \.self, content: { instruction in [1]
                HStack {
                    Text("\(instruction.order). ")
                        .font(.body)
                    Text(instruction.description_)
                        .font(.body)
                        .frame(maxWidth: .infinity, alignment: .leading)
                }
                .padding()
                
            })
        }
    }   
}
{% endhighlight %} 

<b>[1]</b> Using `\.self` is another way to avoid the `LazyVStackLayout` error, but for instructions we could also use `\.order`, because every order is unique
 
Previous structs were not editable, but we also want the user to be able to add or edit recipes in our `RecipeDetailsView`:

{% highlight swift %} 
struct EditIngredients: View {
    
    var ingredients: [Ingredient]?
    var viewModel: RecipeDetailsViewModel [1]
    
    @State private var name: String = "" [2]
    @State private var amountString: String = ""
    @State private var metric: String = ""
    
    private let formatter = NumberFormatter()
    
    private var amount: Double { [3]
        return formatter.number(from: amountString)?.doubleValue ?? 0
    }
    
    private var isValid: Bool {
        return name != "" && amount > 0 && metric != ""
    }
      
    var body: some View {
        Ingredients(ingredients: ingredients)
        
        HStack {
            TextField("Ingredient", text: $name)
                .frame(maxWidth: .infinity, alignment: .leading)
            
            HStack {
                TextField("Amount", text: $amountString)
                TextField("Metric", text: $metric)
            }
        }
        .font(.body)
        
        AddButton(action: {
            viewModel.onIngredientsChanged(ingredient: Ingredient(id: nil, name: name, amount: amount, metric: metric)) [4]
            name = ""
            amountString = ""
            metric = ""
        })
        .padding()
    }
}
{% endhighlight %} 
 
<b>[1]</b> This creates an instance of the shared `RecipeDetailsViewModel`, so that we can use `onIngredietnsChanged()` function from it<br>
<b>[2]</b> `Name`, `amountString` and `metric` are editable states<br>
<b>[3]</b> In Swift `Optional` and `NumberFormatter()` don't get along well, therefore we need cusotm logic to bind values to `TextField`, because we don't want to see anything but a hint in the Amount `TextField` if `amount==nil`<br>
<b>[4]</b> Here we only care about updating the `RecipeDetailsViewModel` after the user clicks the Add (Ingredient) Button again: then the current state of the previous ingredient is refreshed in the `RecipeDetailsViewModel`<br>
 
We use the same principle for `EditInstructions`:

{% highlight swift %}
struct EditInstructions: View {
    
    var instructions: [Instruction]?
    var viewModel: RecipeDetailsViewModel
    
    @State private var description: String = ""
    
    
    var body: some View {
        
        Instructions(instructions: instructions)
        
        HStack {
            Text ("\((instructions?.count ?? 0) + 1). ")
                .font(.body)
            TextField("Description", text: $description)
                .font(.body)
                .frame(maxWidth: .infinity, alignment: .leading)
    
        }
        
        
        AddButton(action: {
            viewModel.onInstructionsChanged(instruction: Instruction(id: nil, order: Int32((instructions?.count ?? 0) + 1), description: description))
            description = ""
        })
        
            .padding() 
    } 
}

{% endhighlight swift %} 
 
Then simply add all these components to the `RecipeDetailView` `body`:
{% highlight swift %} 
 var body: some View {
        ScrollView {
            KFImage(URL(string: "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"))
                .resizable()
                .frame(width: 200, height: 150)

            if (recipeId != nil) {
                Text(state.recipe?.title ?? "")
                    .font(.headline)
            } else {
                TextField("What is the name of this dish?", text: Binding(
                    get: { state.recipe?.title ?? "" },
                    set: { state.viewModel.onTitleChanged(title: $0) }
                )).multilineTextAlignment(.center)
            }
            
            Text("Ingredients") [1]
                .font(.subheadline)
            if (recipeId != nil) {
                Ingredients(ingredients: state.recipe?.ingredients)
            } else {
                EditIngredients(ingredients: state.recipe?.ingredients, viewModel: state.viewModel )
            }
            
            Text("Instructions") [2]
                .font(.subheadline)
            if (recipeId != nil) {
                Instructions(instructions: state.recipe?.instructions)
            } else {
                EditInstructions(instructions: state.recipe?.instructions, viewModel: state.viewModel )
            }
        }
    }

{% endhighlight %} 
 
<b>[1]</b> If there’s already a `recipeId`, show the recipe details, otherwise use the editable struct<br>
<b>[2]</b> Same for the instructions section<br>

Build and run the app.
If you choose a recipe from the list you'll see the details view as before:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-23/2.png" alt="Old recipe details screen" width="300"/></div>

If you click on the big blue add recipe button you'll see the empty add recipe screen:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-23/3.png" alt="Add new recipe details screen" width="300"/></div>

With this setup we can either view the recipe details OR add a new one, meaning we can’t edit an existing recipe. But that’s enough for now, we can adjust this behaviour later.<br>

Last but not least, we need a button to save the new recipe.<br>
Add `saveRecipe()` function to `RecipeDetailState` to leverage the method from the shared `RecipeDetailsViewModel`:

{% highlight swift %} 
func saveRecipe(){
        viewModel.saveRecipe()
    }
{% endhighlight %}   


Create a style for the submit button:

{% highlight swift %} 
struct FilledButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration
            .label
            .font(.title3)
            .foregroundColor(.white)
            .padding(.horizontal)
            .padding(.vertical, 12)
            .frame(maxWidth: .infinity)
            .background(configuration.isPressed ? Color.blue.opacity(0.3) : Color.blue)
            .cornerRadius(25)
    }
}
{% endhighlight %} 
 

Then place it at the end of the `body` inside `RecipeDetailView`:

{% highlight swift %} 
if (recipeId == nil) {
                Button("Save recipe") {
                    state.saveRecipe()
                }
                .buttonStyle(FilledButtonStyle())
                .padding()
            }
{% endhighlight %}  

Build and run the app. Here it is, our `saveRecipe()` button:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-23/4.png" alt="Add new recipe details screen" width="300"/></div>

We want the RecipeDetailsView to close automatically once the user saves the recipe.
Add to the top of `RecipeDetailView` `struct`:

{% highlight swift %} 
@Environment(\.presentationMode) var presentationMode
{% endhighlight swift %} 

Use it to close the view when clicking on save `Button` (this will later be enhanced with upload flow):

{% highlight swift %} 
        Button("Save recipe") {
            state.saveRecipe()
            presentationMode.wrappedValue.dismiss()
        }
{% endhighlight %} 
 
Build and run the iOS app.<br>
Now we can fill out the form and press the save button, but there's <b>no new recipe in the list</b>...Why?<br>
We’ll investigate it <b>next time</b>.