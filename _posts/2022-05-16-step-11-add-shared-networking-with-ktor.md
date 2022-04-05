---
title:  "Step 11: Add networking layer to shared KMM module with Ktor" 
layout: post
categories: shared KMM Ktor networking api
--- 

Now that we have a backend for the recipes we’ll use <b><i>KMM power</i> </b>to implement some shared layers, such as networking, local app storage and even viewmodels. 

Let’s start with <b>networking</b>, where we will use all the functionality from our backend.

First, add <b>Ktor Client dependencies</b> to our project. Yes, we’ll be using the same Ktor we used for our backend, but it’s Client functionality this time. 

Add a `Common` object with necessary dependencies to your `Versions.kt`:

{% highlight kotlin %}
object Common {
   const val KTOR_CLIENT_CORE = "io.ktor:ktor-client-core:$KTOR_VERSION"
   const val KTOR_CLIENT_JSON = "io.ktor:ktor-client-json:$KTOR_VERSION"
   const val KTOR_LOGGING = "io.ktor:ktor-client-logging:$KTOR_VERSION"
   const val KTOR_CLIENT_SERIALIZATION = "io.ktor:ktor-client-serialization:$KTOR_VERSION"
}
{% endhighlight %}


Alongside with the client dependency and additional features like `logging` and `serialization` we add engine dependencies for our platform sides:

{% highlight java %}
object Android {
   const val KTOR_CLIENT = "io.ktor:ktor-client-android:$KTOR_VERSION"
   const val KTOR_OKHTTP = "io.ktor:ktor-client-okhttp:$KTOR_VERSION"
} 

object iOS {
   const val KTOR_CLIENT = "io.ktor:ktor-client-ios:$KTOR_VERSION"
}
{% endhighlight %}

Now implement these dependencies in `build.gradle.kts` in your `shared` module <i>(not backend!)</i>. Modify the corresponding sourceSets as follows:

{% highlight kotlin %}
val commonMain by getting {
   dependencies {
       // Ktor
       implementation(Versions.Common.KTOR_CLIENT_CORE)
       implementation(Versions.Common.KTOR_LOGGING)
       implementation(Versions.Common.KTOR_CLIENT_JSON)
       implementation(Versions.Common.KTOR_CLIENT_SERIALIZATION)
   }
}

val androidMain by getting {
   dependencies {
       implementation(Versions.Android.KTOR_CLIENT)
       implementation(Versions.Android.KTOR_OKHTTP)
   }
}

val iosMain by getting {
   dependencies {
       implementation(Versions.iOS.KTOR_CLIENT)
   }
}
{% endhighlight %}


Also add `serialization` plugin to the `shared` module `plugins`, so that we can deserialize data from network responses into data classes:

{% highlight kotlin %}
plugins {
   kotlin("multiplatform")
   kotlin("native.cocoapods")
   id("com.android.library")
   kotlin("plugin.serialization") version Versions.KOTLIN_VERSION
}
{% endhighlight %}

Sync your project files.
Now, let’s create our <b>Ktor client</b>.
First, in the shared module in `commonMain` create a new package `api` and add a `KtorApi` interface: 

<img src="{{site.baseurl}}/assets/images/step-11/1.png" alt="create KtorApi interface" width="350"/>

Here we declare the Ktor client and the methods which we’ll use for all api calls. 

{% highlight kotlin %}
interface KtorApi {
   val client: HttpClient
   fun HttpRequestBuilder.apiUrl(path: String)
   fun HttpRequestBuilder.json()
}
{% endhighlight %}

With `apiUrl` we’ll set the request path and with `json` we’ll tell the client to receive data in form of json.

Then we add a `KtorApiImpl.kt` inext to `KtorApi.kt` and write an implementation:

{% highlight kotlin %}
class KtorApiImpl() : KtorApi {

   val prodUrl = "https://enigmatic-sands-01782.herokuapp.com/" [1]

       override val client = HttpClient {
       install(JsonFeature) { [2]
           serializer = KotlinxSerializer()
       }
   }

   override fun HttpRequestBuilder.apiUrl(path: String) {
       url {
           takeFrom(prodUrl) [3]
           encodedPath = path
       }
   }

   override fun HttpRequestBuilder.json() {
       contentType(ContentType.Application.Json) [4]
   }
}
{% endhighlight %}

[1] For now we hardcoded the backend urls, later we’ll see how to manage them in build configuration<br>
[2] We’ll need to serialize and deserialize our objects to receive and send them to our backend, therefore we use kotlinx serializer feature, that we imported earlier<br>
[3] Specify the url that should be used to fetch data: it consists of the base production url and the following endpoint path<br>
[4] Specify the content type for all requests, here we are using json<br>

Now let’s create our actual `RecipesApi`.
Your `commonMain` should look like this by now:

<img src="{{site.baseurl}}/assets/images/step-11/2.png" alt="create KtorApi interface" width="350"/>

`RecipeApi` will extend the `KtorApi` and contain the recipe endpoint:

{% highlight kotlin %}
class RecipesApi(private val ktorApi: KtorApi) : KtorApi by ktorApi {
   companion object {
       const val RECIPES_BASE_URL = "recipes"
   }
}
{% endhighlight %}

Also add all the necessary requests for our app:

{% highlight kotlin %}
suspend fun getPizza(): String {
   return client.get {
       apiUrl("pizza") [1]
   }
}

suspend fun getRecipes(): List<RecipeResponse> {
   return client.get { [2]
       apiUrl(RECIPES_BASE_URL)
   }
}

suspend fun getRecipe(id: Long): RecipeResponse {
   return client.get {
       apiUrl("$RECIPES_BASE_URL/$id")
   }
}

suspend fun postRecipe(recipe: Recipe): Long [3] {
   return client.post {
       json() [4]
       apiUrl(RECIPES_BASE_URL)
   }
}
{% endhighlight %}

[1] For our test endpoint we don’t use the recipes url: instead we call the `pizza` url and should get a `String` in return<br>
[2] Use the `GET` function to receive data from backend as a List of RecipeResponses<br>
[3] When we post a recipe we  get an ID of type Long in return<br>
[4] Don’t forget to specify the content type for your post request<br>

As you can see, we just mirrored the requests that we already have in our backend api. We also have an option to upload pictures, but we’ll see how to do it after we’re done with basic text requests.

You may be wondering, why `Recipe` and `RecipeRepsonse` are unresolved. Well, we’ve created an entity for our model, but it was in another module, the `backend` module. 

To solve this problem, let’s copy that `Entity.kt` file and put it into a `model` folder in our `shared` module. Now you can resolve the references in `RecipeApi`.

Let’s remove the `model` package from our `backend` module and implement the `shared` project there instead.

To do so, in shared `build.gradle.kts` first add `jvm()` to your targets:
{% highlight kotlin %}
kotlin {
   android()
   jvm()

. . .
}
{% endhighlight %}

Now, implement the shared project in the `backend` `build.gradle.kts` under `dependencies`:
{% highlight kotlin %}
implementation(project(":shared"))
{% endhighlight %}

<b>Sync your project files</b> and fix the Exposed entities in your backend module to import corresponding classes from `kmpizza.model` package.

<i>Now we have our model only in one location, and it’s extremely convenient, isn’t it?</i>

Finally, let’s create a remote data source, which we can later use alongside a local source to get appropriate data.

Create `RecipeRemoteSource` class in `shared/commonMain/remote`.

<img src="{{site.baseurl}}/assets/images/step-11/3.png" alt="create KtorApi interface" width="350"/>

This class will receive `recipesApi` as a parameter and use it to fetch recipes with `getRecipes` from our `RecipesApi`, as well as add a recipe with `postRecipe`.

{% highlight kotlin %}
class RecipeRemoteSource(
   private val recipesApi: RecipesApi
) {

   suspend fun getRecipes() = recipesApi.getRecipes()
   suspend fun postRecipe(recipe: Recipe) = recipesApi.postRecipe(recipe)
}
{% endhighlight %}

Easy as that, with the help of Ktor we’ve just <b>added a networking layer</b> to our shared KMM module. This will be the only location where we implement our api requests. You’ll see how to use them on platform sides in the following steps. However, we’re not done with the shared part. <b>In the next step</b> we’ll look into sharing a <b>viewmodel layer</b>.