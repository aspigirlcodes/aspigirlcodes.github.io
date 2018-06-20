---
title: Adding Service Workers to your Django App (part 2/3)
description: Learn how to add service workers to the polls app from the official django tutorial.
header: Adding Service Workers to your Django App (part 2/3)
categories: Programming
---

This is part 2 of a 3-part tutorial about adding service workers to a django web-app. In this part we will register our first service worker and use it to display an offline page.

* This is part 2: an offline page using a service worker.
* [part 1: preparation and setup]({% post_url 2018-03-24-django-offline-1 %})
* [part 3: implementing a caching strategy]({% post_url 2018-03-24-django-offline-3 %})


## Registering Your First Service Worker

The service worker logic will live in a separate JavaScript file called
`serviceworker.js`. The location of this file as seen by the browser is important for the
scope the service worker will have. The maximum scope of the service
worker is limited by the fact that it can only control files that are
below the `serviceworker.js` file in the URL path hierarchy.

JavaScript files are static
files and are traditionally served under the `/static` path in your Django
project. However if you would serve your file on the path, the scope
would also be limited to `/static`, and there is not a single HTML page there.
Therefore, the easiest way to serve your `serviceworker.js` file is by using a Django view.

In this project, you can put this view in the polls app. It will then be
served under `/polls/serviceworker.js` and its scope will be the polls apps custom views.

In a
real project you might want to serve your service worker in the
namespace without URL prefix. It will then be served at `/serviceworker.js` and will have
your whole project as its scope instead of just one app.

Alternatively,
you can also have several service workers, each with their own scopes.
Scopes should not overlap though, as in that case only the most specific
service worker will apply (and the newest one if they have exactly the
same scope).

To serve the serviceworker as a view, first create a new view in your polls app’s `views.py`:

{% highlight python %}
...

class ServiceWorker(generic.TemplateView):
    template_name = "polls/serviceworker.js"
    content_type = "application/JavaScript"
{% endhighlight %}

Then add the the URL to the polls app’s `urls.py`:

{% highlight python %}
...

urlpatterns = [
    ...
    url(r'^serviceworker.js$', views.ServiceWorker.as_view())
]
{% endhighlight %}

Finally, add the service worker itself. Create `polls/templates/polls/serviceworker.js` and add the following
content to it:

{% highlight JavaScript %}
self.addEventListener("fetch", function(event){
  console.log("Fetch request for:", event.request.url);
});
{% endhighlight %}

This is a simple service worker that listens to fetch events, and logs
them.

You now have created a service worker and Django is serving it at `/polls/serviceworker.js`. But you have not told the browser yet that you want to register the
file at `/polls/serviceworker.js` as a service worker. This should be done from inside a
JavaScript script in a page the user is visiting.

So you will have to
add another JavaScript file containing the service worker registration
and include it in your templates. As this file is just a normal
JavaScript file loaded directly in the page, and not a service worker
you can add it to the static files.

Create a file called `app.js` in `polls/static/polls`, and add
the following content to it:

{% highlight JavaScript %}
if ("serviceWorker" in navigator){ // Check if browser supports service workers
  navigator.serviceWorker.register("/polls/serviceworker.js")
  //register returns a promise
    .then(function(registration){ // if promise resolved
      console.log("Service Worker registered with scope:", registration.scope);
    }).catch(function(err){ // if promise rejected
      console.log("Service worker registration failed:", err);
    });
}
{% endhighlight %}

Then add the following line to the scripts block in your `base.html` file:

{% highlight htmldjango %}
{% raw %}
...
{% block scripts %}
  <script src="{% static 'polls/app.js' %}"></script>
{% endblock scripts %}
...
{% endraw %}
{% endhighlight %}

If you now run your Django development server (`python manage.py runserver`), and visit one of the
pages from your polls app, for example `localhost:8000/polls/`, your
service worker will be registered.

You can check this by looking at the
browser logs using the developer tools of your browser. When you
subsequently click around within the polls app by opening questions and
casting votes, you should see that each request is logged by your
service worker.

> **Note:** If you do not have any questions in your local database yet,
> use your superuser credential to go to the Django admin and create
> some.

## Letting Your Service Worker Serve an Offline HTML Page

Now that you have registered your service worker, you are now ready to use it for serving a customized offline page to your users.

First,
create an offline page in your polls app. To do this, you need to make
the following changes:

Add a new file called `offline.html` in `polls/templates/polls`:

{% highlight htmldjango %}
{% raw %}
{% extends "polls/base.html" %}

{% block title %} Polls {% endblock title %}

{% block content %}
  <h1>You are offline</h1>
  <p>
    We are sorry that you are experiencing connectivity issues
    and hope you come back soon to share your opinion with our polls.
  </p>
{% endblock content %}
{% block scripts %}
{% endblock %}
{% endraw %}
{% endhighlight %}

Add code for the offline view to `polls/views.py`:

{% highlight python %}
...
class OfflineView(generic.TemplateView):
    template_name = "polls/offline.html"
{% endhighlight %}

And add the URL to `polls/urls.py`:

{% highlight python %}
...
urlpatterns = [
    ...
    url(r'^offline/$', views.OfflineView.as_view(), name="offline")
]
{% endhighlight %}

Next, in order to be able to show the user this custom offline page that
you have defined on your server, you will first have to store it in the
users browser. A good time to do this is when installing your service
worker. This way you can be sure that this page is available in the
cache when your service worker becomes active after installation.

Remove
the existing code from your `serviceworker.js` file and instead add the following:

{% highlight JavaScript %}
self.addEventListener("install", function(event){
  event.waitUntil(// dont' let install event finish until this succeeds
    caches.open("polls-cache")// open new or existing cache, returns a promise
      .then(function(cache){// if cache-opening promise resolves
        return cache.add("/polls/offline/")// add offline page
    })
  );
});
{% endhighlight %}

This adds an event listener for the install event. The install event
happens after the registration and before the activation of your service
worker. (In the next section I will explain the service worker life cycle and
which events happen when in more detail.)

You want the install event
to succeed only when it manages to cache the offline page. If not the
install event should fail, and reattempted later. This is achieved by
using `event.waitUntil(function)`. The function passed to `waitUntil` should return a promise. Only if this
promise resolves will the install event be allowed to end.

`caches.open` either opens
an existing cache with the name you pass on to it, or, if no cache with
this name exists, it creates a new one. The opened cache is returned in
a promise. So after this promise resolves, we can be sure to have a
usable cache instance, and add our offline page to it. Resources are
stored in the cache with their path as a key. Old entries with the same
key will be overwritten.

Now that you have put your offline page in the
cache, you can retrieve it when fetching content from the server fails.
To do this, add the following to your `serviceworker.js` file:

{% highlight JavaScript %}
self.addEventListener("fetch", function(event){
  event.respondWith(// answer the request with ...
    fetch(event.request) // try to fetch from server
      .catch(function(){ // if this is rejected
        return caches.match("/polls/offline/") // return offline page from cache
      })
  );
});
{% endhighlight %}

> **Note:** The promise that is returned by caches.match always
> resolves, even if no match was found. In this case this does not
> really matter. We know we loaded the page during the install phase,
> and if for some reason it would not be there anymore, we would not be
> able to get it from the server anyway as we already know we are
> offline. In other situations you might want to check for it though:
>
> ```JavaScript
>     caches.match("request/path").then(function(response){
>       if (response){
>         return response;
>       }
>     });
>     
> ```

Testing the new service worker is easiest when you delete the old one
manually.

In Firefox first go to the list of service workers by typing `about:debugging\#workers`
in your address bar or by choosing `WebDeveloper>ServiceWorker` from the menu.

In Chrome the list of
service workers can be found under `chrome://serviceworker-internals.`. Then click unregister next to your
`localhost:8000/polls` service worker.

Another way to make sure the
service worker gets replaced is by closing all the tabs related to the
polls app, and waiting until the service worker stops running. Only now,
open a polls page in a new tab again for the register event to be
triggered and installing the new service worker version. The following
section on service worker life cycle will help you understand better as
to why this is necessary.

Then run your Django development server (`python manage.py runserver`) and
visit the page to let your new service worker install. In the shell
where you are running the Django server, you should see that a get
request for the `/polls/offline/page` is made. Then quit the development server (`ctrl-c`) and reload
the page in your browser. You should now see your custom offline page.

## Intermezzo: Service Worker Life Cycle

What makes service workers slightly more complicated than normal
JavaScript run in a webpage is the fact that they are not bound to a
single page they can live and die with. They can control multiple pages
in different browser windows or tabs, and can even react to certain
events without needing any open tab or window at all.

To handle this
complexity correctly service workers have well defined life cycles with
different states. You have already encountered the installation state,
triggered by registering the service worker, and the active state, in
which the service worker can listen to fetch events. But installed does
not necessarily mean active. Before a newly installed service worker can
become active, the browser needs to make sure that the old one is done
with its work. So as long as there are pages controlled by the old
service worker open, the new one will have to stay in the waiting state.

![Service worker’s life cycle<span
data-label="fig:sw-lifecycle"></span>](sw-lifecycle){width="80.00000%"}

Figure \[fig:sw-lifecycle\] shows a diagram of the service worker life
cycle.

The first state is the **installing** state. It is triggered by
the register function, but only if the service worker that is asking to
be registered is different from the currently active one. So you can be
sure that one and the same service worker is only installed once, even
if the register logic is hit multiple times. You can add your own logic
to the installing phase by listening to the “install" event and using
waitUntil() to keep the service worker in that state until your logic
either succeeds or fails. If it fails, your service worker will go to
the redundant state. Only when the register function is called again
(for example by refreshing the page) will their be another install event
and another chance to get your service worker active.

After successfull
installation, the service worker comes in a **waiting** state. If no
active service worker is for the same scope is detected, it can move
straight on to the next state. If there is an active service worker
however, it has to wait here, until the active service worker is no
longer controlling any pages and can be turned of and moved to the
redundant state.

> **Note:** Refreshing a page is not enough to make a service worker
> redundant and start using a new one. During refresh, the service
> worker will remain in control. The only way to stop it is to navigate
> to a page outside of its scope, or to close the tab it is controlling.

The service worker now enters the **activating** state. This is the
place where you can do things that have to be done before the new
service worker becomes active that might have a negative impact on an
old service worker that might have been still running during the
installing phase. To add such logic to your service worker listen to the
“activate" event. Again you can use waitUntil() to avoid that the
service worker moves on to the active state already and starts receiving
fetch events.

After the activating process has finished, the service
worker moves to the **active** state. In this state it can react to
various events such as “fetch", “sync", “push" and “message".

The final
state of the service worker is **redundant**. Service workers will land
in this state when registration (for example: the specified service
worker path does not exist) or installation (the promise passed to the
“install" events waitUntil method rejects) fail. Or when they are
replaced by a new service worker.

## Caching Multiple Assets and Cleaning up Old Caches

You might have noticed that the poll’s background image is not loaded as
part of your offline page. That is because you have only cached the HTML
page and not the additional assets such as CSS files and images. You
will have to make to more changes to the `serviceworker.js` file to change this.

First
change the install event logic in `serviceworker.js` like so:

{% highlight JavaScript %}
var CACHE_NAME = "polls-cache-v2";
var CACHED_URLS = [
  "/polls/offline/",
  "/static/polls/style.css",
  "/static/polls/images/background.png"
]

self.addEventListener("install", function(event){
  event.waitUntil(// dont' let install event finish until this succeeds
    caches.open(CACHE_NAME)// open new or existing cache, returns a promise
      .then(function(cache){// if cache-opening promise resolves
        return cache.addAll(CACHED_URLS)// add CACHED_URLS
    })
  );
});
...
{% endhighlight %}

Instead of `cache.add` the service worker now uses `cache.addAll` to add a list of URLs to the
cache.

Second, you have to update the fetch event logic to return the
cached assets, and not just the offline HTML page, as it is doing now.
This can be done as follows:

{% highlight JavaScript %}
...
self.addEventListener("fetch", function(event){
  event.respondWith(// answer the request with ...
    fetch(event.request) // try to fetch from server
      .catch(function(){ // if this is rejected
        return caches.match(event.request) // try to return from cache
          .then(function(response){ // caches.match allways resolves
            if (response) { // check if something was found in cache
              return response;
            } else if
              (event.request.headers.get("accept").includes("text/html")){
              // requesting a html file and not finding a match in cache
              return caches.match("/polls/offline/")
            }
          });
      })
  );
});
{% endhighlight %}

To test, remove the old service worker manually using the developer
tools. Then run your Django development server(`python manage.py runserver`), and load a polls app
page to register and install the new service worker. Quit the Django
development server and reload the page in your browser. Does your
offline page now have the background image?

Two questions remain to be
answered. Why did I tell you to change the cache name by adding a v2
tag to it? And, as a new cache was created instead of using the old one,
what happens to the old caches?

There are a couple of reasons for
versioning your cache. First, while the browser does detect changes to
the `serviceworker.js` file automatically and reinstalls it, if we would make changes to
any of the cached assets, this will not be seen by the browser. By
changing the cache name each time you change something, you can make
sure a new service worker is installed and your cached assets are
updated. On top of that, the new assets will be downloaded in a new
cache. This is good, because caching is done during the installation
phase. This means the old service worker, that might rely on the old
assets, is still active.

> **Note:** While it is considered good practice to version your cache
> by changing the cache name with each version, you should try not to
> change your service worker URL as this can cause unwanted side
> effects. For example, suppose you have an active that is active and
> caches rigorously. If you would change it to , you would then also
> have to change the argument of the register function, which might be
> part of the cached assets. How are you ever going to get this update
> live? It is better to keep the service worker URL the same, and let
> the browser take care of updating your service worker when something
> has changed.

You do, however, need to take care of deleting old caches. Browser
storage is limited, and although the actual limits vary based on various
circumstances, it is a good habit to clean up after yourself from the
start. Cleaning up the cache is typically done in the activation phase
of the service worker life cycle, as at this point you can be sure that
the old service worker is no longer needing it.

Add the following code
to your `serviceworker.js` file:

{% highlight JavaScript %}
self.addEventListener("activate", function(event){
  event.waitUntil( // do not finish activation until this succeeds
    caches.keys() // a promise containing an array of all cache names
      .then(function(cacheNames){
        return Promise.all(
        // return a promise that only succeeds
        // if all of the promises in the array succeed
          cacheNames.map(function(cacheName){
          // to create an array of promises, map the array of cachenames
            // for each of them:
            if (CACHE_NAME !== cacheName ){
            // if name is not the same as current caches name
              return caches.delete(cacheName);
              // delete it and succeeds after that
            };
          })
        );
      })
  );
});
{% endhighlight %}

This code listens to the “activate" event. It first gets an array of
cache names. This array is then transformed to an array of promises, one
for each cache we want to delete, succeeding exactly then when the cache
is deleted. This array is passed to `Promise.all()`, which only succeeds if all
elements of the array succeed.

> **Note:** Depending on your application you might need a more
> complicated if clause. For example if you are using the cache storage
> at other places in your app, or if you use multiple caches at once.

You have now enhanced your Django web app with a custom offline page
using a service worker. That is nice, but you can do more than that with
service workers. In the next section you will implement a more complete
caching strategy.

* This was part 2: an offline page using a service worker.
* [part 1: preparation and setup]({% post_url 2018-03-24-django-offline-1 %})
* [part 3: implementing a caching strategy]({% post_url 2018-03-24-django-offline-3 %})
