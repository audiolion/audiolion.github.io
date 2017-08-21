---
layout: post
title:  "Models and Managers Part I: Where business logic goes in Django"
date:   2016-12-03 15:08:00
author: Ryan Castner
categories: Django
tags: django models business logic
cover:  "/assets/header_image_default.jpg"
custom_js_highlight:
- django_highlights
---

# Models and Managers Part I: Where business logic goes in Django

I want to talk about custom managers in Django, but before I can do that I want to make sure the reader has a firm grasp of what is possible in terms of implementing business logic in Django and what the best practices are for where to put that logic. If you are already familiar with the models.py / views.py pattern in Django and understand how to decouple logic from views and templates then please move on to [Models and Managers Part II: Custom Managers in Django](https://audiolion.github.io/django/2016/12/03/models-and-managers-part-ii.html). If you are unsure or want a refresher, please read on!


# Models setup

Let's set up a simple model to illustrate custom managers.

```
class Course(models.Model):
    title = models.CharField()
    start = models.DateTimeField()
    end = models.DateTimeField()
```

We have a Course model that represents a school class. The class spans some date range, hence it having a `start` and `end`, this might be a 15 week semester range, or a 10 week quarter, or a year long thesis, it's flexible. The Course would naturally have other models that would pair with it. For the purposes of this series, we are going to keep things simple and just use this model definition.

Now we may want to know in our views some information about the Course:
- Has the Course started?
- Has it ended?
- Is it in session?

## Answer: Django Templates

To accomplish this in Django we have a couple of options, at the highest level this logic could be included through a Template Tag that is processed by Django's Template Language (or Jinja2 Templates) to define this logic. Imagine a template

```
{% raw %}
{% extends base.html %}

{% block content %}

  <h1>Course Information</h1>
  {% now "DATETIME_FORMAT" as today %}
  {% if course.start|date:"DATETIME_FORMAT" > today %}
    {# course has not started yet #}
  {% elif course.start|date:"DATETIME_FORMAT" < today
      and course.end|date:"DATETIME_FORMAT" > today %}
    {# course is in session #}
  {% elif course.end|date:"DATETIME_FORMAT" < today %}
    {# course has ended #}
  {% endif %}

{% endblock %}
{% endraw %}
```

This is at the highest level where the processing is being done in the template language. While Django Templates are a great tool, logic implemented here is not going to be as performant or efficient. Furthermore it can get pretty messy, especially if we have complex interactions going on. Lastly, it doesn't follow the DRY (don't repeat yourself) principle where we likely have to reimplement this logic everywhere we go.

## Answer: views.py logic

Our second option is to define this logic at the `views.py` level. Having our logic in the Views is considered a poor practice because the job of the views it to manage the context and request and pass off the appropriate calls to other systems. It is supposed to be loosely coupled with the models, this follows the MVC (Model-View-Controller) pattern and when we include our logic in the views this pattern's intention breaks down.

Having the logic in our view at the very least is better than having it in the template, we have the opportunity to create re-usable classes, abstracting away common functionality and using mixins so we follow DRY. The performance should also be improved granted we are correctly using the tools Django provides when querying with the ORM and using caching strategies where appropriate.

```
from django.utils import timezone


def in_session(course):
    now = timezone.now()
    if course.start < now and course.end > now:
        return True
    return False

def not_started(course):
    now = timezone.now()
    if course.start > now:
        return True
    return False

def ended(course):
    now = timezone.now()
    if course.end < now:
        return True
    return False


class CourseDetailView(DetailView):
    model = Course
    template = "<appname>/course_detail.html"

    # we extend the get_context_data method to provide the information
    # we want to display in the template
    def get_context_data(self, **kwargs):
        context = super(CourseDetailView, self).get_context_data(**kwargs)
        if in_session(context['course']):
            context['info_msg'] = 'This course is currently in session.'
        elif not_started(context['course']):
            context['info_msg'] = 'This course has not started yet.'
        elif ended(context['course']):
            context['info_msg'] = 'This course has ended.'
        return context
```

Now whenever we need to find out information about a course we can use these methods to render the appropriate logic. As stated above though, this couples the logic with the view and not with the model.

## Answer: models.py logic

We now arrive at the best place for this business logic, coupled with the model's themselves. Let's move those `in_session(course)` and `not_started(course)` methods from our views into our models themselves.

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

The `course` parameter chances to `self` as it references the object and we can now use these model methods in our `views.py`.

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

Whenever we need business logic, we can use our predefined methods on the models. In this manner the logic is coupled at the lowest level besides the database queries themselves. This helps ensure reusability of the logic so we don't repeat ourselves and reduces the chance for bugs to creep into our code by having one source of truth that can have appropriate tests that pair with it to ensure its validity.

Now that we are here, we can start talking about that next level down, the database queries themselves, read on to [Models and Managers Part II: Custom Managers in Django](https://audiolion.github.io/django/2016/12/03/models-and-managers-part-ii.html).





