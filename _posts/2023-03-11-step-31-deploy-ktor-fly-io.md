---
title: "Step 31: Deploy Ktor backend on Fly.io"
layout: post
categories: kmm docker ktor fly.io
--- 

"Where should I move my hobby rpject from Heroku?"  After researching the alternatives, I’ve settled with <b>Fly.io</b>. It looks like a fairly easy (compared to Google Cloud Platform) option and has good plans for hobby projects. 

Also I love their UI. <br>
Isn’t this art and the haikus inspiring?<br>
It took me 5 minutes to log in just because I was refreshing the page for more haikus 😀

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/0.png" alt="fly haiku" width="300"/></div>

The best thing for your small pet project is that every Fly.io plan has a free allowance, which (according to documentation) includes:

 - Up to 3 shared-cpu-1x 256mb VMs
 - 3GB persistent volume storage (total) 
 - 160GB outbound data transfer 

Additional resources are billed at the usage-based pricing (see [documentation](https://fly.io/docs/about/pricing/)).

To <b>deploy KMPizza backend</b>x on Fly.io, we’ll need the <b>docker image</b> from the previous step.

The steps to get started are well described [here](https://fly.io/docs/hands-on/).<br>
I’ll summarize what I’ve done on my Mac:

First <b>install flyctl</b> tool:
`brew install flyctl`

Then <b>sign up</b>
`flyctl auth signup`

I found that signing up with GitHub would be the most convenient for me.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/1.png" alt="sign up" width="300"/></div>

You can “Try Fly.io for free”, but you’ll still need to add you credit card data afterwards, so I advise you to do it right away.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/2.png" alt="get started" width="400"/></div>

After signing up and logging in you <b>create a new Fly App</b>.<br>
Fly.io is perfect for deploying <b>docker images</b>. That’s why in [the previous step](https://hlnstepanova.github.io/kmpizza/step-30-docker-elephant-sql/) you learned how to use Docker containers.

Go back to KMPizza project. To deploy on Fly your app needs a <b>fly.toml</b> file with instructions how to deploy your app. In command line go to the <b>root</b> of the project and create this file with:

`fly launch`

You’ll see something like:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/3.png" alt="fly launch" width="700"/></div>

In the next step, as you already have a [postgres database separately with ElephantSQL](https://hlnstepanova.github.io/kmpizza/step-30-docker-elephant-sql/) reply no:

{% highlight kotlin %} 
? Would you like to set up a Postgresql database now? No
? Would you like to set up an Upstash Redis database now? No
{% endhighlight %} 

Also reply no to whether you want to deploy right now:

{% highlight kotlin %} 
? Would you like to deploy now? (y/N) N
{% endhighlight %} 

Finally in the command line you’ll see 

```
Your app is ready! Deploy with flyctl deploy
```

After these steps in the project root you’ll have a <b>fly.toml</b> file with a default configuration to deploy your app, something like this:

{% highlight kotlin %} 
app = "kmpizza"
kill_signal = "SIGINT"
kill_timeout = 5
processes = []

[env]

[experimental]
 auto_rollback = true

[[services]]
 http_checks = []
 internal_port = 8080
 processes = ["app"]
 protocol = "tcp"
 script_checks = []
 [services.concurrency]
   hard_limit = 25
   soft_limit = 20
   type = "connections"

 [[services.ports]]
   force_https = true
   handlers = ["http"]
   port = 80

 [[services.ports]]
   handlers = ["tls", "http"]
   port = 443

 [[services.tcp_checks]]
   grace_period = "1s"
   interval = "15s"
   restart_limit = 0
   timeout = "2s"
{% endhighlight %} 

As you can see, it has the basic information for deployment on Fly.io: the app's name, used ports, concurrent connection limits etc.

Now <b>build your Docker image locally</b> (if you haven’t done it in the previous step yet) and <b>deploy the image on Fly</b> with:

`fly deploy  --local-only`

In the end you’ll see something like:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/4.png" alt="deploy locally" width="700"/></div>

But the deployment fails. Ooops.

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/5.png" alt="failed deployment" width="760"/></div>

If you take a look at the logs you’ll see our old grumpy friend, the NullPointerException:

`Caused by: java.lang.NullPointerException: getenv("bucketName") must not be null`

To <b>set your secrets for Fly.io</b> use all the environment variables you need for the deployment (just like every time before) in the following command:

`flyctl secrets set SECRET_ONE=secret1 SECRET_TWO=secret2`

For KMPizza it’s something like

`flyctl secrets set bucketName=kmpizza secretKey=MYVERYSECRETKEY accessKey=MYVERYSECRETKEY region=eu-central-1 DB_NAME=ElephantDb DB_USER=ElephantUser DB_PASS=ElephantPassword`

Run `fly deploy --local-only` again:

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/6.png" alt="success" width="760"/></div>

Yay! It worked.

Check out your app with
`flyctl status`

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/7.png" alt="fly status" width="800"/></div>

One of the most useful pieces of information here is your hostname: kmpizza.fly.dev.<br>
You can also check it in the user friendly Fly web UI, and browse through the app overview, set secrets, manage certificates and billing.<br>

<div style="text-align: center"><img src="{{site.baseurl}}/assets/images/step-31/8.png" alt="web app UI" width="800"/></div>

To <b>visit your app</b> either run `flyctl open` or type https://kmpizza.fly.dev/pizza in a browser.<br>
You’ll see an initial page with KMPizza greeting you.

Finally go back to your KMPizza project in Android Studio and in <b>KtorApiImpl.kt</b> use the <b>new backend url</b>:

{% highlight kotlin %} 
val prodUrl = "https://kmpizza.fly.dev/"
{% endhighlight %} 

Now I’m hungry and I’ll go grab a slice of pizza.