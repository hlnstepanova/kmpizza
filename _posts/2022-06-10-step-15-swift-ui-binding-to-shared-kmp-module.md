---
title: "Step 15: Bind iOS UI to the shared KMM ViewModel" 
layout: post
categories: shared swift ios kmm viewmodel koin
--- 

Now let’s move to the iOS side for a while and implement the UI there.<br>
First, open the workspace from iosApp folder inside your project folder in XCode.

<img src="{{site.baseurl}}/assets/images/step-15/1.png" alt="open iosApp workspace" width="350"/>

Open ContentView

<img src="{{site.baseurl}}/assets/images/step-15/2.png" alt="open ContentView" width="250"/>

There you’ll see

{% highlight swift %}
struct ContentView: View {
	let greet = Greeting().greeting()
 
	var body: some View {
		Text(greet)
	}
}
{% endhighlight %} 
 
As we’ve already removed the `Greeting` file from our shared module, we won’t be able to access it here, so let’s replace

{% highlight swift %} 
let greet = Greeting().greeting()
{% endhighlight %} 

with

{% highlight swift %}
let greet = "Give me pizza!"
{% endhighlight %} 

You should be able to run the code now and see the app asking for pizza.
 
Let’s bind this iOS app to the recipe backend just like we did for Android.<br>
The networking logic is already in the shared module. All we need to do is to find a way to use that logic in our ios UI.
 
First, create an observable state for our recipe list screen. <br>
Organise the iosApp structure bit: Create a New Group `ui` and move `ContenView` there.<br>
Also add `recipes` group to the `ui` package. <br>
In `recipes` Create a New File `RecipesState`.

<img src="{{site.baseurl}}/assets/images/step-15/3.png" alt="create RecipesState" width="250"/>
 
The simplest version of `RecipesState` looks like this:
{% highlight swift %} 
import Foundation
import shared
 
class RecipesState: ObservableObject { [1]     
    let viewModel: RecipeViewModel
    
    @Published private var recipes: [RecipeResponse]  = [] [2]
    
    init() { 
        viewModel = RecipeViewModel() [3]
         
        viewModel.observeRecipes { recipes in
            self.recipes = recipes [4]
        }
    }
    
    deinit {
        viewModel.dispose()
    }
}
{% endhighlight %} 
 
<b>[1]</b> `@Published` wrapper combined with the `ObservableObject` protocol will help us to observe changes to the published properties within this observable `RecipesState` and redraw our UI accordingly.<br>
<b>[2]</b> In this `@Published` `recipes` list property we’ll store the recipes received from backend through the `shared` module.<br>
<b>[3]</b> Look how easily we can initiate the shared `RecipeViewModel` instance in iOS!<br>
<b>[4]</b> Thanks to `observeRecipes` function from the `shared` module we can get the recipes from the backend and use them for the iOS UI<br>

Now, add `RecipesView` file to the `recipes` group.<br>
Use the observable `RecipesState` object in the `RecipesView` like this:

{% highlight swift %}
import SwiftUI
import shared
 
struct RecipesView: View {
     
    @ObservedObject var state: RecipesState [1]
    
    init() { 
        state = RecipesState()
    }
    
    var body: some View {
            List(state.recipes, id: \.id) { recipe in [2]
                    Text(recipe.title)
            }
            .listStyle(PlainListStyle())
    }
}
{% endhighlight %} 
 
<b>[1]</b> Here we’re using the `RecipesState` as an `ObservedObject`. <br>
<b>[2]</b> Whenever the `@Published` list of recipes in `RecipesState` changes, we’ll refresh the `List` (analog for `lazyColumn` in Compose) with recipe titles.
 
Now change the main `ContentView` to display the `RecipeView`:

{% highlight swift %} 
struct ContentView: View {
    
    var body: some View {
            VStack {
                RecipesView()
            }
    }
}
{% endhighlight %} 
 
But if you try running the app, you’ll get an error:
```Uncaught Kotlin exception: kotlin.Throwable: The process was terminated due to the unhandled exception thrown in the coroutine [StandaloneCoroutine{Cancelling}@258c458, MainDispatcher]: KoinApplication has not been started```

AGAIN?! 😱 <br>
No panic. <br>
We’ll just need to setup Koin for the iosApp.<br>
First, go back to your Android Studio and in `CommonModule` add 

{% highlight swift %} 
fun initKoin() = initKoin {}
{% endhighlight %} 

Then return to XCode and change your `iosApp.swift` to the following:
{% highlight swift %} 
import SwiftUI
import shared
 
@main
struct KMPizzaApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
 
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
 
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Override point for customization
        CommonModuleKt.doInitKoin()
        return true
    }
}
{% endhighlight %} 
 
 
This piece of code allows us to customize the entry point for the iosApp and lets us define when to start Koin.<br>
Here we create an `AppDelegate` and while doing it, initialize Koin with this line: `CommonModuleKt.doInitKoin()`

Now run the app again.<br>
It may take a while, but you’ll finally see the “Pizza dough” item in our one-item-list.<br>
Yay! We’ve moved one step forward. <br>
<iframe src="https://giphy.com/embed/3oKIPdJxNVK86FzNiE" width="480" height="324" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="https://giphy.com/gifs/papajohns-dancing-pizza-3oKIPdJxNVK86FzNiE"></a></p>
<b>In the next step</b> we’ll come up with a better design and learn to use <b>KMP image libraries</b>.