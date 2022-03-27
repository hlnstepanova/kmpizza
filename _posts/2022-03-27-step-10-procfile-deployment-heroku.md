---
title:  "Step 10 (bonus): Use Procfile for deployment on Heroku" 
layout: post
categories: database heroku backend
--- 
 
So far we've tested our backend with images locally, now let’s move it to Heroku! But this time we’ll make things easier for us by deploying it <i>directly on Heroku</i> (without having to build the jar in Android Studio and deploying it manually after with `heroku deploy:jar`).

First, add `Procfile` to the project `root`.

<img src="{{site.baseurl}}/assets/images/step-10/1.png" alt="create procfile" width="200"/>

Write the following inside the `Procfile`:
```
web: java -jar ./backend/build/libs/Backend.jar
```

It will tell Heroku to start a <b>web dyno</b> and deploy our `backend.jar` file.

If you haven’t done so yet, add `system.properties` to the project root as well.

It should contain the following settings, which will specify which module to build and what Java runtime to use:
```
java.runtime.version=11
rootProject.name = "backend"
```

Git `add` and `commit` both files: make sure `system.properties` is committed, because it can be ignored!

Then do git `push`.

Now run the following command in terminal: 
```
heroku config:set GRADLE_TASK=":backend:build"
```

<i>===== side note =====</i><br>
(If you want to manually build a jar file again, you’ll need to run
`heroku buildpacks:clear --remote <branch-name>`)

If you are deploying the heroku project on a new branch, don’t forget to specify it with `–remote` postfix and also remember to  install a postgres plugin there, for example like this:
```
heroku addons:create heroku-postgresql --remote <branch-name>
```
<i>===== side note =====</i>

We need to set a `buildpack` for Heroku. <br> We’re using gradle, so we have to run the following command in terminal:
```
heroku buildpacks:set heroku/gradle
```
Finally, run to create a new release using this buildpack.
```
git push heroku
```

You will see a build failure 😱 <br>
Looks like we’ve forgotten something important. <br>
If you look at the build logs, you’ll notice this error: 
```
* What went wrong:
remote:        A problem occurred configuring project ':buildSrc'.
remote:        > Could not resolve all artifacts for configuration ':buildSrc:classpath'.
remote:        > Could not resolve org.jetbrains.kotlin:kotlin-stdlib-jdk8:1.5.21.
```

Let’s add the following line to our dependencies in <b>backend</b> `gradle.build.kts`:
```
implementation(kotlin("stdlib"))
```

Run the `git push heroku` command again.

It will take some time for the project to build, but finally you should see something like this:
<img src="{{site.baseurl}}/assets/images/step-10/2.png" alt="heroku build success" width="600"/>

Before serving your pizza directly from Heroku, you will need to set the necessary environment variables like this:
```
heroku config:set accessKey="MyAccessKeyForAWS"
heroku config:set secretKey="MySecretKeyForAWS"
heroku config:set bucketName="kmpizza"
heroku config:set region="eu-central-1"
``` 

Now, you can go to [https://<b><i>your-heroku-app-name</i></b>.herokuapp.com/pizza](https://enigmatic-sands-01782.herokuapp.com/pizza) and get a greeting from your backend, comfortably deployed on Heroku.

 If you are hungry enough, then you can even try and add a pizza recipe with an image using Postman, just like we did previously when testing it locally!

<i>Hey, we're done with the backend for now!</i> 🎉<br> We’ve build our basic backend for the app, so let’s <b>move on to the world of shared KMM code</b>!

In the next steps we’ll be using this backend while building the shared part of our future iOS and Android apps.