---
title:  "Step 8: Deploy KMP backend on Heroku" 
layout: post
categories: database exposed postgres heroku
--- 

The backend works locally and it’s already a big step forward, but how can we deploy it?<br>
We’ll use Heroku for it.<br>
If you don’t have a Heroku account you first need to [sign up](https://signup.heroku.com/signup/dc).<br>
You’ll also need <strong>Heroku CLI</strong> installed, here are the [instructions](https://devcenter.heroku.com/articles/heroku-cli).

To make sure that the appropriate <strong>Java Runtime version</strong> is used while deploying on Heroku we need one more configuration . Add `system.properties` file to the root of your project with the following lines:
```
java.runtime.version=11
rootProject.name = "backend"
```

After you’ve registered and installed Heroku CLI, you should be able to <strong>login</strong> from the terminal with the following command:
```
heroku login
```
And then <strong>create</strong> a new application with
```
heroku create 
```

You’ll get the `id` of your newly created app in return, like `enigmatic-sands-01782`

<img src="{{site.baseurl}}/assets/images/step-8/1.png" alt="create heroku app" width="700"/>

Then you will need to install a Heroku deploy plugin:
```
heroku plugins:install heroku-cli-deploy
```
Now <strong>deploy</strong> the jar created by `shadowJar` under the name of our Heroku app with this command. Don't forget to change the app name.
```
heroku deploy:jar ./backend/build/libs/Backend.jar --app enigmatic-sands-01782
```

It’ll take some time to upload the build

<img src="{{site.baseurl}}/assets/images/step-8/2.png" alt="create heroku app" width="400"/>

After a while you’ll see the upload succeeded:

<img src="{{site.baseurl}}/assets/images/step-8/3.png" alt="create heroku app" width="600"/>

Finally, <strong>add</strong> your remote heroku git source: 
```
git remote add heroku git@heroku.com:enigmatic-sands-01782.git 
```

In the future, every time you make changes you’ll need to update you remote like this:
```
git push heroku master
```

Try opening [https://enigmatic-sands-01782.herokuapp.com/pizza](https://enigmatic-sands-01782.herokuapp.com/pizza)<br>
<i>Don’t forget to replace the Heroku app name with your own.</i><br>
You’ll see an error. <br>
Try investigating with `heroku logs --tail`, you'll find:<br>

```
Caused by: org.koin.core.error.InstanceCreationException: Could not create
 instance for [Singleton:'com.example.recipecollection.backend.localSource
 LocalSource']
```
or
```
Caused by: java.lang.IllegalArgumentException: jdbcUrl is required with driverClassName.
```

It happens because we are missing a <strong>postgres database plugin</strong> which comes for <strong>free</strong> with Heroku. Let’s add it with
```
heroku addons:create heroku-postgresql
```

You’ll see something like this:

<img src="{{site.baseurl}}/assets/images/step-8/4.png" alt="create heroku app" width="500"/>

This has created your database url and should have automatically set the `JDBC_DATABASE_URL` environment variable in heroku. To check it, you can either use 
```
heroku config
```
or
```
heroku run echo \$JDBC_DATABASE_URL
```

Let's check out our app again with
```
heroku open
```
No errors.<br>
Go to [https://enigmatic-sands-01782.herokuapp.com/pizza](https://enigmatic-sands-01782.herokuapp.com/pizza).<br>
Awesome, now we can see our application serving pizza from Heroku!

<img src="https://i.ibb.co/93rWkkL/20220308-145818.gif" alt="pizza!"/> 





