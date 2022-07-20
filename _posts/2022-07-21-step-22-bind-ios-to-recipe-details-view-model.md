---
title: "Step 22: Bind iOS Swift UI to shared RecipeDetail ViewModel and add Navigation" 
layout: post
categories: kmm ios kingfisher
--- 

Now let’s add a new <b>Swift UI</b> Recipe Details screen and <b>Navigation</b> to it.<br>
Create `recipeDetail` folder and add `RecipeDetailState.swift` and `RecipeDetailView.swift` files:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-22/1.png" alt="create recipe detail state and view files" width="200"/></div>

Add `Navigation` to `ContentView`:

{% highlight swift %} 
        NavigationView {
            VStack {
                RecipesView()
            }
        }
{% endhighlight %} 
 
Then, add a navigation link to the recipes list in `RecipesView.swift` like this:

{% highlight swift %} 
List(state.recipes, id: \.id) { recipe in
            NavigationLink(destination: RecipeDetailView (id: KotlinLong.init(value: recipe.id))) { [1]
                    RecipeView(item: recipe)
               
            }
        }
{% endhighlight %} 
 
<b>[1]</b> Here we’re transforming Swift `Int64?` To `KotlinLong?`, which is a Kotlin Native type of `recipeId`<br>
 
To make it neater, let’s write an extension.
Create a separate `Extensions.swift` in a `utils` group:
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-22/2.png" alt="add toKotlinLong() extension" width="200"/></div>

{% highlight swift %} 
import shared
 
extension Int64 {
    func toKotlinLong() -> KotlinLong {
        return KotlinLong(value: self)
    }
}
{% endhighlight %} 
 
Now you can use this extension instead of `KotlinLong.init(value: recipe.id))`:

{% highlight swift %} 
NavigationLink(destination: RecipeDetailView (id: recipe.id.toKotlinLong())) {
                    RecipeView(item: recipe)
               
            }
{% endhighlight %} 
 
Populate the `RecipeDetailState.swift`:

{% highlight swift %} 
import SwiftUI
import Foundation
import shared
 
class RecipeDetailState: ObservableObject{
    
    let recipeId: KotlinLong? [1]
    let viewModel: RecipeDetailsViewModel [2]
    
    @Published private(set) var recipe: RecipeUiModel? = nil [3]
    @Published private(set) var upload: Bool? = nil
    
    init(recipeId: KotlinLong?) {
        self.recipeId = recipeId
        viewModel = RecipeDetailsViewModel(id: recipeId)
        
        viewModel.observeRecipe { recipe in
            self.recipe = recipe
        }
    }
    
    deinit {
        viewModel.dispose()
    }
}
{% endhighlight %} 
 
<b>[1]</b> We will use `KotlinLong` for our recipe ids<br>
<b>[2]</b> The shared `RecipeDetailsViewModel` will hold the data and deliver it to iOS through Kotlin flows, just like it did before in the recipes list view model<br>
<b>[3]</b> In this `@Published` `recipe` property we’ll store the recipe details received through the shared view model.<br>
 
Finally, add this to the `RecipeDetailView.swift`:

{% highlight swift %} 
import SwiftUI
import shared
import Kingfisher
 
struct RecipeDetailView: View {
    
    let recipeId: KotlinLong?
    @ObservedObject var state: RecipeDetailState
    
    init(id: KotlinLong?) {
        self.recipeId = id
        state = RecipeDetailState(recipeId: id)
    }
    
    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(state.recipe?.ingredients ?? [], id: \.id, content: { ingredient in
                    Text(ingredient.name)
                        .font(.body)
                })
            }
        }
        
    }
    
}
{% endhighlight %} 
 
 
<b>[1]</b> The observable `RecipeDetailState` will hold the data for the `RecipeDetailView`<br>
<b>[2]</b> To begin with, we are just displaying a list of ingredients to make sure everything works<br>
 
Run the App now and try navigation to the pizza recipe. You’ll see the list of ingredients.
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-22/3.png" alt="add toKotlinLong() extension" width="300"/></div> 
 
Now add other views to the recipe detail screen:

{% highlight swift %} 
var body: some View {
        ScrollView { [1]
            Text(state.recipe?.title ?? "")
                .font(.headline)
           KFImage(URL(string: "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"))
                .resizable()
                .frame(width: 200, height: 150)
                .padding(.trailing)
            
            Text("Ingredients") [3]
                .font(.subheadline)
            LazyVStack {
                ForEach(state.recipe?.ingredients ?? [], id: \.id, content: { ingredient in
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
            
            Text("Instructions") [4]
                .font(.subheadline)
            LazyVStack {
                ForEach(state.recipe?.instructions ?? [], id: \.id, content: { instruction in
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
 
<b>[1]</b> Use `ScrollView` to make `RecipeDetailsView` scrollable in case it has to many ingredients or instructions<br>
<b>[2]</b> Add a recipe title image with `Kingfisher` image library<br>
<b>[3]</b> Add a section with ingredients’ names and measurements<br>
<b>[4]</b> Add a section with instructions<br>
 
Run the app again, go to recipe detail view and you’ll see, it looks much better now.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-22/4.png" alt="add toKotlinLong() extension" width="300"/></div>
 
Our <b>next step</b> will be: adding a <b>floating button</b> and <b>edit</b> recipe functionality.