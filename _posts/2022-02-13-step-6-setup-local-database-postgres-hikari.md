---
title:  "Step 6: Setup a local database with PostgreSQL and HikariCP" 
layout: post
categories: setup postgres hikari introductory database
--- 

By now we’ve prepared everything to setup our local database. We’ll use postgreSQL for it. If you don’t have it installed already, follow the instructions [here](https://devcenter.heroku.com/articles/heroku-postgresql#local-setup).

Then create a new database for our app with the following sequence of commands from the terminal:
```
psql -U postgres
CREATE DATABASE recipecollection;
```

(If you later need to delete the database, use `DROP DATABASE recipecollection;`)

Pay attention to `“;”`! 

Then switch to the newly created database:
```
\c recipecollection
```

You should see the following message:
You are now connected to database `recipecollection` as user `postgres`.

​​Congrats, you are a proud owner of a local database! It’s empty for now, but you don’t need to create the tables manually, as our exposed backend will take care of it with the help of all the tables we’ve already defined in our app.

We’ll use JDBC (Java Database Connectivity) to manage connection to our database and execute requests. A lightweight JDBC library `HikariCP` will help us create a connection pool that can be reused when we make a new request to the database. Let’s add missing dependencies to our app:

{% highlight kotlin %}
Add these to your Versions.kt:
const val POSTGRESQL_VERSION = "42.3.1"
const val HIKARI_VESRION = "5.0.0"

...

object Jvm {
const val HIKARI_CONNECTION_POOL = "com.zaxxer:HikariCP:$HIKARI_VESRION"
const val POSTGRESQL = "org.postgresql:postgresql:$POSTGRESQL_VERSION"
}
{% endhighlight %}
 
And then use the dependencies in the `backend` module:

{% highlight kotlin %}
implementation(Versions.Jvm.POSTGRESQL)
implementation(Versions.Jvm.HIKARI_CONNECTION_POOL)
{% endhighlight %}

Sync your project dependencies.

Now let’s go back to the app and actually create a local source implementation so that we can connect it to our REST api.
First, create a file `LocalSource.kt` in the `exposed` directory and define an interface:

{% highlight kotlin %}
internal interface LocalSource {
}
{% endhighlight %}

Later on, we’ll think about what methods we want to use here to access the data in our database. For now we’re more interested in implementing and configuring it. So let’s create an implementation for our `LocalSource`. Add a `LocalSourceImpl.kt` next to `LocalSource.kt`. Here we’ll use `HikariCP` functionality to configure our database connection and afterwards fill up the database with the tables we specified before within the `Exposed` framework:

{% highlight kotlin %}
   @OptIn(ObsoleteCoroutinesApi::class)
   internal class LocalSourceImpl(application: io.ktor.application.Application) : LocalSource {
      private val dispatcher: CoroutineContext

      init {
         val config = application.environment.config.config("database") [1]
         val url = System.getenv("JDBC_DATABASE_URL") [2]
         val driver = config.property("driver").getString()
         val poolSize = config.property("poolSize").getString().toInt()
         application.log.info("Connecting to db at $url")

         dispatcher = newFixedThreadPoolContext(poolSize, "database-pool") [3]
         val hikariConfig = HikariConfig().apply {
            jdbcUrl = url
            maximumPoolSize = poolSize
            driverClassName = driver
            validate()
         }

         Database.connect(HikariDataSource(hikariConfig)) [4]

         transaction { [5]
            SchemaUtils.createMissingTablesAndColumns(
                  RecipeTable,
                  IngredientTable,
                  InstructionTable
            )
         }

      }
   }
{% endhighlight %}

[1] Here we specify some of the configuration parameters. Therefore we’ll also need to extend our `application.conf` file with the database `driver` and `poolSize`. Add this to `application.conf`:

```
ktor {
 deployment {
   port = 8080
   port = ${?PORT}
 }
 application {
   modules = [dev.tutorial.kmpizza.backend.MainKt.module]
 }
}

database {
 driver = "org.postgresql.Driver"
 poolSize = 20
}
```

[2] We will get the URL for our database from environment variables. To test the database locally we can set this variable in our terminal, which I’ll show you how to do later<br>
[3] Create a coroutine execution context with the fixed-size thread-pool<br>
[4] Connect to our exposed database using Hikari Configuration<br>
[5] Create missing tables from our definitions<br>

Now that we’ve set up a backend and a local database, let’s bind them together.
We’ll use `Koin` dependency injection for it. First, add these to `Versions.kt`:

{% highlight kotlin %}
const val KOIN_VERSION = "3.1.2"

object Jvm {
const val KOIN_KTOR = "io.insert-koin:koin-ktor:$KOIN_VERSION"
…
}
{% endhighlight %}

Add the dependency to your backend `build.gradle.kts`:

{% highlight kotlin %}
implementation(Versions.Jvm.KOIN_KTOR)
{% endhighlight %}

Sync your dependencies.

Now we can create a Koin Module. Add `KoinModule.kt` to a new directory “di” (as in “dependency injection”) in your backend module.

<img src="{{site.baseurl}}/assets/images/step-6/1.png" alt="add koin module" width="300"/>

Create a module with a single local source, adding other necessary imports:
 
{% highlight kotlin %}
import io.ktor.application.*
import org.koin.dsl.module

internal fun Application.getKoinModule() = module {
   single<LocalSource> { LocalSourceImpl(this@getKoinModule) }
}
{% endhighlight %}

Then add Koin to `Application` module in your `Main.kt`:

{% highlight kotlin %}
import org.koin.ktor.ext.Koin

internal fun Application.module() {

   install(Koin) {
       modules(getKoinModule())
   }

   install(Routing) {
       api()
   }
}
{% endhighlight %}

Ok, let’s test how our local source works with the backend. We’ll add one test function to the `localSource`:

{% highlight kotlin %}
internal interface LocalSource {
   suspend fun getPizza(): String
}
{% endhighlight %}

And implement this function in `LocalSourceImpl`:

{% highlight kotlin %}
override suspend fun getPizza(): String = withContext(dispatcher) {
 "Pizza!"
}
{% endhighlight %}

Now let’s change our routing. First, change the `Routing.hello()` function:
{% highlight kotlin %}
private fun Routing.pizza(localSource: LocalSource) {
   get("/pizza") {
       localSource.getPizza().let {
           call.respond(it)
       }
   }
}
{% endhighlight %}

And change `Routing.api()`:
{% highlight kotlin %}
internal fun Routing.api(localSource: LocalSource) {
   pizza(localSource)
}
{% endhighlight %}

To finally connect all together we need to inject local source in our backend application and use it with Routing. Change `Application.module()` to look like this:

{% highlight kotlin %}
internal fun Application.module() {
   install(Koin) {
       modules(getKoinModule())
   }

   val localSource by inject<LocalSource>()

   install(Routing) {
       api(localSource)
   }
}
{% endhighlight %}


Time to test!
First, rebuild and make the project.
Then in the terminal add the local environment variable to your environment: 
```
export JDBC_DATABASE_URL=jdbc:postgresql:recipecollection?user=postgres
```
And then run our backend locally:
```
java -jar ./backend/build/libs/Backend.jar -port=9000
```

Go to `localhost:9000/pizza` and if you get a pizza craving from this request, then we’re on the right track!

<img src="https://images.unsplash.com/photo-1590947132387-155cc02f3212?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=2940&q=80" alt="pizza!" width="500"/> 





