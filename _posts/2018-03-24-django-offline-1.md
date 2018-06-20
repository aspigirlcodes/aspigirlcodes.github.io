---
title: Adding Service Workers to your Django App (part 1/3)
description: Learn how to add service workers to the polls app from the official django tutorial.
header: Adding Service Workers to your Django App (part 1/3)
categories: Programming
---


In this 3 part tutorial you will learn how to add offline functionality to a
Django web-app (the one from the official django tutorial) using a service worker.

I created this tutorial as part of my bachelor thesis, and didn't change much to it before publishing it here. Hence the somewhat long explanations and formal language in some places.

* This is part 1: preparation and setup.
* [part 2: an offline page using a service worker]({% post_url 2018-03-24-django-offline-2 %})
* [part 3: implementing a caching strategy]({% post_url 2018-03-24-django-offline-3 %})

## Introduction

### What is a service worker?
A service worker is a script
written in JavaScript that is run by the browser. However, instead of
running inside your page like your normal client-side code, it runs in
the background. From this position, the service worker can interact both
with the pages it controls, and the server (although it does not have
direct access to any single pages DOM). In particular, a service worker
can intercept requests from a page to the server and do something with
them. It could cache certain responses and return them from the cache
when another request matching those responses comes in, but it can also
send different requests, or read the responses before returning them
back to the page.

The unique possibilities of service workers make them
very powerful. To fully leverage this power, a whole ecosystem of APIs
is being build around service workers, including data storage,
background sync, and push notifications. Those allow websites to evolve
from a simple web page to a full fledged progressive web app that can do
everything a native app can.

### What will we do in this tutorial?
Despite the possibilities, the aim of this
tutorial is not to build a progressive web app with lots of bells and
whistles. That would definitely be too much to cover in one go. Things
like background sync, push messages and IndexedDB (a client-side data
storage) will not be covered here.

Instead the tutorial will focus on
the basics of adding a simple service worker to an existing Django
project and discuss things like caching strategies and service worker
life cycle.

On a practical level this tutorial consists of 3 parts. In the first part we go over the Prerequisites and prepare the project for our tutorial.
Then we will create our first service worker and use it to create a simple offline page.
In the third part we will take it further and implement a caching strategy, that will allow us to show the user an older version of the poll data while they are offline.
If you want to go further after that, you will find some useful
resources in the short what’s next section at the end of this tutorial.

> **Note:** Most of this tutorial is inspired by [Raphael Merx’s talk on
> offline Django with Service workers at Pycon Australia 2017](https://www.youtube.com/watch?v=70L8saIq3uo)
> and [Tal Ater’s book Building Progressive Web Apps](http://shop.oreilly.com/product/0636920052067.do).

Prerequisites
-------------

The starting point of this tutorial is the end of the official Django
tutorial (<https://docs.Djangoproject.com/en/1.11/intro/tutorial01/>). A
repository with the codebase to start from is provided. Download
instructions are further down in this section. The Django tutorial is
written for Django 1.11 and Python 3. It consists of 7 parts in which
you learn how to create a simple poll app.

The app is based on two data models. The question model has a publication
date and a question text.
The choice model on the other hand, has a foreign key relationship with
question. This means that one question can relate to multiple choices,
but each choice can only belong to one question. Choices have a choice
text and keep track of the number of times the choice has been chosen.

There are also three main views: one displays a list of questions, the
next displays one question with its choices in a form for voting, and
the last one shows the result of the votes for one question. There are
no custom views for creating and publishing questions. Instead, the
Django admin is used for this.

This tutorial, however, will only add
offline functionality to the custom views, and leave the Django admin
untouched. If you don’t want to do the Django tutorial (again), the
final state of the Django tutorial, and thus the beginning state of this
tutorial is tagged ‘start’ in the following GitHub repository: <https://github.com/aspigirlcodes/django-serviceworker-tutorial.git> . This
means you can access it as follows:

{% highlight shell-session %}
$ git clone https://github.com/aspigirlcodes/django-serviceworker-tutorial.git
$ cd django-serviceworker-tutorial/
$ git checkout tags/start -b mytutorial
{% endhighlight %}

> **Note:** You still have to create a virtual environment and install
> Django 1.11 after cloning the repository.
>
> ```shell-session
>     $ python3 -m venv service_worker
>     $ source service_worker/bin/activate
>     $ pip install Django==1.11
> ```
>
> Then set up your project by doing the initial migrations and creating
> a superuser
>
> ```shell-session
>     $ python manage.py migrate
>     $ django-admin createsuperuser
> ```

If you have finished the Django tutorial, you will have enough knowledge
of Python and Django to follow along with this tutorial. As service
workers are implemented in JavaScript, some experience with JavaScript
is useful as well. However, you do not need to be a JavaScript expert.
If you have followed a beginner’s tutorial about JavaScript or added
some JavaScript to a web page by Google, trial and error, you should be
good to go. One thing you might want to have a look at if you are not
familiar with them already are JavaScript promises, as they are used a
lot in service workers.

You will also need a browser with developer
tools for service workers. The browsers Firefox and Chrome both have
this.

Preparation: Improve Django Templates to Output a Propper HTML Page
-------------------------------------------------------------------

The Django tutorial sticks to the bare minimum when it comes to the
templates HTML content. Html, head and body tags are not included. This
is perfectly acceptable for a back end framework introduction tutorial,
but now that you will start working on the front end more, it is a good
time to fix this first.

First, create a new file named `base.html` in `polls/templates/polls` , and put the
following content in it:

{% highlight htmldjango %}
{% raw %}
{% load static %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1,
      shrink-to-fit=no">
    <title>{% block title %}{% endblock title %}</title>

    <link rel="stylesheet" type="text/css"
      href="{% static "polls/style.css" %}" />
  </head>
  <body>
    {% block content %}
    {% endblock content %}
    {% block scripts %}
    {% endblock scripts %}
  </body>
</html>
{% endraw %}
{% endhighlight %}

Then, change the three existing templates. Note that you can delete the
stylesheet link in `index.html`, as it is included in the above `base.html` template.

{% highlight htmldjango %}
{% raw %}
{% extends "polls/base.html" %}

{% block title %} Polls List {% endblock title %}
{% block content %}
  {% if latest_question_list %}
    ...
  {% endif %}
{% endblock content %}
{% endraw %}
{% endhighlight %}

{% highlight htmldjango %}
{% raw %}
{% extends "polls/base.html" %}

{% block title %} Answer poll {% endblock title %}

{% block content %}
  <h1>...
  ...
  </form>
{% endblock content %}
{% endraw %}
{% endhighlight %}

{% highlight htmldjango %}
{% raw %}
{% extends "polls/base.html" %}

{% block title %} Poll results {% endblock title %}

{% block content %}
  <h1>...
  ...
  ....</a>
{% endblock content %}
{% endraw %}
{% endhighlight %}

Although this is not absolutely necessary, you can also add the
following extra rule to `style.css` so that the html and body elements fill the
whole page. By this, the image, set to be in the lower right corner of
the body element, will be at the bottom of the page, no matter how much
content the page has.

{% highlight css %}
html, body{
  height:100% !important;
}
...
{% endhighlight %}

That was it for the preparation step! You are now ready to begin working on
our first service worker script.

* This was part 1: preparation and setup.
* [part 2: an offline page using a service worker]({% post_url 2018-03-24-django-offline-2 %})
* [part 3: implementing a caching strategy]({% post_url 2018-03-24-django-offline-3 %})
