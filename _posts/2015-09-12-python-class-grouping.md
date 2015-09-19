---
layout: post
title: Metaclass to Group subclass and relationships to Django models
---


{% gist b04deddd5f7a7766c00b %}

Using this class as metaclass in baseclass, you can retrieve subclasses from baseclass. Below example is in python 3. For python 2 instead of `metaclass=Container` user `__metaclass__ = Container` inside class like a class variable.

{% highlight python %}
from .utils import Container


class Verb(metaclass=Container):
    def __str__(self):
        # this is needed for admin interface to work
        # for python 2 use `__unicode__`
        return self.name
    def __init__(self, *args, **kwargs):
        return None
    def execute(self, *args, **kwargs):
        return None

class Create(Verb):
    def execute(self, **kwargs):
        print("creating something")

class SendEmail(Verb):
    def execute(self, **kwargs):
        print("sending email")


In [1]: from myapp.verbs import Verb

In [2]: Verb.objects['create']
Out[2]: myapp.verbs.Create

In [3]: Verb.objects['create']().execute()
creating something

In [4]: Verb.objects['sendemail']().execute()
sending email

{% endhighlight %}

It is also possible to use this baseclass like foriegn key to django model by using a custom django model field.

{% gist 10b15a81b0d2a1e4e554 %}

{% highlight python %}
from django.db import models
from .utils import ContainerField
from .verbs import Verb
from django.contrib.auth.models import User


class UserVerb(models.Model):
    user = models.ForeignKey(User)
    verb = ContainerField(classvar=Verb, max_length=20)

In [1]: from myapp.models import UserVerb

In [2]: from myapp.verbs import Verb

In [3]: from django.contrib.auth.models import User

In [4]: me = User.objects.get()

In [5]: from myapp.verbs import SendEmail

In [6]: userverb = UserVerb.objects.create(user=me, verb=SendEmail.name)

In [7]: userverb = UserVerb.objects.get(verb=SendEmail.name)

In [8]: userverb.verb().execute()
sending email

{% endhighlight %}

Entire implementation is a little bizarre(or is it. I am not sure about that). I am thinking about implementing this in a project. Currently i am handling these situations by creating a new model for each baseclass to store name of each subclass and import that class by that name. This solution seems elegant than current one. Please share your thoughts and leave a comment. 