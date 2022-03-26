---
title:  "Step 9: Add images to KMP backend using AWS S3 Storage" 
layout: post
categories: database exposed postgres aws s3
--- 

There is one thing that is currently missing in our recipes: <br>mouthwatering pictures of pizza with lots of cheese, like this one:

<img src="https://images.unsplash.com/photo-1588315029754-2dd089d39a1a?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2071&q=80" alt="pizza cheese yummy" width="400"/>

So let’s add pictures to our backend. We’ll use <b>Amazon S3 Storage</b> to upload and store images. 

<i><b>Attention</b>: it requires creating an AWS account and binding in to your credit card, but we’ll only use free features, so no worries about having to pay. If you want to continue without images then just skip this part. </i>

First, we need to register. Go to [AWS portal](https://portal.aws.amazon.com/billing/signup#/) and create an <b>AWS account</b>.<br> Verifying and binding your account to your credit card may take several days, then you’ll be able to start using AWS services.

When you’re verified, go to <b>Amazon Services -> Storage -> S3</b>

<img src="{{site.baseurl}}/assets/images/step-9/1.png" alt="go to aws" width="600"/>

There create your first bucket, where you’ll store pizza pictures.

<img src="{{site.baseurl}}/assets/images/step-9/2.png" alt="create bucket" width="600"/>

<img src="{{site.baseurl}}/assets/images/step-9/3.png" alt="configure bucket" width="600"/>

You can choose an AWS r<b>region</b> for your bucket. To reduce latency choose the closest to you. Also <strong>do not</strong> choose “block public access”.

<img src="{{site.baseurl}}/assets/images/step-9/4.png" alt="cblock public access" width="600"/>

After your create a bucket it should appear in the list like this:

<img src="{{site.baseurl}}/assets/images/step-9/5.png" alt="list of buckets" width="1200"/>

Remember your bucketName and region for future use in the app. In this case they are: `bucketName: kmpizza` and `region: eu-central-1`

We’ll also need an `accessKey` and a `secretKey`.

Go to  [IAM Console](https://console.aws.amazon.com/iamv2/home?#/users).
And add a <b>new user</b> with programmatic access

<img src="{{site.baseurl}}/assets/images/step-9/6.png" alt="create new user" width="700"/>

Add <b>permissions</b> for S3 to this user

<img src="{{site.baseurl}}/assets/images/step-9/7.png" alt="permissions" width="700"/>

In stage 4 it should look like this:

<img src="{{site.baseurl}}/assets/images/step-9/8.png" alt="stage 4" width="700"/>

When your user is created, copy your `secretKey` and `accessKey`
```
accessKey: ******
secretKey: ******
```

Now that we have Amazon Storage all set up and ready, let’s get back to our backend module.

First, add AWS SDK to `Versions.kt`:

{% highlight kotlin %}
const val AWS_VERSION = "2.17.102"

object Jvm {
...
const val AWS_JAVA_SDK = "software.amazon.awssdk:bom:$AWS_VERSION"
const val AWS_S3 = "software.amazon.awssdk:s3"

}
{% endhighlight %}

Then implement it in your backend `build.gradle.kts` dependencies:

{% highlight kotlin %}
implementation(platform(Versions.Jvm.AWS_JAVA_SDK))
implementation(Versions.Jvm.AWS_S3)
{% endhighlight %}

Now let’s create a <b>aws</b> package and put it next to <b>exposed</b> in the storage directory:

<img src="{{site.baseurl}}/assets/images/step-9/9.png" alt="create aws package" width="350"/>

Inside this package create a `FileStorage` interface with a function to save files:
{% highlight kotlin %}
import java.io.File

interface FileStorage {
   suspend fun save(file: File): String
}
{% endhighlight %}

Then create an AmazonFileStorage class that will implement this save function:
{% highlight kotlin %}
class AmazonFileStorage : FileStorage {

   private val client: S3Client

   private val bucketName: String = System.getenv("bucketName") [1]

   init {
       val region = System.getenv("region") [2]
       val accessKey = System.getenv("accessKey")
       val secretKey = System.getenv("secretKey")

       val credentials = AwsBasicCredentials.create(accessKey, secretKey)
       val awsRegion = Region.of(region)
       client = S3Client.builder()
           .credentialsProvider(StaticCredentialsProvider.create(credentials)) [3]
           .region(awsRegion)
           .build() as S3Client
   }

   override suspend fun save(file: File): String = [4]
   withContext(Dispatchers.IO) {
       client.putObject(
               PutObjectRequest.builder().bucket(bucketName).key(file.name).build(),
               RequestBody.fromFile(file)
           ) [5]
       val request = GetUrlRequest.builder().bucket(bucketName).key(file.name).build()
       client.utilities().getUrl(request).toExternalForm()
   }
}
{% endhighlight %}

[1] Get the bucket name from environment variables<br>
[2] Also get other settings to initiate an AWS S3 client: <b>region</b> (optional), <b>access</b> and <b>secret</b> Keys<br>
[3] Use specified settings to build an <b>S3Client</b><br>
[4] Implement the save function from the `FileStorage` Interface <br>
[5] Use the object request to put a file into the AWS S3 bucket and get the source url<br>

We want to use AWS functionality to upload images from the application when saving new recipes, so add `FileStorage` to `LocalSourceImpl`’s constructor like this:

{% highlight kotlin %}
internal class LocalSourceImpl(
   private val fileStorage: FileStorage,
   application: io.ktor.application.Application
) : LocalSource {
. . .
}
{% endhighlight %}

Just like with our local storage we want to use the AWS storage simply by injecting where we need it, so add one line to your `KoinModule.kt` and modify the way we create `LocalSourceImpl` by injecting the Amazon storage into it:

{% highlight kotlin %}
internal fun Application.getKoinModule() = module {
   single<FileStorage> { AmazonFileStorage() }
   single<LocalSource> { LocalSourceImpl(get(), this@getKoinModule) }
}
{% endhighlight %} 

Let’s add a request to save images.<br> First, we’ll extend our recipes with a list of images that will show how delicious it is. Add a `RecipeImage` data class to your `Entity.kt` file
{% highlight kotlin %}
@Serializable
data class RecipeImage(
   val id: Long = 0, 
   val image: String
)
{% endhighlight %} 

We can create recipes with images in two steps:<br> 1) use the previously created Recipe entity to post a recipe to the backend and receive the id of newly created Recipe<br> 2) add pictures to it. 

But to receive recipes with images directly from the backend whenever we need them we’ll introduce a `RecipeReponse` entity: 

{% highlight kotlin %}
 @Serializable
data class RecipeResponse(
   val id: Long = 0,
   val title: String,
   val ingredients: List<Ingredient>,
   val instructions: List<Instruction>,
   val images: List<RecipeImage>
)
{% endhighlight %} 

Now we need to adjust our database. In a new <b>image</b> package create a `RecipeImageTable`:
<img src="{{site.baseurl}}/assets/images/step-9/10.png" alt="create recipe image table" width="250"/>

{% highlight kotlin %}
 @Serializable
object RecipeImageTable : IntIdTable("recipeImages") { 
   val image = varchar("image", 1024)
   val recipe = reference(
       "recipe_id",
       RecipeTable,
       onDelete = ReferenceOption.CASCADE,
       onUpdate = ReferenceOption.CASCADE
   )
}
{% endhighlight %} 

And a corresponding Entity:
{% highlight kotlin %}
class RecipeImageEntity(id: EntityID<Int>) : IntEntity(id) {
   companion object : IntEntityClass<RecipeImageEntity>(RecipeImageTable)
 
   var image by RecipeImageTable.image
   var recipe by RecipeEntity referencedOn RecipeImageTable.recipe
}

fun RecipeImageEntity.toRecipeImage() = RecipeImage(id.value.toLong(), image)
{% endhighlight %} 

Finally, we need to adjust our `RecipeEntity` by adding a `recipeImage` property:

{% highlight kotlin %}
val recipeImages by RecipeImageEntity referrersOn RecipeImageTable.recipe
{% endhighlight %} 

Add changing the converter function accordingly:
{% highlight kotlin %}
fun RecipeEntity.toRecipeResponse() = RecipeResponse(
   id.value.toLong(),
   title,
   ingredients.map { it.toIngredient() },
   instructions.map { it.toInstruction() },
   recipeImages.map { it.toRecipeImage() })
{% endhighlight %} 

Let’s use our newly created entities.<br> In your `LocalSource` interface declare this function:
{% highlight kotlin %}
suspend fun saveImage(recipeId: Long, image: File)
{% endhighlight %} 

Also change the return type of `getRecipes` and `getRecipe`:
{% highlight kotlin %}
suspend fun getRecipes() : List<RecipeResponse>
suspend fun getRecipe(recipeId: Long) : RecipeResponse
{% endhighlight %} 

And in the `LocalSourceImpl` write the following implementations:

{% highlight kotlin %}
override suspend fun saveImage(recipeId: Long, image: File) {
   withContext(dispatcher) {
       val imageUrl = fileStorage.save(image) [1]       
           transaction {
           val recipe = RecipeEntity[recipeId.toInt()] [2]
           RecipeImageEntity.new { [3] 
               this.image = imageUrl
               this.recipe = recipe
           }
       }
   }
}
{% endhighlight %} 
 
[1] Save the image in S3 storage and get the image url in return<br>
[2] Get the recipe from the database that the image should be attributed to<br>
[3] Create a new recipe image entity and save a reference to the recipe <br>

And don’t forget to change the return type of `getRecipes` and `getRecipe`:
{% highlight kotlin %}
override suspend fun getRecipes(): List<RecipeResponse> = withContext(dispatcher) {
   transaction {
       RecipeEntity.all().map { it.toRecipeResponse() }
   }
}

override suspend fun getRecipe(recipeId: Long): RecipeResponse = withContext(dispatcher) {
   transaction {
       RecipeEntity[recipeId.toInt()].toRecipeResponse()
   }
}
{% endhighlight %} 


Also, we have to create a new image recipe table in the `LocalSourceImpl`:

{% highlight kotlin %}
transaction {
   SchemaUtils.createMissingTablesAndColumns(
       RecipeTable,
       IngredientTable,
       InstructionTable,
       RecipeImageTable
   )
}
{% endhighlight %} 


Finally, let’s add the routes to our api. Add this function to `Routing`:

{% highlight kotlin %}
addRecipeImage(localSource) 
{% endhighlight %} 

And then implement it:

{% highlight kotlin %}
private fun Routing.addRecipeImage(localSource: LocalSource) {
   post("/recipes/{recipeId}/recipeImage") {
       val recipeId = call.parameters["recipeId"]!!.toLong()
 
       var image: File? = null

       call.receiveMultipart().forEachPart { [1]
           when (it) { 
               is PartData.FileItem -> image = it.toFile() [2]
               else -> Unit
           }
           it.dispose()
       }
       localSource.saveImage(
           recipeId, 
           image ?: throw BadRequestException("image part is missing")
       )
       image?.delete()[3]
       call.respond(HttpStatusCode.OK)
   }
}
{% endhighlight %} 

[1] We receive the image file from the client side as a <b>multipart form data</b><br>
[2] This will help us transform part data to a file type so that we can save the image data in our database. It is unresolved for now and still has to be implemented in the next step<br>
[3] If an error happens we throw an exception and do not save the image file<br>

As mentioned in [2] we still need to create the `toFile` function that will help us transform the `PartData.FileItem` to a `File`. Create a <b>utils</b> package in our backend package and put an `Extensions.kt` file there with the following function

{% highlight kotlin %}
import io.ktor.http.content.*
import java.io.File
:
fun PartData.FileItem.toFile() = File(originalFileName!!).also { file ->
   streamProvider().use { input ->
       file.outputStream().buffered().use { output ->
           input.copyTo(output)
       }
   }
}
{% endhighlight %} 

Now we can test uploading pictures to the recipes. First <b>rebuild and make</b> the project.
In the <b>terminal</b> set local environment variables according to your AWS settings.

```
export bucketName=kmpizza
export secretKey=******
export accessKey=******
export region=eu-central-1
export JDBC_DATABASE_URL=jdbc:postgresql:recipecollection?user=postgres
``` 

then run the backend locally as many times before:
```
java -jar ./backend/build/libs/Backend.jar -port=9000
```

Check with postman if you still have any recipes in your local database, otherwise add the pizza dough recipe again.
If your database is fresh you get you `recipeId: 1`, otherwise choose the id of the recipe you want to add pictures to.

Now you can add an image to your recipe. Using that id: 
```
http://localhost:9000/recipes/1/recipeImage
```
<img src="{{site.baseurl}}/assets/images/step-9/11.png" alt="post image" width="500"/>

We get status `200 OK` in response. Let’s check if the recipe has an image now with `getRecipes`. Indeed in our recipe we see a list of images now:
```
 "images": [
            {
                "id": 1,
                "image": "https://kmpizza.s3.eu-central-1.amazonaws.com/kmpizza.jpg"
            }
        ]
```
 
Yay! 🥳<br>
We’ve built our very functional backend to store delicious recipes with images. <b>In the next step</b> we’ll see how to publish our backend on Heroku automatically. Soon after that we’ll move on to using the backend we built within the shared code between iOS and Android.

