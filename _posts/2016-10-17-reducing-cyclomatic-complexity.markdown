---
layout: post
title:  "Reducing Cyclomatic Complexity with Python"
date:   2016-10-17 10:26:00
author: Ryan Castner
categories: Python
tags: python cyclomatic complexity tips programming
cover:  "/assets/python-programming.jpg"
custom_js_highlight:
- python_highlights
---

# Reducing Cyclomatic Complexity with Python

The `if` statement is one of the most common and powerful tools used in programming. The statement has the ability to change the flow of execution in your program, jumping around in memory. In fact, in assembly language, one of the closest languages to machine code, the instruction used to change program flow is called `jump` because you are literally jumping around in memory to execute code stored at different non-continguous locations. The `if` statement is also one of the most abused statements by young programmers who have yet to get a solid grasp of theory, and I daresay more experienced programmers can fall into the trap of using it. `If` statements are easy, they are a build-as-you-go approach to software development that do not require you to plan ahead for how to structure your code and just deal with bugs, edge cases, and other problems as they crop up by throwing a statement in there.

```
class Bird(object):
  name = ''
  flightless = False
  extinct = False

  def get_speed(self):
    if self.extinct:
      return -1 # we do not care about extinct bird speeds
    else:
      if self.flightless:
        if self.name == 'Ostrich':
          return 15
        elif self.name == 'Chicken':
          return 7
        elif self.name == 'Flamingo':
          return 8
        else:
          return -1 # bird name not implemented
      else:
        if self.name == 'Gold Finch':
          return 12
        elif self.name == 'Bluejay':
          return 10
        elif self.name == 'Robin':
          return 14
        elif self.name == 'Hummingbird':
          return 16
        else:
          return -1 # bird not implemented
```

If a new bird is added, the code must be updated to include another if for this condition, correctly classified in the nested logic. This contrived example (and note bird's speeds are completely fictional) shows how messy code like this can get. This code comes with a `Cyclomatic Complexity` of `10`. Cyclomatic complexity is a metric used in software development to calculate how many independent paths of execution exist in code. The `Bird` class above has a cyclomatic complexity of 10, right on the cusp of where we don't want to be. While there is no hard-and-fast rule for max code complexity, typically 10 or more is a sign that you should refactor.

Studies on cyclomatic complexity as it relates to number of defects that appear in software show a correlation but have not necessarily implied causation. Needless to say, there are design patterns and principles we can use to combat cyclomatic complexity that also make our code easier to maintain and more highly extensible. In that case we may as well kill two birds (or 10) with one stone.

## Computing Cyclomatic Complexity with Radon
[Radon](http://radon.readthedocs.io/en/latest/index.html) is a Python library that computes a couple of code metrics, one of which is code complexity. To install it run `pip install radon`. To calculate complexity we use the `cc` command line argument and specify the directories or files we want to compute statistics on. The `-s` option shows the actual computed cyclomatic complexity in the output.

```
radon cc -s birds.py
  ./birds.py
      C 1135:0 Bird - B (10)
      M 1140:2 Bird.get_speed - B (10)
```

## Reducing Cylcomatic Complexity with Polymorphism

For those who want to freshen up on the idea of polymorphism, [Jeff Knup](https://jeffknupp.com/blog/2014/06/18/improve-your-python-python-classes-and-object-oriented-programming/) has a great article on it. The main idea behind polymorphism is that you abstract away features common to many classes which inherit and can implement any differences that may exist. Another way of looking at the problem is that we can define an abstract interface, and have classes implement that interface contract. Each class then becomes the `if` condition, based on its type.

```
class Bird(object):
  name = ''
  flightless = False
  extinct = False

  def get_speed(self):
    raise NotImplementedError

class Robin(Bird):
  name = 'Robin'

  def get_speed(self):
    return 14

class GoldFinch(Bird):
  name = 'Gold Finch'

  def get_speed(self):
    return 12

class Ostrich(Bird):
  name = 'Ostrich'
  flightless = True

  def get_speed(self):
    return 15

class Pterodactyl(Bird):
  name = 'Pterodactyl'
  extinct = True

  def get_speed(self):
    return -1
```

We have now have reduced the Cyclomatic complexity of the Bird class and all subclasses of Bird to 1.

```
radon cc -s birds.py
  ./birds.py
    C 1166:0 Bird - A (1)
    M 1171:2 Bird.get_speed - A (1)
    C 1174:0 Robin - A (1)
    M 1177:2 Robin.get_speed - A (1)
    C 1180:0 GoldFinch - A (1)
    M 1183:2 GoldFinch.get_speed - A (1)
    C 1186:0 Ostrich - A (1)
    M 1190:2 Ostrich.get_speed - A (1)
    C 1193:0 Pterodactyl - A (1)
    M 1197:2 Pterodactyl.get_speed - A (1)
```

The concept seems easy enough, right? The basic idea is we can decompose conditionals into classes that are polymorphic, the base class will implement the method and each subclass implements it in its own way. In this respect, we never need to worry about the logic when we want a bird's speed, we just call the `my_bird.get_speed()` method and the result is computed without any code caring about what type of bird it is.

In this way we have completely eliminated the usage of the `if` statement! In fact, if you want a challenge, try coding without using an `if` statement for control flow. It is possible, and sometimes a simple `if` is much better than abstracting code away, but the idea here is to break away our reliance on it and see other means to accomplish the same end.

---

At this point we will delve into a much longer and bigger example that comes from my own work. While it uses the polymorphic method above to reduce cyclomatic complexity, it is not a perfect example and requires a lot of other refactoring. So if you want to see my journey in taking a monstrosity method and breaking it out into reasoned classes then carry on!

## A more complex example

This example is taken from a codebase I work on. It was written to respond to an `AJAX` request with appropriate `JSON` so that a calendar could be rendered on a website. The training wheels are coming off now, don't worry if you have no idea what this code is doing. Even with intimate knowledge of the codebase I have a hard time understanding it. I will break it down into parts so we can look at it.

```
class EventAPIList(View):
    """
    View which provides the Event Json stream for the JS Calendar
    """
    def get(self, request, *args, **kwargs):
        default_time = datetime.now()
        start = parse(request.GET.get("start", ""), default=default_time)
        end = parse(request.GET.get("end", ""), default=default_time)

        event_occurrences = []
        if request.GET.get("event", None) is not None:
            events = (get_object_or_404(Event, pk=int(request.GET.get("event"))), )
        else:
            events = Event.published.all()

        for event in events:
            for meeting in event.meetings_between(start, end):
                occurrence, occurrence_type = meeting

                color = EventDictionary.STRAND_COLORS_DICT.get(event.strand_type)
                if color is None:
                    color = EventDictionary.STRAND_COLORS_DICT.get(EventDictionary.GENERAL)

                strand = EventDictionary.STRAND_TYPES_DICT.get(event.strand_type)
                strand_type = event.strand_type

                if strand is None:
                    strand = EventDictionary.STRAND_TYPES_DICT.get(EventDictionary.GENERAL)
                    strand_type = EventDictionary.GENERAL

                if occurrence_type == 'reg_open':
                    event_occurrences.append({
                        "title": 'Registration Opens for ' + event.title,
                        "description": 'Registration Opens for ' + event.title,
                        "color": "#34495e",
                        "strand": strand_type,
                        "start": str(occurrence),
                        "url": str(event.get_absolute_url()),
                        "occurrence_type": occurrence_type
                    })
                elif occurrence_type == 'reg_close':
                    event_occurrences.append({
                        "title": 'Registration Closes for ' + event.title,
                        "description": 'Registration Closes for ' + event.title,
                        "color": "#34495e",
                        "strand": strand_type,
                        "start": str(occurrence),
                        "url": str(event.get_absolute_url()),
                        "occurrence_type": occurrence_type
                    })
                else:
                    description = strip_tags(event.description)
                    description = description.replace('&nbsp;', ' ')
                    event_occurrences.append({
                        "title": event.title,
                        "description": strand + "\n" + description[:140] + '..',
                        "color": color,
                        "strand": strand_type,
                        "start": str(occurrence),
                        "url": str(event.get_absolute_url()),
                        "occurrence_type": occurrence_type
                    })
        return HttpResponse(json.dumps(event_occurrences), content_type="application/json")


class EventAPICart(View):
    """
    View which provides the Event Json stream for the JS Calendar
    """
    def get(self, request, *args, **kwargs):
        default_time = datetime.now()
        start = parse(request.GET.get("start", ""), default=default_time)
        end = parse(request.GET.get("end", ""), default=default_time)

        event_occurrences = []
        cart = ShoppingCart.objects.get(owner=request.user.pk)
        cart_events = cart.events.all()
        usr = User.objects.get(pk=request.user.pk)
        user_regs = usr.registration_set.exclude(status=Registration.DENIED)
        user_enrolled_events = []
        user_pending_events = []

        for reg in user_regs:
            if reg.event.in_session():
                if reg.status == Registration.APPROVED:
                    user_enrolled_events.append(reg.event)
                else:
                    user_pending_events.append(reg.event)

        events_list = [cart_events, user_pending_events, user_enrolled_events]

        for idx, lst in enumerate(events_list):
            for event in lst:
                for meeting in event.meetings_between(start, end):
                    occurrence, occurrence_type = meeting

                    if idx == 0:
                        color = "#F36E21"
                    else:
                        if event in user_enrolled_events:
                            color = "#5cb85c"
                        else:
                            color = "#5bc0de"

                    strand = EventDictionary.STRAND_TYPES_DICT.get(event.strand_type)
                    strand_type = event.strand_type

                    if strand is None:
                        strand = EventDictionary.STRAND_TYPES_DICT.get(EventDictionary.GENERAL)
                        strand_type = EventDictionary.GENERAL

                    if occurrence_type == 'reg_open':
                        event_occurrences.append({
                            "title": 'Registration Opens for ' + event.title,
                            "description": 'Registration Opens for ' + event.title,
                            "color": "#34495e",
                            "strand": strand_type,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type
                        })
                    elif occurrence_type == 'reg_close':
                        event_occurrences.append({
                            "title": 'Registration Closes for ' + event.title,
                            "description": 'Registration Closes for ' + event.title,
                            "color": "#34495e",
                            "strand": strand_type,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type
                        })
                    else:
                        description = strip_tags(event.description)
                        description = description.replace('&nbsp;', ' ')
                        event_occurrences.append({
                            "title": event.title,
                            "description": strand + "\n" + description[:140] + '..',
                            "color": color,
                            "strand": strand_type,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type
                        })
        return HttpResponse(json.dumps(event_occurrences), content_type="application/json")
```

The cyclomatic complexity of this code comes in at a whopping 21 points! You can likely see now why this metric has been shown to correlate with number of defects in code. Obviously the method can still be tested, validated, and error-free. However, the code is extremely hard to maintain and the ease of which an error could be introduced is much higher.

Let's start with an obvious fact, we have two classes `EventAPIList` and `EventAPICart`. Besides the less-than-stellar names we have two classes whose responsibilities are incredibly similar and we have a lot of duplicated code. Polymorphism and Inheritance help us combat this duplication by putting common code into a parent class with individual differences being exhibited in the subclasses.

These classes are contained within a `views.py` file, the Django -- [a web framework for perfectionists with a deadline](https://www.djangoproject.com/) -- standard naming convention for the file which contains the views (V in MVC pattern) for a website application. Let's first move these classes from the 1000+ [LLOC](http://radon.readthedocs.io/en/latest/intro.html#raw-metrics) views file they are in to a separate `api.py` file. While this is not a regular Django convention, it clearly indicates the responsibility of the file and the code contained within. With a little documentation (through the docstring) we can clear up any questions a Django developer might have about the file.

The `api.py` abstraction allows us to remove `API` from the names of the classes and give them more representative names.

```
## ./api.py

# -*- coding: utf-8 -*-
"""API Endpoints for the Calendar

Endpoints designed to be called with AJAX requests and retrieve the events
that should populate the webpage's calendar.
"""


class SingleEvent(View):
    """Provides JSON Endpoint for a Single Calendar Events Schedule
    """


class AllEvents(View):
    """Provides JSON Endpoint for All Events on Calendar

    Events include Registration open and close dates, general events, and
    events that have multiple occurrences over time. Events are colored based
    on their Event Strand color.
    """


class UserEvents(View):
    """Provides JSON Endpoint for User's Schedule

    Events returned are only those that are part of the requested user's
    schedule. Events are broken into three categories, events in the user's
    shopping cart (potentially registering for), events that the user has
    requested to register for, and events that the user has been approved
    for.
    """

```

We have now created a separation of concerns. Notice there is no parent class as of yet. I think it is easier to see the solution by first coding duplication into the classes and abstracting it later.

First let's separate out a few of the `if` statements logic into their respective classes.

```
if request.GET.get("event", None) is not None:
    events = (get_object_or_404(Event, pk=int(request.GET.get("event"))), )
else:
    events = Event.published.all()
```

The above was originally being used to direct the flow control by changing what events were grabbed from the database, all events or a single event, based on whether the word 'event' was part of the 'GET' url query parameters e.g. `?event=10&start=2016-09-16&end=2016-11-05`.

Now we can break up that statement and assign `events` in each respective view.

```
class SingleEvent(View):

    def get(self, request, *args, **kwargs):
        default_time = datetime.now()
        start = parse(request.GET.get("start", ""), default=default_time)
        end = parse(request.GET.get("end", ""), default=default_time)

        events = (get_object_or_404(Event, pk=int(request.GET.get("event"))), )


class AllEvents(View):

    def get(self, request, *args, **kwargs):
        default_time = datetime.now()
        start = parse(request.GET.get("start", ""), default=default_time)
        end = parse(request.GET.get("end", ""), default=default_time)

        events = Event.published.all()
```

Cheers for -1 cyclomatic complexity! I hope we already see some obvious duplication of code going on. Let's make that base class and put the common code in there.

```
class EventApiMixin(object):
    """Base class for the Event API holding common code
    """

    def parse_request(self, request):
        default_time = datetime.now()
        start = parse(request.GET.get("start", ""), default=default_time)
        end = parse(request.GET.get("end", ""), default=default_time)
        return (start, end)
```

The parse_request function handles the parsing of the request and returns a tuple with the start and end time. A quick refactor of our inheriting classes gives us:

```
class SingleEvent(View):

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        events = (get_object_or_404(Event, pk=int(request.GET.get("event"))), )


class AllEvents(View):

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        events = Event.published.all()


class UserEvents(View):

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        events = Event.published.all()
```

Now let's tackle that for block of code.

```
def get(self, request, *args, **kwargs):
  """
  Iterate through all the events queried from the database, grab their associated meetings between the requested
  start and end date. With each meeting occurrence create a json object and append it to a list where the json
  generated relies on the occurrence type, a registration open occurrence, a registration close occurrence, or
  a regular occurrence.
  """

  for event in events:
    for meeting in event.meetings_between(start, end):
        occurrence, occurrence_type = meeting

        color = EventDictionary.STRAND_COLORS_DICT.get(event.strand_type)
        if color is None:
            color = EventDictionary.STRAND_COLORS_DICT.get(EventDictionary.GENERAL)

        strand = EventDictionary.STRAND_TYPES_DICT.get(event.strand_type)
        strand_type = event.strand_type

        if strand is None:
            strand = EventDictionary.STRAND_TYPES_DICT.get(EventDictionary.GENERAL)
            strand_type = EventDictionary.GENERAL

        if occurrence_type == 'reg_open':
            event_occurrences.append({
                "title": 'Registration Opens for ' + event.title,
                "description": 'Registration Opens for ' + event.title,
                "color": "#34495e",
                "strand": strand_type,
                "start": str(occurrence),
                "url": str(event.get_absolute_url()),
                "occurrence_type": occurrence_type
            })
        elif occurrence_type == 'reg_close':
            event_occurrences.append({
                "title": 'Registration Closes for ' + event.title,
                "description": 'Registration Closes for ' + event.title,
                "color": "#34495e",
                "strand": strand_type,
                "start": str(occurrence),
                "url": str(event.get_absolute_url()),
                "occurrence_type": occurrence_type
            })
        else:
            description = strip_tags(event.description)
            description = description.replace('&nbsp;', ' ')
            event_occurrences.append({
                "title": event.title,
                "description": strand + "\n" + description[:140] + '..',
                "color": color,
                "strand": strand_type,
                "start": str(occurrence),
                "url": str(event.get_absolute_url()),
                "occurrence_type": occurrence_type
            })
```

The start of this block is a little interesting, we are defining the `color`, `strand`, and `strand_type` variables that go into our json property that rely solely on the `event` object, but we are doing so inside the `meeting` inner for-loop. In essence, we are doing a bunch of extra computation for no reason. Let's bump this code out to the first for-loop.

```
def get(self, request, *args, **kwargs):
  # ...

  for event in events:
    color = EventDictionary.STRAND_COLORS_DICT.get(event.strand_type)
    if color is None:
        color = EventDictionary.STRAND_COLORS_DICT.get(EventDictionary.GENERAL)

    strand = EventDictionary.STRAND_TYPES_DICT.get(event.strand_type)

    if strand is None:
        strand = EventDictionary.STRAND_TYPES_DICT.get(EventDictionary.GENERAL)
        strand_type = EventDictionary.GENERAL

    for meeting in event.meetings_between(start, end):
      # ...
```

After bumping it out, we take a look at the code. The `strand` and `strand_type` definitions are confusing, what is the difference between them? `strand` appears to be the verbose display string for the `strand_type` which is a unique two character code. We will rename these to `event_strand` and `strand_verbose`. Furthermore, all these codes and `ifs` are being duplicated across our three classes. To the parent `EventsApiMixin` class we go!

```
class EventApiMixin(object):
  # ...

  def get_color(self, event):
      color = EventDictionary.STRAND_COLORS_DICT.get(event.strand_type)
      if color is None:
          color = EventDictionary.STRAND_COLORS_DICT.get(EventDictionary.GENERAL)
      return color

  def get_strand(self, event):
      event_strand = event.strand_type
      strand_verbose = EventDictionary.STRAND_TYPES_DICT.get(event_strand)

      if strand_verbose is None:
          strand = EventDictionary.STRAND_TYPES_DICT.get(EventDictionary.GENERAL)
          event_strand = EventDictionary.GENERAL

      return (event_strand, strand_verbose)
```

Now each event api subclass is looking a bit like this:

```
class AllEvents(EventApiMixin, View):

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        events = Event.published.all()

        events_json = []
        for event in events:
            color = self.get_color(event)
            strand, strand_verbose = self.get_strand(event)

            for meeting in event.meetings_between(start, end):
              # ...
```

Looking great so far. We will now go into the inner meeting for-loop chunk of code and refactor it.

```
def get(self, request, *args, **kwargs):
  # ...
    for meeting in event.meetings_between(start, end):
      occurrence, occurrence_type = meeting

      if occurrence_type == 'reg_open':
          event_occurrences.append({
              "title": 'Registration Opens for ' + event.title,
              "description": 'Registration Opens for ' + event.title,
              "color": "#34495e",
              "strand": strand_type,
              "start": str(occurrence),
              "url": str(event.get_absolute_url()),
              "occurrence_type": occurrence_type
          })
      elif occurrence_type == 'reg_close':
          event_occurrences.append({
              "title": 'Registration Closes for ' + event.title,
              "description": 'Registration Closes for ' + event.title,
              "color": "#34495e",
              "strand": strand_type,
              "start": str(occurrence),
              "url": str(event.get_absolute_url()),
              "occurrence_type": occurrence_type
          })
      else:
          description = strip_tags(event.description)
          description = description.replace('&nbsp;', ' ')
          event_occurrences.append({
              "title": event.title,
              "description": strand + "\n" + description[:140] + '..',
              "color": color,
              "strand": strand_type,
              "start": str(occurrence),
              "url": str(event.get_absolute_url()),
              "occurrence_type": occurrence_type
          })
```

We have an if-clause based on the event. Lets define a polymorphic class that takes away the `if` clause checking in this method.

```
class BaseOccurrence(object):
    title = ""
    description = ""
    color = ""
    strand = ""
    start = ""
    url = ""
    occurrence_type = ""

    def __init__(self, **kwargs):
        self.strand = kwargs.get('strand', '')
        self.start = kwargs.get('start', '')
        self.url = kwargs.get('url', '')

    def to_dict(self):
        return {"title": self.title,
                "description": self.description,
                "color": self.color,
                "strand": self.strand,
                "start": self.start,
                "url": self.url,
                "occurrence_type": self.occurrence_type}


class EventOccurrence(BaseOccurrence):
    occurrence_type = "event"

    def __init__(self, **kwargs):
        self.title = kwargs.get('title', '')
        self.description = kwargs.get('description', '')
        self.color = kwargs.get('color', '')
        super(BaseOccurrence, self).__init__(**kwargs)


class OpenOccurrence(BaseOccurrence):
    occurrence_type = "reg_open"
    color = "#34495e"

    def __init__(self, **kwargs):
        self.title = 'Registration Opens for ' + kwargs.get('title', '')
        self.description = 'Registration Open for ' + kwargs.get('description', '')
        super(BaseOccurrence, self).__init__(**kwargs)


class ClosedOccurrence(BaseOccurrence):
    occurrence_type = "reg_close"
    color = "#34495e"

    def __init__(self, **kwargs):
        self.title = 'Registration Closes for ' + kwargs.get('title', '')
        self.description = 'Registration Closes for ' + kwargs.get('description', '')
        super(BaseOccurrence, self).__init__(**kwargs)
```

Now we update our `EventsApiMixin` to have a `create_event_dict` method that accepts the kwargs and constructs the appropriate object, returning a dictionary representation of our object.

```
class EventApiMixin(object):
  def create_event_dict(self, **kwargs):
    if kwargs.get('occurrence_type', '') == OpenOccurrence.occurrence_type:
        return OpenOccurrence(**kwargs).to_dict()
    elif kwargs.get('occurrence_type', '') = ClosedOccurrence.occurrence_type:
        return ClosedOccurrence(**kwargs).to_dict()
    elif kwargs.get('occurrence_type', '') = EventOccurrence.occurrence_type:
        return EventOccurrence(**kwargs).to_dict()
    else:
        return BaseOccurrence.to_dict()
```

This dictionary representation of our object can now be appended to the list of dictionaries that will be converted to json and dumped across the wire.

```
class AllEvents(EventApiMixin, View):

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        events = Event.published.all()

        events_json = []
        for event in events:
            color = self.get_color(event)
            strand, strand_verbose = self.get_strand(event)

            for meeting in event.meetings_between(start, end):
                occurrence, occurrence_type = meeting
                kwargs = {  "title": event.title,
                            "description": event.description,
                            "color": color,
                            "strand": strand,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type}
                events_json.append(self.create_event_json(event, **kwargs))
```

With this we now have two of our api endpoints knocked out `SingleEvent` and `AllEvents` as the only differentiating logic was the initial definition of events. We will now move to the `UserEvents` view. The original view was as follows

```
class EventAPICart(View):

    def get(self, request, *args, **kwargs):
        default_time = datetime.now()
        start = parse(request.GET.get("start", ""), default=default_time)
        end = parse(request.GET.get("end", ""), default=default_time)

        event_occurrences = []
        cart = ShoppingCart.objects.get(owner=request.user.pk)
        cart_events = cart.events.all()
        usr = User.objects.get(pk=request.user.pk)
        user_regs = usr.registration_set.exclude(status=Registration.DENIED)
        user_enrolled_events = []
        user_pending_events = []

        for reg in user_regs:
            if reg.event.in_session():
                if reg.status == Registration.APPROVED:
                    user_enrolled_events.append(reg.event)
                else:
                    user_pending_events.append(reg.event)

        events_list = [cart_events, user_pending_events, user_enrolled_events]

        event_occurrences = []
        for idx, lst in enumerate(events_list):
            for event in lst:
                for meeting in event.meetings_between(start, end):
                    occurrence, occurrence_type = meeting

                    if idx == 0:
                        color = "#F36E21"
                    else:
                        if event in user_enrolled_events:
                            color = "#5cb85c"
                        else:
                            color = "#5bc0de"

                    strand = EventDictionary.STRAND_TYPES_DICT.get(event.strand_type)
                    strand_type = event.strand_type

                    if strand is None:
                        strand = EventDictionary.STRAND_TYPES_DICT.get(EventDictionary.GENERAL)
                        strand_type = EventDictionary.GENERAL

                    if occurrence_type == 'reg_open':
                        event_occurrences.append({
                            "title": 'Registration Opens for ' + event.title,
                            "description": 'Registration Opens for ' + event.title,
                            "color": "#34495e",
                            "strand": strand_type,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type
                        })
                    elif occurrence_type == 'reg_close':
                        event_occurrences.append({
                            "title": 'Registration Closes for ' + event.title,
                            "description": 'Registration Closes for ' + event.title,
                            "color": "#34495e",
                            "strand": strand_type,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type
                        })
                    else:
                        description = strip_tags(event.description)
                        description = description.replace('&nbsp;', ' ')
                        event_occurrences.append({
                            "title": event.title,
                            "description": strand + "\n" + description[:140] + '..',
                            "color": color,
                            "strand": strand_type,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type
                        })
        return HttpResponse(json.dumps(events_occurrences), content_type="application/json")
```

The `UserEvents` view has custom logic for its `get_color` function so we override the superclasses definition with our own. We then account for the slight difference in how events are grabbed and put in a list of event lists and iterated through. The same deal happened here with the logic for grabbing color and strand were inside the nested forloop so we moved them up a block.

```
class UserEvents(View):
    """Provides JSON Endpoint for User's Schedule

    Events returned are only those that are part of the requested user's
    schedule. Events are broken into three categories, events in the user's
    shopping cart (potentially registering for), events that the user has
    requested to register for, and events that the user has been approved
    for.
    """
    user_enrolled_events = []
    user_pending_events = []

    def get_color(self, idx, event):
        if idx == 0:
            color = "#F36E21"
        else:
            if event in self.user_enrolled_events:
                color = "#5cb85c"
            else:
                color = "#5bc0de"
        return color

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        cart = ShoppingCart.objects.get(owner=request.user.pk)
        cart_events = cart.events.all()

        usr = User.objects.get(pk=request.user.pk)
        user_regs = usr.registration_set.exclude(status=Registration.DENIED)

        for reg in user_regs:
            if reg.event.in_session():
                if reg.status == Registration.APPROVED:
                    self.user_enrolled_events.append(reg.event)
                else:
                    self.user_pending_events.append(reg.event)

        events_list = [cart_events, user_pending_events, user_enrolled_events]

        events_json = []
        for idx, lst in enumerate(events_list):
            for event in lst:
                color = self.get_color(idx, event)
                strand, strand_verbose = self.get_strand(event)
                for meeting in event.meetings_between(start, end):
                    occurrence, occurrence_type = meeting
                    kwargs = {  "title": event.title,
                                "description": event.description,
                                "color": color,
                                "strand": strand,
                                "start": str(occurrence),
                                "url": str(event.get_absolute_url()),
                                "occurrence_type": occurrence_type}
                    events_json.append(self.create_event_json(event, **kwargs))
        return HttpResponse(json.dumps(events_json), content_type="application/json")
```

We now have the same functionality as the previous view in the now refactored endpoint. This view only lets the user see their own schedule, and this is by design for now. Though with some quick changes we can create a separate endpoint for administrators that queries any view.

```
class UserSchedule(GroupRequiredMixin, UserEvents):
    group_required = [u"admins", u"managers"]

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)
        events_list = self.create_event_list(kwargs['pk'])
        events_json = self.create_events_json(events_list)
        return HttpResponse(json.dumps(events_json), content_type="application/json")
```

We override the get() method that is inherited from the UserEvents class to take the primary key from the kwargs on the endpoint and check that the user requesting is a member of the admins or managers group. To reduce code duplication I broke the event_list creation section and json for loop into separate functions in the UserEvents class so it now looks like this.

```
class UserEvents(EventApiMixin, View):
    """Provides JSON Endpoint for User's Schedule

    Events returned are only those that are part of the requested user's
    schedule. Events are broken into three categories, events in the user's
    shopping cart (potentially registering for), events that the user has
    requested to register for, and events that the user has been approved
    for.
    """
    user_enrolled_events = []
    user_pending_events = []

    def get_color(self, idx, event):
        if idx == 0:
            color = "#F36E21"
        else:
            if event in self.user_enrolled_events:
                color = "#5cb85c"
            else:
                color = "#5bc0de"
        return color

    def create_event_list(self, user_pk):
        cart = ShoppingCart.objects.get(owner=user_pk)
        cart_events = cart.events.all()

        usr = User.objects.get(pk=user_pk)
        user_regs = usr.registration_set.exclude(status=Registration.DENIED)

        for reg in user_regs:
            if reg.event.in_session():
                if reg.status == Registration.APPROVED:
                    self.user_enrolled_events.append(reg.event)
                else:
                    self.user_pending_events.append(reg.event)
        return [cart_events, self.user_pending_events, self.user_enrolled_events]

    def create_events_json(self, events_list):
        events_json = []
        for idx, lst in enumerate(events_list):
            for event in lst:
                color = self.get_color(idx, event)
                strand, strand_verbose = self.get_strand(event)
                for meeting in event.meetings_between(start, end):
                    occurrence, occurrence_type = meeting
                    kwargs = {  "title": event.title,
                                "description": event.description,
                                "color": color,
                                "strand": strand,
                                "start": str(occurrence),
                                "url": str(event.get_absolute_url()),
                                "occurrence_type": occurrence_type}
                    events_json.append(self.create_event_json(event, **kwargs))

        return events_json

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)
        events_list = self.create_event_list(request.user.pk)
        events_json = self.create_events_json(events_list)
        return HttpResponse(json.dumps(events_json), content_type="application/json")
```

Congratulations. You made it. Well, almost. Let's take a look at our `SingleEvent` and `AllEvents` views.

```
class SingleEvent(EventApiMixin, View):

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        events = (get_object_or_404(Event, pk=int(request.GET.get("event"))), )
        events_json = []
        for event in events:
            color = self.get_color(event)
            strand, strand_verbose = self.get_strand(event)

            for meeting in event.meetings_between(start, end):
                occurrence, occurrence_type = meeting
                kwargs = {  "title": event.title,
                            "description": event.description,
                            "color": color,
                            "strand": strand,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type}
                events_json.append(self.create_event_json(event, **kwargs))
        return HttpResponse(json.dumps(events_json), content_type="application/json")

class AllEvents(EventApiMixin, View):

    def get(self, request, *args, **kwargs):
        start, end = self.parse_request(request)

        events = Event.published.all()

        events_json = []
        for event in events:
            color = self.get_color(event)
            strand, strand_verbose = self.get_strand(event)

            for meeting in event.meetings_between(start, end):
                occurrence, occurrence_type = meeting
                kwargs = {  "title": event.title,
                            "description": event.description,
                            "color": color,
                            "strand": strand,
                            "start": str(occurrence),
                            "url": str(event.get_absolute_url()),
                            "occurrence_type": occurrence_type}
                events_json.append(self.create_event_json(event, **kwargs))
        return HttpResponse(json.dumps(events_json), content_type="application/json")
```

I hope by now you are seeing the duplicated code and know the drill.

```
def create_events_json(self, events):
  events_json = []
  for event in events:
    color = self.get_color(event)
    strand, strand_verbose = self.get_strand(event)

    for meeting in event.meetings_between(start, end):
        occurrence, occurrence_type = meeting
        kwargs = {  "title": event.title,
                    "description": event.description,
                    "color": color,
                    "strand": strand,
                    "start": str(occurrence),
                    "url": str(event.get_absolute_url()),
                    "occurrence_type": occurrence_type}
        events_json.append(self.create_event_json(event, **kwargs))
  return events_json
```

Add that method to our mixin and we get our final code. The gists can be viewed [here](https://gist.github.com/audiolion/93eacfb86033a67e4ca02fdc1282de8c)

Our final radon run on cyclomatic complexity looks much better:

```
  $ radon cc -s pd/api.py
pd/api.py
    M 104:4 EventApiMixin.create_event_dict - A (4)
    M 184:4 UserEvents.create_event_list - A (4)
    M 199:4 UserEvents.create_events_json - A (4)
    M 114:4 EventApiMixin.create_events_json - A (3)
    C 162:0 UserEvents - A (3)
    M 174:4 UserEvents.get_color - A (3)
    C 79:0 EventApiMixin - A (2)
    M 89:4 EventApiMixin.get_color - A (2)
    M 95:4 EventApiMixin.get_strand - A (2)
    C 19:0 BaseOccurrence - A (1)
    M 28:4 BaseOccurrence.__init__ - A (1)
    M 33:4 BaseOccurrence.to_dict - A (1)
    C 43:0 EventOccurrence - A (1)
    M 46:4 EventOccurrence.__init__ - A (1)
    M 52:4 EventOccurrence.parse_description - A (1)
    C 59:0 OpenOccurrence - A (1)
    M 63:4 OpenOccurrence.__init__ - A (1)
    C 69:0 ClosedOccurrence - A (1)
    M 73:4 ClosedOccurrence.__init__ - A (1)
    M 83:4 EventApiMixin.parse_request - A (1)
    C 133:0 SingleEvent - A (1)
    M 137:4 SingleEvent.get - A (1)
    C 144:0 AllEvents - A (1)
    M 152:4 AllEvents.get - A (1)
    M 218:4 UserEvents.get - A (1)
    C 225:0 UserSchedule - A (1)
    M 228:4 UserSchedule.get - A (1)
```
