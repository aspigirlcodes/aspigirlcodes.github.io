---
title: Adding Service Workers to your Django App (part 3/3)
description: Learn how to add service workers to the polls app from the official django tutorial.
header: Adding Service Workers to your Django App (part 3/3)
categories: Programming
---

This is part 3 of a 3-part tutorial about adding service workers to a django web-app. In this part we will implement a caching strategy to show old poll data to the user when they are offline.

* This is part 3: Implementing a caching strategy.
* [part 1: preparation and setup]({% post_url 2018-03-24-django-offline-1 %})
* [part 2: an offline page using a service worker]({% post_url 2018-03-24-django-offline-2 %})

Implementing a Caching Strategy
===============================

## Caching Patterns

Before starting implementing a caching strategy for the polls app, this
subsection is dedicated to give an overview of some common caching
patterns. Your caching strategy will typically include multiple of those
patterns and variations on them, depending on the content.

Cache only:

:   Assets that almost never change, such as certain CSS files and
    background or header images, can just be loaded once during the
    service worker installation, and then always be served from cache.

Network only:

:   This is what would happen without a service worker. It can be
    achieved by simply ignoring the requests fetch event, which then by
    default gets past on to the server.

Cache, fallback to network:

:   In this case, the service worker tries to find the asset in the
    cache first, and if this fails, it attempts to serve it from
    the network. This can be used for a type of assets that you know
    will never change once it is served, but for some reason you cannot
    or do not want to download it yet at the moment the service worker
    is installed.

Network, fallback to cache:

:   Here you will always try to fetch the request from the network
    first, but if this fails, you will return the version from
    the cache.

Cache then network:

:   This approach tries to be fast and up-to-date. It will first serve
    the version from the cache, as this is faster. Then it will try to
    fetch the same content from the network as well, and update it as
    soon as it arrives.

Generic fallback:

:   This is similar to our custom offline page. When the network request
    fails, we will get some generic asset from the cache, which is
    better than just an error. It can also be used for images. If the
    service worker cannot get this specific image, it might return a
    generic image that has been cached during installation. This will
    look more in theme than just failing to load the image.

Although it is not mentioned explicitly, often when getting a response
from the network, it is a good idea to create a new cache entry, or
update the old one.

## Caching the Polls App

For this tutorial, I have chosen the following caching strategy:

-   Static assets will be cached on service worker installation, and
    served from cache, falling back to network.

-   For HTML pages the service worker will attempt to serve them from
    the network first, and cache them. If the network fails, it will try
    to serve them from the cache, and if they are not found in the
    cache, it will serve our custom offline page.

Here is what this looks like in code (note that the install and activate
events remain the same, and are left out of the code snippet):

{% highlight JavaScript %}
var CACHE_NAME = "polls-cache-v3";
var CACHED_URLS = [
  "/polls/offline/",
  "/static/polls/style.css",
  "/static/polls/images/background.png",
  "/static/polls/app.js"
]

... // Install and activate events remain the same

self.addEventListener("fetch", function(event){
  var requestURL = new URL(event.request.url);
  // request for a html file
  if (event.request.headers.get("accept").includes("text/html")){
    event.respondWith( // answer the request with ...
      caches.open(CACHE_NAME).then(function(cache){ //open cache
        return fetch(event.request) // try to fetch from server
          .then(function(networkResponse){ //if suceeds
            if (networkResponse.ok){ //if it is not some errorpage
              //put a copy of the response in the cache
              cache.put(event.request, networkResponse.clone());
            };
            return networkResponse; //and return the response
          }).catch(function(){ // if fetching from server fails
            // return copy from cache or offline page from cache
            return cache.match(event.request).then(function(response){
              return response || cache.match("/polls/offline/");
            });
          });
      })
    );
  //request is one of the resources cached during registration
  } else if(
    CACHED_URLS.includes(requestURL.href) ||
    CACHED_URLS.includes(requestURL.pathname)
  ){
    event.respondWith( // answer the request with ...
      caches.open(CACHE_NAME).then(function(cache){ // open cache
        return cache.match(event.request).then(function(response){
          // return matched result from cache or try to fetch from network
          return response || fetch(event.request);
        });
      })
    );
  };
});
{% endhighlight %}

The cache name is updated to v3. And the file is now also added to the
array of URLs to be cached during installation. The install and activate
event logic remains the same. But the fetch event is updated to fulfill
the new requirements. It exists of two parts, written as an if and an
else if clause. The if clause filters out all requests for HTML files.
Then it opens the cache and tries to fetch those request from the
network. If this succeeds, a response will be returned. A clone of this
response has to be put in the cache, except if it was some kind of error
response, as you do not want to clutter your cache with those. It is
important to clone the response, as each response can only be consumed
once. The response itself is returned to the page. If the network fetch
fails, first the script tries to match the actual request with the cache
entries. Remember that succeeds even when nothing is found. Therefore
the script returns this response or the offline page from the cache. The
second part (the else if clause) takes care of those assets that were
cached during installation. It opens the cache and returns the response
from it. This should normally be there, but if for some reason it isn’t,
we can still try to fetch it from the network. Requests that do not fit
any of those clauses will just pass trough the service worker and be
requested from the server only. Now, run your Django development server
() and test out your new service worker. Be sure not to forget to give
your service worker the chance to be activated before you start testing
its behavior.

## Letting the User Know the Content Might be Outdated

Thanks to our caching strategy users will be able to view some content
even if they are offline. This also means, however, that they might see
outdated content without noticing. Also, the way the polls app functions
right now, users might try to submit a vote to a poll when offline,
which won’t work. From a usability point of view, we should let users
know in advance, rather than letting them find out another way. This
looks more straightforward than it is. The service worker can fetch
responses or craft its own, but it can not easily make changes to
responses it got from the server or the cache. These are read-only. (If
you want to try it anyway, Craig Russell wrote a blogpost about it
@serviceworker.) From the page itself, we have easy access to the
property, however this is implemented differently by different browsers,
and does not always mean the same as the service worker has served this
page from cache. Because of this, the most recommended way to check
whether or not someone is offline is by sending an AJAX request to the
server. When you have a kind of app-shell HTML page, that then uses an
AJAX-request to get from the server, this comes together really nicely.
You serve the shell HTML page from cache. It has some JavaScript code to
(1) set up an eventlistener for messages from te service worker, and (2)
request . The service workers fetch event will probably contain a
separate if clause for the case. It might get the data with a network,
fallback to cache approach with updating the cache. In the case the data
comes from the cache, the service worker can now additionally send a
message to the page that the data has been served from the cache.
However this approach does not work when requesting the whole page at
once, because by the time the eventlistener will be set up and running,
the message has long been issued and gone. For this tutorial, rather
than issuing a useless AJAX request from the page, I chose to send a
message to the service worker. The service worker reacts to this message
by requesting a page from the server, and in case this fails, sends a
message back to the page confirming the offline status. This approach is
not perfect, as the page might have been delivered from the network if
the network issues occurred right in between the page load and the
message, but the result is usable and lets me demonstrate both
directions of page - service worker messaging. There is more to
messaging than this though. A service worker can also send broadcast
messages to all of its pages, and you can set up a channel for
communicating forth and back between a service worker and a page. Those
things are not covered here. First, set up the page side of the
messaging. To do this, add the following code to the file:

{% highlight JavaScript %}
...
if ("serviceWorker" in navigator && navigator.serviceWorker.controller){
  // serviceWorker in navigator does not mean service worker installed.
  // we need controller to be available
  navigator.serviceWorker.addEventListener("message", function(event){
    // when receiving a message
    if (event.data === "offline"){ // if the data is "offline"
      // add an alert to the top of the page
      var newDiv = document.createElement('div')
      var newContent = document.createTextNode("You are currently offline. " +
        "The content of this page may be out of date.");
      newDiv.appendChild(newContent);
      newDiv.classList.add('alert')
      document.body.insertBefore(newDiv, document.body.firstChild);
      // if there is a submit button on the page, dissable button
      var b = document.querySelector("input[type='submit']");
      if (b){
        b.setAttribute("disabled", "");
      };
    };
  });
  // send a message to the service worker
  navigator.serviceWorker.controller.postMessage("offline?")
}
{% endhighlight %}

If service workers are supported and a service worker is installed
(controller is available), this code first sets up an event listener for
message events. As there might at some point be different messages we
check for the message data, and if it fits we create a new div with a
warning message. JQuery is not a dependency for this project, so vanilla
JavaScript is used to do this. Of course you could also use JQuery if
you want to. Add the alert class to the div so that you can style it
later. Also, there is code to disable the submit button if there is one.
This code disables the first submit button only, if you have more than
one submit button, you would have to adapt it. After setting up the
event listener, a message is sent to the service worker. On the service
worker side, we will update the version number and add a new event
listener for messages like so (note that all other event listeners
remain the same and are omitted in the code snippet):

{% highlight JavaScript %}
var CACHE_NAME = "polls-cache-v4";

... // CACHED_URLS and all other event listeners remain the same

self.addEventListener("message", function(event){
  if (event.data === "offline?"){ // check the message data
    fetch("/polls/").catch(function(response){
    // fetch from the network and if it fails
      self.clients.get(event.source.id).then(function(client){
      // get the client that sent this message
        client.postMessage("offline"); // and send a confirmation message
      });
    });
  };
});
{% endhighlight%}

While there is only one service worker, there can be multiple clients.
The client-id of the client sending a message to the service worker is
recorded in . Based on this the service worker can get this client and
post it a message. It does this only when the server is not reachable
though. When the fetch request succeeds, no action is taken. Finally,
you can style the offline warning by adding some rules to your style.css
file:

{% highlight css%}
...

.alert{
  background-color: #F8D7DA;
  color: #7A1C2F;
}
{% endhighlight %}

Time to test! Fire up your Django development server (), let your
service worker install and activate, and browse some pages to cache
them. Then turn of the server, and revisit those pages. You should now
see the warning message and the voting button should be disabled.

> **Note:** Django serves static files with a longer cache header. This
> becomes very annoying when working with service workers. There is no
> such thing as reload service worker while ignoring cache, as there is
> for reload page. So you might have to clear your browser cache
> manually for this to work. This is a good reminder that you will have
> to take care of what cache header your static files will be served
> with in production.

What is Next?
-------------

You now have a pretty good idea of what a service worker is and how you
can use it for caching specific pages of your website. You also got
acquainted with the service worker lifecycle and might have encountered
some service worker pitfalls already. The one thing holding me back from
using service workers for more complex things is that there is still no
easy solution for automated testing of service workers. Lack of browser
support for the newer APIs could be another minor reason, but this one
can be counted for by progressive implementation and will solve itself
eventually. I learned a lot from the book “Building progressive web
apps" by Tal Ater @ater2017building. If you want to look online for
resources, you might want to search for ‘indexedDB’, ‘Background Sync’
and ‘Push Notifications’. Also not covered in this tutorial is the Web
app manifest.

* This was part 3: Implementing a caching strategy.
* [part 1: preparation and setup]({% post_url 2018-03-24-django-offline-1 %})
* [part 2: an offline page using a service worker]({% post_url 2018-03-24-django-offline-2 %})
