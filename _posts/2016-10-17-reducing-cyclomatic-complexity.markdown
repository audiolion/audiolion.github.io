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