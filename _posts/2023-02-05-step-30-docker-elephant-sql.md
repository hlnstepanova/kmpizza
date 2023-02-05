---
title: "Step 30: Store data with ElephantSQL and containerize Ktor backend application with Docker"
layout: post
categories: kmm docker postgresql elephantsql
--- 

As Heroku cancelled its free plans and sharing knowledge has suddenly become expensive, I’ve decided to move KMPizza Ktor backend somewhere else. To do so I also had to learn how to <b>containerize my backend app with Docker</b>.

Another change was database related. Even though many PaS services offer a Postgres plugin, it usually comes with extra expenses, so I decided to host the <b>PostgresSQL database separately</b>. I went with [ElephantSQL](https://www.elephantsql.com), as they have very attractive offers for small projects. And they have a free plan for such tiny ones as KMPizza. But the best thing about ElephantSQl is that you don’t have to enter your credit card data if you only need a free plan.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-30/0.png" alt="elephantsql plans" width="700"/></div>

<h1>Connecting to ElephantSQL</h1>

To start with, go to [ElephantSQL](https://www.elephantsql.com).<br>
Sign up, create a new team and then click on <b>Create a New Instance</b>.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-30/1.png" alt="create a new instance button" width="200"/></div>

Then go on and create a <b>Tiny Turtle Database</b> instance.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-30/2.png" alt="select tiny turtle plan" width="700"/></div>

It takes less than a minute. <br> Voilà, you have the <b>connection data</b>:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-30/3.png" alt="connection data" width="700"/></div>

Here you see the most important pieces: user `DB_USER`, default database name `DB_NAME`, password `DB_PASS` and the URL which you’ll partly use as your connection string.

And you can see the new instance in the list of all your projects:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-30/4.png" alt="database list" width="850"/></div>

Now let’s get back to our app and see what we need to change in the code.

First, in `application.conf` extend the database configuration with the following properties:

{% highlight kotlin %} 
user = "postgres"
user = ${?DB_USER}
password = "password"
password = ${?DB_PASS}
db_name = "recipecollection"
db_name = ${?DB_NAME}
{% endhighlight %} 


These properties will be set from the environment variables later. In the past we used to set the whole `JDBC_DATABASE_URL` string from the environment, but I’ve dicided it’d be nicer to split it into several parts and concatenate into a connection string after.

Hence, adjust the `init {}` in your <b>LocalSourceImpl.kt</b> to use the new connection data:
{% highlight kotlin %} 
init {
   val config = application.environment.config.config("database")
   val dbUser = config.property("user").getString() [1]
   val dbPassword = config.property("password").getString() [2]
   val dbName = config.property("db_name").getString() [3]
   val instanceConnection = config.property("instance_connection").getString()
   val url = "jdbc:postgresql://snuffleupagus.db.elephantsql.com/$dbName" [4]

…

   dispatcher = newFixedThreadPoolContext(poolSize, "database-pool")
   val hikariConfig = HikariConfig().apply {
       jdbcUrl = url
       maximumPoolSize = poolSize
       driverClassName = driver
       username = dbUser [5]
       password = dbPassword [6]
       validate()
   }

…

}
{% endhighlight %} 

<b>[1], [2], [3]</b> Extract the database user, password and database name from the environment variables <br>
<b>[4]</b> Use the new database url to connect <br>
<b>[5], [6]</b> Use the username and password to authenticate <br>

Now proceed to the next step before deploying the backend: packaging it into a docker container.

<h1>Using Docker</h1>

Follow the [instructions](https://docs.docker.com/get-docker/) to install desktop <b>Docker</b> application on your computer. It’s rather uncomplicated if you carefully follow all the steps and check that your computer meets all the system requirements.

Now return to KMPizza. Create a <b>new file</b> in the KMPizza <b>project root</b> called <b>Dockerfile</b>:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-30/5.png" alt="dockerfile" width="250"/></div>

Copy the following configuration to the <b>Dockerfile</b>:

{% highlight kotlin %} 
FROM openjdk:11
EXPOSE 8080:8080
RUN mkdir /app
COPY ./backend/build/libs/Backend.jar /app/Backend.jar
ENTRYPOINT ["java","-jar","/app/Backend.jar"]
{% endhighlight %} 

This file basically tells Docker to build an image from our shadow jar file in `./backend/build/libs/Backend.jar` and expose `port 8080` for the access.

Now  we need to create a new <b>shadow jar</b> package for the backend, so that we can <b>containerize</b> it. Just like in <b>Step 4</b> we simply need to rebuild the project and look for a new jar file in `/backend/build/libs/`. 

However, currently I’m running  into an `IR lowering error` trying to rebuild the project. But here’s a <b><i>temporary workaround</i></b> for it. 

As the error comes from Koin usage in <b>CommonModule.kt</b> we can replace

{% highlight kotlin %} 
fun initKoin(appDeclaration: KoinAppDeclaration) = startKoin {
    appDeclaration()
    modules(
        apiModule,
        repositoryModule,
        viewModelModule,
        platformModule,
        coreModule
    )
{% endhighlight %} 

with 

{% highlight kotlin %} 
fun initKoin(appDeclaration: KoinAppDeclaration) =  {}
{% endhighlight %} 

Now make the project again and the new <b>Backend.jar</b> will appear in the <b>build/libs</b> folder.<br>
May be there’s a better solution to this problem - reach out to me if you have an idea.<br>
<b><i>Don’t forget to reverse your changes to CommonModule.kt afterwards!</i></b>

Now it’s time to containerize this Backend.jar according to the `Dockerfile`.<br>
Open your command line in the KMPizza project root.<br>
Run the following command:

`docker build -t kmpizza . `

This tell the Docker to create and image using the <b>Dockerfile</b> from the current directory. The `-t` parameter tells it to tag the image “kmpizza”. You should be able to see your new image in the Docker Desktop:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-30/6.png" alt="docker desktop image" width="850"/></div>


Finally, we can <b>set the local environment variables</b> from the commands line just like  before:

```
export bucketName=kmpizza
export secretKey=your_AWS_secret_key
export accessKey=your_AWS_access_key
export region=eu-central-1
export DB_NAME=your_ElephantSQL_db_name
export DB_USER=your_ElephantSQL_user
export DB_PASS=your_ElephantSQL_password
```

Now tell Docker to <b>use the variables</b> from local environment when you <b>test run the container locally</b>:

`docker container run --env accessKey --env secretKey --env region --env bucketName --env DB_NAME --env DB_USER --env DB_PASS -p8080:8080 kmpizza`

<b>[1]</b> Here with `–env` parameter you pass every <b>secret</b> to Docker from the environment variables. <br>
<b>[2]</b> With `-p8080:8080` you specify the <b>ports</b> you’ve exposed in the Dockerfile. <br>
<b>[3]</b> And `kmpizza` stands for the application <b>tag</b> you used when building an image. <br>

As usually, go to `localhost:8080/pizza` and verify it. Alternatively, with Postman you can try sending a recipe to your new database with the `/recipes` endpoint and see that it’s still working.

Our Ktor server <b>runs locally</b> now and is connected to <b>ElephantSQL</b> cloud database.<br>
In the next step we’ll see how to <b>deploy it somewhere else</b> 😉