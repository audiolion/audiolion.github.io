---
layout: post
title:  "Optimizing Queries across Foreign Keys in Django"
date:   2016-12-02 6:54:00
author: Ryan Castner
categories: Django
tags: django query queries values optimize webdev
cover:  "/assets/header_image_default.jpg"
custom_js_highlight:
- django_highlights
---

# Optimizing Queries across Foreign Keys in Django

The other day I was working on implementing a bit of business logic that had me traversing across multiple foreign key relationships in my Django models. In an effort to optimize it I did some research on how to do so and wanted to share my findings with others in the hopes it may help them optimize some of their queries.

## Defining Models

```
class Event(models.Model):
    title = models.CharField()


class Registration(models.Model):
    APPROVED = 'a'
    PENDING = 'p'

    REGISTRATION_STATUS_CHOICES = (
        (APPROVED, 'Approved'),
        (PENDING, 'Pending'),
    )
    user = models.ForeignKey(settings.AUTH_USER_MODEL)
    event = models.ForeignKey(Event, related_name='registrations')
    status = models.CharField(max_length=1, choices=REGISTRATION_STATUS_CHOICES, default=PENDING)


class AttendanceLog(models.Model):
    date = models.DateField()
    event = models.ForeignKey(Event)


class AttendanceRecord(models.Model):
    PRESENT = 'p'
    ABSENT = 'a'

    ATTENDANCE_STATUS_CHOICES = (
        (PRESENT, 'Present'),
        (ABSENT, 'Absent')
    )
    attendancelog = models.ForeignKey(AttendanceLog, related_name='records')
    attendee = models.ForeignKey(settings.AUTH_USER_MODEL)
    status = models.CharField(max_length=1, choices=ATTENDANCE_STATUS_CHOICES, default=ABSENT)
```

The story is that an Event requires Registration to attend. We want to be able to take Attendance to an event. That is where we derive our models from. We have our Event model which has many Registration models associated with it. AttendanceLog and AttendanceRecord are split so that we don't duplicate the date and event field on every record, it also plays nicer with Django being able to create a nested form where the user only has to enter the date once.

The problem is the AttendanceLog and AttendanceRecord models know nothing about the Registration model. According to our business logic, attendance can only be taken for a user if they have an `APPROVED` Registration to the event. Instead of trying to pair up these models with a many-to-many through relationship we can just include some simple logic on the model save to ensure that the business logic is enforced.

```
class AttendanceRecord(models.Model):
    PRESENT = 'p'
    ABSENT = 'a'

    ATTENDANCE_STATUS_CHOICES = (
        (PRESENT, 'Present'),
        (ABSENT, 'Absent')
    )
    attendancelog = models.ForeignKey(AttendanceLog, related_name='records')
    attendee = models.ForeignKey(settings.AUTH_USER_MODEL)
    status = models.CharField(max_length=1, choices=ATTENDANCE_STATUS_CHOICES, default=ABSENT)

    def save(self, *args, **kwargs):
        attendees = self.get_registered_attendees()
        if self.attendee not in attendees:
            raise ValidationError("Cannot register attendance for a user without an approved registration for the event.")
        super(AttendanceRecord, self).save(*args, **kwargs)

    def get_registered_attendees(self):
        # TODO
```

Here we have an overriden save method that gets a list of all attendees and checks if the attendee is in the list of registered attendees, if they aren't it kicks back a validationerror and the save doesn't go through. Otherwise we call the normal save method by referencing `super()`.

## Writing get_registered_attendees

*Round 1*
Our solution will be using a list comprehension generator expression to get a list of `User` model objects with `Registration.APPROVED` statuses for the given event so it can be compared to the `AttendanceRecord` model's `attendee` field. We will define a method on `AttendanceRecord` model called `get_registered_attendees()` to do so.

```
def get_registered_attendees(self):
    attendees = [registration.user for registration in self.attendancelog.event.registrations.filter(status=Registration.APPROVED)]
    return attendees
```

We query across to our `ForeignKey(AttendanceLog)` to get to its `ForeignKey(Event)`, then we use the `related_name='registrations'` on the Registration model's `Foreign Key(Event, related_name='registrations')` to query registration objects that are filtered on the `Registration.APPROVED` status.

That is a lot of querying and foreignkey traversals!

Pros
- It accomplishes the business logic

Cons
- In-memory comprehension, could be slow or eat up a lot of memory if the returned set is big (the event is a concert)
- We don't need the entire `User` (attendee) object, we only need to know if they have a registration with approved status
- Holding all the `User` objects eats up more memory

*Round 2*

We don't need the entire user object, if we could eliminate that and go by instead a unique identifier for the user and check if that unique identifier is in the returned `attendees` list we would be doing great. Hmmm.. what could that be? Oh right, the model's primary key. Django by default creates a dummy autoincrementing primary key integer field for every model and provides you with two aliases, `model.id` or `model.pk` to reference it. `id` is the actual name of the column, `pk` is a more general reference that can be to a custom primary key as well if one is specified on the model.

How do we throw out the `Registration` object, and all the `User` data? Django provides a wonderful method called `values()` which can be called on a `QuerySet`. `values()` takes a list of fields as parameters and by default returns a list of dictionaries for each row of the 'field': value. For instance `Event.registrations.all().values('user','status') would return:

```
[
    {'user': 1,
     'status': 'a'},
    {'user': 2,
     'status': 'a'},
     ...,
    {'user': n,
     'status': m}
]
```

For our purposes we only need the `user` field. The `user` field is technically a foreign key on the `User` model and so Django by default will return that `User` models primary key as the value with the field label. We could also specify `user_id` if we wanted the Django dummy `id` field explicitly. For now `user` will work well enough.

```
def get_registered_attendees(self):
    attendees = [registration['user'] for registration in self.attendancelog.event.registrations.filter(status=Registration.APPROVED).values('user')]
    return attendees
```

Now when our code checks to see if the attendee is part of the registered attendees, it is no longer holding objects in memory and comparing those objects, it is simply going to iterate through the list of integers and compare them.

We make a slight modification to the save code to get our `AttendanceRecord` model's `attendee` field's `pk` attribute instead to compare it.

```
def save(self, *args, **kwargs):
    attendees = self.get_registered_attendees()
    if self.attendee.pk not in attendees:
        raise ValidationError("Cannot register attendance for a user without an approved registration for the event.")
    super(AttendanceRecord, self).save(*args, **kwargs)
```

Pros:
- Solves the problem
- Uses integer comparisons instead of object comparisons
- Database query can be optimized to not store and return all the extra details, only the fields we need
- Django doesn't need to construct Model objects and place them in a QuerySet object before returning the result

Cons:
- Our values() returns a dictionary which we don't really need as we are only using the single field for comparison

*Round 3*

We have pretty neatly wrapped up our problem except the creation of a dictionary bothers me since we really only need a flat-list of values. We are right now reducing the dictionary in our list comprehension generator expression by calling `registration['user']` to get the value out of the dictionary. That is all well and good but we could also let Django handle it.

Another nifty Django method that complements `values()` is `values_list()`. The complementary method returns a list of tuples instead of a dictionary. Updating our code to use the `values_list('user')` method would return:

```
[
    (1,),
    (2,),
    ...,
    (n,)
]
```

Our list comprehension would change from
`[registration['user'] for registration in self.attendancelog.event.registrations.filter(status=Registration.APPROVED).values_list('user')]`
to
`[registration[0] for registration in self.attendancelog.event.registrations.filter(status=Registration.APPROVED).values_list('user')]`

We are still doing a list access and reducing the object here. However, Django provides a way to flatten this list of tuples in the case where we are only getting a single field from the `values_list()` query. If we add it in we get our nice flattened list of integers and updated statement.

```
[user for user in self.attendancelog.event.registrations.filter(status=Registration.APPROVED).values_list('user', flat=True)]

# returns
[1, 2, ... n]
```

We now have our final neatly wrapped solution. The last modification I am going to make is moving the `get_registered_attendees()` call to the `AttendanceLog` model itself. This saves us some verbosity in our generator expression and keeps the code more tightly coupled to the model that is using it.

```
class AttendanceLog(models.Model):
    date = models.DateField()
    event = models.ForeignKey(Event)

    def get_registered_attendees(self):
        return self.event.registrations.filter(status=Registration.APPROVED).values_list('user', flat=True)

class AttendanceRecord(models.Model):
    PRESENT = 'p'
    ABSENT = 'a'

    ATTENDANCE_STATUS_CHOICES = (
        (PRESENT, 'Present'),
        (ABSENT, 'Absent')
    )
    attendancelog = models.ForeignKey(AttendanceLog, related_name='records')
    attendee = models.ForeignKey(settings.AUTH_USER_MODEL)
    status = models.CharField(max_length=1, choices=ATTENDANCE_STATUS_CHOICES, default=ABSENT)

    def save(self, *args, **kwargs):
        attendees = self.get_registered_attendees()
        if self.attendee.pk not in attendees:
            raise ValidationError("Cannot register attendance for a user without an approved registration for the event.")
        super(AttendanceRecord, self).save(*args, **kwargs)

    def get_registered_attendees(self):
        return self.attendancelog.get_registered_users()
```

Now an `AttendanceLog` object **or** and `AttendanceRecord` object can call `get_registered_attendees()`.

## Epilogue

Django's `values()` and `values_list()` methods are valuable outside of grabbing a foreign key model's primary keys. They can be used on any query where you only need specific data and won't need to use the model's methods. Ultimately using this technique can save some time, memory, and processing power for your websites.
