---
layout: post
title:  "Models and Managers Part II: Custom Managers in Django"
date:   2016-12-03 15:08:00
author: Ryan Castner
categories: Django
tags: django models business logic
cover:  "/assets/header_image_default.jpg"
custom_js_highlight:
- django_highlights
---

If you haven't already, you can [read](https://audiolion.github.io/2016/12/03/models-and-managers-part-i.html) the first part of this series.

# Models and Managers Part II: Custom Managers in Django

Custom managers in Django are a topic I feel people never properly understand or readily utilize in their programming. Django has created a beautifully extensible system for providing custom methods to generate QuerySet's so that we don't have to repeat ourselves (DRY). Furthermore this system can allow for more optimized queries than the general hacks you will find on StackOverflow when you are googling to figure out how to solve a problem where you want to filter a queryset based on some specific settings. Finally, usage of custom managers allows us to place our business logic at the lowest level, in the SQL queries themselves which provides us the greatest optimization and Django's ORM allows us to do it in a DRY manner that can be tested and verified.

We will bring back our Model from Part I:

```
from django.utils import timezone

class Course(models.Model):
    title = models.CharField()
    start = models.DateTimeField()
    end = models.DateTimeField()

    def in_session(self):
        now = timezone.now()
        if self.start < now and self.end > now:
            return True
        return False

    def not_started(self):
        now = timezone.now()
        if self.start > now:
            return True
        return False

    def ended(self):
        now = timezone.now()
        if self.end < now:
            return True
        return False
```

We had a nice `DetailView` that allowed us to use these methods and render an appropriate message to our user.

```
class CourseDetailView(DetailView):
    model = Course
    template = "<appname>/course_detail.html"

    def get_context_data(self, **kwargs):
        context = super(CourseDetailView, self).get_context_data(**kwargs)
        course = context['course']
        if course.in_session():
            context['info_msg'] = 'This course is currently in session.'
        elif course.not_started():
            context['info_msg'] = 'This course has not started yet.'
        elif course.ended():
            context['info_msg'] = 'This course has ended.'
        return context
```

The problem now is, lets say we want a page that displays all upcoming courses (not started yet) to our user. How can we use our model methods to achieve this? When we use Django's ORM to return a `QuerySet` of objects we have a lot of power to control the generated SQL in an easy, safe, and clear manner, however, the ORM cannot utilize model methods like `not_started()`. If you were approaching this problem and googling around you might come across these [commonly](http://stackoverflow.com/a/2276826/5657142) [referenced](http://stackoverflow.com/a/5685046/5657142) solutions on StackOverflow involving a list comprehension and generator statement.

```
Class UpcomingCourseListView(ListView):
    model = Course
    tempalte = "<appname>/upcoming_course_list.html"

    def get_queryset(self):
        queryset = Course.objects.all()
        ids = [course.id for course in queryset if course.not_started()]
        queryset = queryset.filter(id__in=ids)
        return queryset
```

We override the `get_queryset()` method of Django's `ListView`, first retrieving all `Course` objects. Then we generate a list of ids by iterating through each item in the `queryset` and calling the `Course` model object's `not_started()` method, returning the `course.id` if the method returns `True`. We then `filter()` the queryset by iterating through it again, only keeping the `id`'s that match our list of `ids`. As you might guess, this is pretty inefficient. We have our initial query which iterates through `O(n)`, we then do another `O(n)` iteration to generate the `ids` list and a final `O(n)` iteration through the `queryset` where each `object` does a `O(n)` comparison through the list of `ids`.

The biggest performance issue here is that we have to hit the database multiple times to do this processing. We first need to touch it to get the entire `Course` table and then again to go through the same table and grabbing only what we want.

How can we do something more efficient? Well, quite simply the `not_started()` model method we want to call can be implemented in a SQL Query.

Let's take a gander at what that might look like.

```
from django.utils import timezone

class UpcomingCourseListView(ListView):
    model = Course
    tempalte = "<appname>/upcoming_course_list.html"

    def get_queryset(self):
        now = timezone.now()
        queryset = Course.objects.filter(start__gt=now)
        return queryset
```

We just eliminated all of those extra iterations and are performing the business logic at the database level where we have the best case optimization scenario. We don't have to store objects in memory of our python method, hit the database multiple times, nor use our web server's processing power (we offload computation to the database server).

However we have our logic again coupled with the `views.py`, it is not clear at a glance to another programmer reading the code what the intention of the overrided `get_queryset()` is for nor that it corresponds to the model method's `not_started()` logic.

## Custom Managers to the Rescue!

We are all familiar with querying through models by calling `ModelName.objects...`. When we instantiate a Django model (passing `models.Model` to the class) it assigns a `objects = models.Manager()` by default to the model, providing query support for the model. It does this entirely through introspection and is quite a marvelous piece of engineering. We can piggy back off that great code and augment it with business logic methods. The `Manager` has an associated `QuerySet` that is uses to do the querying, a nice decoupling by Djangoo. What we want to do here is implement the business logic methods at the `QuerySet` level and incorpate them into our custom `Manager` and assign that manager to our `Course` model.

```
from django.utils import timezone


class CourseQuerySet(models.QuerySet):

    def not_started(self):
        now = timezone.now()
        return self.filter(start__gt=now)

    def in_session(self):
        now = timezone.now()
        return self.filter(start__lte=now, end__gte=now)

    def ended(self):
        now = timezone.now()
        return self.filter(end__lt=now)


class CourseManager(models.Manager):

    def get_queryset(self):
        return CourseQuerySet(self.model, using=self._db)

    def not_started(self):
        return self.get_queryset().not_started()

    def in_session(self):
        return self.get_queryset().in_session()

    def ended(self):
        return self.get_queryset().ended()


class Course(models.Model):
    title = models.CharField()
    start = models.DateTimeField()
    end = models.DateTimeField()

    objects = CourseManager()

    def in_session(self):
        now = timezone.now()
        if self.start < now and self.end > now:
            return True
        return False

    def not_started(self):
        now = timezone.now()
        if self.start > now:
            return True
        return False

    def ended(self):
        now = timezone.now()
        if self.end < now:
            return True
        return False
```

We have just implemented these business logic methods at the `QuerySet` level, as close to the database we can get as it translates directly to SQL. In addition, with this implementation the methods are chainable, while chaining `not_started()`, `in_session()` and `ended()` might not make sense, suppose we had a `publish_date` method for the course.

```
from django.utils import timezone


class CourseQuerySet(models.QuerySet):

    def not_started(self):
        now = timezone.now()
        return self.filter(start__gt=now)

    def in_session(self):
        now = timezone.now()
        return self.filter(start__lte=now, end__gte=now)

    def published(self):
        now = timezone.now()
        return self.filter(publish_date__lte=now)


class CourseManager(models.Manager):

    def get_queryset(self):
        return CourseQuerySet(self.model, using=self._db)

    def not_started(self):
        return self.get_queryset().not_started()

    def in_session(self):
        return self.get_queryset().in_session()

    def published(self):
        return self.get_queryset().published()


class Course(models.Model):
    title = models.CharField()
    start = models.DateTimeField()
    end = models.DateTimeField()
    publish_date = models.DateTimeField()

    objects = CourseManager()

    def in_session(self):
        now = timezone.now()
        if self.start < now and self.end > now:
            return True
        return False

    def not_started(self):
        now = timezone.now()
        if self.start > now:
            return True
        return False

    def ended(self):
        now = timezone.now()
        if self.end < now:
            return True
        return False
```

We could now make a call like this to get all published courses that have not started yet:

`Course.objects.not_started().published()`

Let's update our `UpcomingCourseListView` with the new code.

```
from django.utils import timezone


class UpcomingCourseListView(ListView):
    model = Course
    tempalte = "<appname>/upcoming_course_list.html"

    def get_queryset(self):
        return Course.objects.not_started().published()
```

Now it is very clear to another programmer reading through this code, you overrode the `get_queryset()` method so that the list of `Course` objects is `not_started()` and `published()`. Exactly what we want to provide to a student. We don't have to rewrite the SQL logic anywhere, we just rely on our written once custom `QuerySet` and `Manager` methods, reducing the amount of testing needed to verify the business logic.

## Epilogue

There may be an outcry, what if the business logic is really complex? My answer is that, Django, most likely, has you covered. Django's ORM provides immense power with SQL statements with functions from `django.db.models` like [Case](https://docs.djangoproject.com/en/1.10/ref/models/conditional-expressions/#case), [When](https://docs.djangoproject.com/en/1.10/ref/models/conditional-expressions/#when), and [F](https://docs.djangoproject.com/en/1.10/ref/models/expressions/#f-expressions).

When you find yourself repeating queries and see the opportunity to abstract them into business logic rules, save yourself and the next developer some trouble and use a custom queryset and manager.

To be fully transparent, the one downside of this approach is that you can end up with monolithic models, where your `models.py` becomes huge. Remember, some queries are simple enough or one-off's that you don't need to implement a custom queryset method for them. You can always separate your custom querysets and managers into a `querysets.py` and `managers.py` file if you would like as well, this helps break down the code into more manageable chunks with better separation of concerns.
