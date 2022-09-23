---
title: "Step 26: Post images to AWS from a KMM Android or iOS app"
layout: post
categories: kmm ios swift ui shared viewmodel compose android ktor aws
--- 

Now we'll add pictures to our recipes. <br>
We already implemented this feature on backend, so we can upload pictures with  `/recipes/{recipeId}/recipeImage` endpoint.
 
If you forgot, this is the method for it:
{% highlight kotlin %} 
private fun Routing.addRecipeImage(localSource: LocalSource) {...}
{% endhighlight %} 
 
Let’s see how to implement it in the `shared` networking and then use it in Android and iOS apps.
 
Go to the shared `commonMain/util` folder and add `File.kt`.<br>
Declare our expected `ImageFile` and a helper function to convert it to `ByteArray`:

{% highlight kotlin %} 
expect class ImageFile
expect fun ImageFile.toByteArray(): ByteArray
{% endhighlight %} 
 
Then just like before write actual functions for `File.kt`
 
<b>androidMain:</b>
{% highlight kotlin %} 
import android.content.ContentResolver
import android.net.Uri
 
actual typealias ImageFile = ImageUri [1]
 
actual fun ImageFile.toByteArray() = contentResolver.openInputStream(uri)?.use { [2]
   it.readBytes()
} ?: throw IllegalStateException("Couldn't open inputStream $uri")
 
class ImageUri(val uri: Uri, val contentResolver: ContentResolver)
 
fun Uri.toImageFile(contentResolver: ContentResolver): ImageFile { [3]
   return ImageFile(this, contentResolver)
}
{% endhighlight %} 
 
<b>[1]</b> Here we set a `typealias` for our expect class `ImageFile`, which in case of Android is `ImageUri`. <br>
<b>[2]</b> Transform `ImageFile`s to `ByteArray` using `contentResolver`.<br>
<b>[3]</b> Also use `contentResolver` functionality to view images.<br>
 
Similarly for iOS we add `File.kt` in `common/iosMain`

<b>iosMain:</b>
{% highlight kotlin %} 
actual typealias ImageFile = UIImage
 
actual fun ImageFile.toByteArray() = UIImagePNGRepresentation(this)?.toByteArray() ?: emptyArray<Byte>().toByteArray()
 
@OptIn(ExperimentalUnsignedTypes::class)
private fun NSData.toByteArray(): ByteArray = ByteArray(length.toInt()).apply {
   usePinned {
       memcpy(it.addressOf(0), bytes, length)
   }
}
{% endhighlight %} 
 
You may see a warning, saying `Expected function 'toByteArray' has no actual declaration in module kmpizza.shared for JVM`. <br>
Suppress this warning with the following annotation to the `expect `functions:
{% highlight kotlin %} 
@Suppress("NO_ACTUAL_FOR_EXPECT")
{% endhighlight %} 
 
Finally, we need to think of how to send the local image from the user’s gallery to the backend. <br> Extend the `RecipeUiModel`, so that it allows manipulations with `ImageFile`s. <br>To start with, add a `localImage` parameter:

{% highlight kotlin %} 
data class RecipeUiModel (
   val title: String,
   val ingredients: List<Ingredient> = listOf(),
   val instructions: List<Instruction> = listOf(),
   val images: List<RecipeImage> = listOf(),
   val localImage: ImageFile? = null,
)
{% endhighlight %} 
 
Go to `RecipesApi` and add the following network request:
{% highlight kotlin %} 
suspend fun postImage(recipeId: Long, icon: ImageFile): Unit = client.submitFormWithBinaryData( [1]
   formData {
       appendInput(key = IMAGE_FILE_PART, headers = Headers.build { [2]
           append(HttpHeaders.ContentDisposition, "filename=${recipeId}_image") [3]
       }) {
           buildPacket { writeFully(icon.toByteArray()) }
       }
   }) {
   apiUrl("$RECIPES_BASE_URL/${recipeId}/recipeImage") [4]
}
   .body()
{% endhighlight %} 
 
 
<b>[1]</b> Create the form data request <br>
<b>[2]</b> Add `const val IMAGE_FILE_PART = "image"` to `companion object` <br>
<b>[3]</b> For now we’re just uploading one image per recipe, therefore we pass no name to the function and save the image just with a generic name relevant to recipeId <br>
<b>[4]</b> use the `recipesApi` to post the `formData`, just like we did using Postman <br>
 
Now add this to `RecipeRemoteSource`:
{% highlight kotlin %} 
suspend fun postImage(recipeId: Long, imageFile: ImageFile) = recipesApi.postImage(recipeId, imageFile)
{% endhighlight %} 
 
Also adjust the `RecipeRepository`:
{% highlight kotlin %} 
suspend fun postRecipeImage(recipeId: Long, imageFile: ImageFile) = recipeRemoteSource.postImage(recipeId, imageFile)
{% endhighlight %} 
 
Finally, when uploading the recipe, right after we send the recipe data, we’ll attach an image to the recipe. <br> Go to `RecipeDetailsViewModel` and change `saveRecipe()` to:
{% highlight kotlin %} 
fun saveRecipe() {
   coroutineScope.launch {
           recipe.value?.let { recipe ->
               if (recipe.title.isNotEmpty() && recipe.ingredients.isNotEmpty() && recipe.instructions.isNotEmpty()){ 
                   val result = recipeRepository.postRecipe(recipe)
                   val imageUploadRequest = recipe.localImage.let { image ->
                       async { [1]
                           result?.let { it ->
                               if (image != null) {
                                   recipeRepository.postRecipeImage(it, image)
                               }
                           }
                       }
                   }
                   imageUploadRequest.await() [2]
                   result?.let { _upload.value = true } [3]
               }
           }
   }
}
{% endhighlight %} 
 
<b>[1]</b> If the recipe was uploaded successfully, post the recipe `Image` asynchronously<br>
<b>[2]</b> Wait for the result of image upload<br>
<b>[3]</b> If it was uploaded successfully, set the upload state value to true<br>
 
Now let’s go back to our apps and adjust the UIs, starting with Android.
In `RecipeDetailsScreen` before your `Previews` add:

{% highlight kotlin %} 
private fun ActivityResultLauncher<String>.launchAsImageResult() = launch("image/*") [1]
 
@Composable
fun registerForGalleryResult(callback: (ImageFile) -> Unit) =
   (LocalContext.current as AppCompatActivity).contentResolver.let { contentResolver -> [2]
       rememberLauncherForActivityResult(ActivityResultContracts.GetContent()) { uri: Uri? ->
           uri?.toImageFile(contentResolver)?.let(callback) [3]
       }
   }

{% endhighlight %} 
 
<b>[1]</b> Start the choose image from Gallery activity in Android <br>
<b>[2]</b> Using `contentResolver`, get the chosen image<br>
<b>[3]</b> Using our actual function, transform the local image to `ImageFile`, which can be sent to the backend<br>
 
Above your `Scaffold` in `RecipeDetailsScreen` add a state value that will trigger the `onImageChanged` in the `ViewModel`:
{% highlight kotlin %} 
val getImage = registerForGalleryResult(viewModel::onImageChanged)
{% endhighlight %} 
 
In the shared `RecipeDetailsViewModel` add `onImageChanged` to the `EditRecipeChangeListener`:
{% highlight kotlin %}  
interface EditRecipeChangeListener {
   fun onTitleChanged(title: String)
   fun onIngredientsChanged(ingredient: Ingredient)
   fun onInstructionsChanged(instruction: Instruction)
   fun onImageChanged(image: ImageFile)
}
{% endhighlight %} 

And implement it in `RecipeDetailsViewModel`:
{% highlight kotlin %} 
override fun onImageChanged(image: ImageFile) {
   _recipe.value = _recipe.value?.copy(localImage = image)
}
{% endhighlight %} 
 
Now let’s go back to `RecipeDetailsScreen`. Here we can finally use `getImage`. First add it to `PlaceholderImage` signature. Also add an `isEditable` boolean to switch on the gallery functionality:
{% highlight kotlin %}  
private fun PlaceholderImage(
    padding: PaddingValues,
    getImage: ActivityResultLauncher<String>,
    localImage: ImageFile?,
    isEditable: Boolean
)
{% endhighlight %} 
 
And modify the `PlaceholderImage`, so that it changes from placeholder image to the chosen image from your library:
{% highlight kotlin %} 
@Composable
private fun PlaceholderImage(
   padding: PaddingValues,
   getImage: ActivityResultLauncher<String>,
   localImage: ImageFile?,
   isEditable: Boolean
) {
   val placeholder = "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg"
   Box(contentAlignment = Alignment.Center) {
      HeaderImage(image = localImage?.uri ?: placeholder, padding = padding)
       if (isEditable) {
           IconButton(
               modifier = Modifier
                   .clip(CircleShape)
                   .background(Color.Cyan),
               onClick = { getImage.launchAsImageResult() }
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
}
{% endhighlight %} 
 
Also adjust the `HeaderImage` composable:
{% highlight kotlin %}  
private fun HeaderImage(image: Any, padding: PaddingValues)
{% endhighlight %} 
 
And use it in `Scaffold`:
{% highlight kotlin %}  
recipe?.images?.let {
   if (it.isNotEmpty()) {
       HeaderImage(it[0].image, padding)
   } else {
       PlaceholderImage(padding, getImage, recipe?.localImage, recipeId == null)
   }
}
{% endhighlight %} 
 
Build and run the app. You’ll see an error. <br>

`java.lang.ClassCastException: dev.tutorial.kmpizza.android.MainActivity cannot be cast to androidx.appcompat.app.AppCompatActivity`
 
That’s because we need to make `MainActivity` an `AppCompatActivity` to use `ActivityResultLauncher`. <br>
Make `MainActivity` extend `AppCompatAcitivity` instead of `ActivityComponent`:

{% highlight kotlin %}  
MainActivity : AppCompatActivity()
{% endhighlight %} 
 
Now if you run it again and try adding a new recipe, you’ll be able to choose an image from your gallery.
 
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-25/1.gif" alt="pizza on android" width="300"/></div>

Now let’s do the same in our iOS project.
 
We can add an `ImagePicker`, which will help us to choose images from your phone library. You can read more about the image picker implementation <a href="https://www.hackingwithswift.com/books/ios-swiftui/importing-an-image-into-swiftui-using-uiimagepickercontroller">here</a>
 
To the `utils` group in the iOS project add a new file `ImagePicker`:

{% highlight swift %}  
import SwiftUI
 
struct ImagePickerIOS: UIViewControllerRepresentable {
    @Environment(\.presentationMode) var presentationMode
    var onImageSelected: ((UIImage) -> ())
 
    func makeUIViewController(context: UIViewControllerRepresentableContext<ImagePickerIOS>) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.delegate = context.coordinator
        return picker
    }
 
    func updateUIViewController(_ uiViewController: UIImagePickerController, context: UIViewControllerRepresentableContext<ImagePickerIOS>) {
 
    }
 
    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }
 
    class Coordinator: NSObject, UINavigationControllerDelegate, UIImagePickerControllerDelegate {
        let parent: ImagePickerIOS
 
        init(_ parent: ImagePickerIOS) {
            self.parent = parent
        }
 
        func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]) {
            if let uiImage = info[.originalImage] as? UIImage {
                parent.onImageSelected(uiImage)
            }
 
            parent.presentationMode.wrappedValue.dismiss()
        }
    }
}
{% endhighlight %} 
 
And the adjust the code in `RecipePlaceholderView`:

{% highlight swift %}   
struct RecipePlaceholderView: View {
    @Binding var image: UIImage? [1]
    @State private var showingImagePicker = false [2]
    
    var body: some View {
        ZStack{
            if let image = image {
                Image(uiImage: image) [3]
                    .resizable()
                    .frame(width: 200, height: 150)
            } else {
                KFImage(URL(string: "https://m.media-amazon.com/images/I/413qxEF0QPL._AC_.jpg")) [4]
                    .resizable()
                    .frame(width: 200, height: 150)
            }
            
            Image(systemName: "camera.circle.fill") [5]
                .font(.system(size: 28, weight: .light))
                .foregroundColor(Color.black)
                .frame(width: 50, height: 50)
                .background(Color.accentColor)
                .clipShape(Circle())
            
        }
        .sheet(isPresented: $showingImagePicker){ [6]
            ImagePickerIOS(onImageSelected: {
                self.image = $0
            })
        }
        .onTapGesture { [7]
            showingImagePicker.toggle()
        }
    }
}
{% endhighlight %}  
 
 
<b>[1]</b> First add a Binding to hold the UIImage <br>
<b>[2]</b> Also a state variable to signal whether the imagePicker button should be visible or not<br>
<b>[3]</b> In the view body if there an image to show, then show it<br>
<b>[4]</b> Otherwise, show a placeholder image<br>
<b>[5]</b> Put an “Add image” button on top<br>
<b>[6]</b> With the showingImagePicker state variable control whether the imagePicker should show<br>
<b>[7]</b> And toggle the showingImagePicker by tapping the image button, thus opening the image picker<br>
 
Also change the relevant code in `RecipeDetailView` body to:
{% highlight swift %}  
  if (state.recipe?.images.isEmpty == false){
                KFImage(URL(string: state.recipe?.images.first?.image ?? "")) [1]
                    .resizable()
                    .frame(width: 200, height: 150)
            } else {
                RecipePlaceholderView(image: Binding(
                    get: { state.recipe?.localImage}, [2]
                    set: {
                        if let image = $0 {
                            state.viewModel.onImageChanged(image: image) [3]
                        }
                    }
                )).padding(16)
            }
{% endhighlight %}   
 
<b>[1]</b> If there is an image in th recipe, show it<br>
<b>[2]</b> Otherwise when the recipe is edited, show a placeholder image view with an add button layover<br>
<b>[3]</b> And set the image in the view model once it’s changed in the app<br>
 
Build and run the app.
<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-26/2.gif" alt="pizza on ios" width="300"/></div>
 
Now we can upload images from the phone!
<b>In the next step</b> we’ll learn how to use a local database as a back up for when there’s no connection.