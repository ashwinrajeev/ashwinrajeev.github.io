---
layout: post
title: Grouping classes and relationships to Django models
---

Recently i came across few use cases where need to link python classes to database models in django project. I solved this issue by creating a new model for each group of class

```python
class JsonSerializer(object):
   ...

class Serializer(models.Model):
   name = models.CharField(max_length=10)
   classpath = models.CharField(max_length=100)

   def get_class(self):
   # import and return class

Serializer.objects.get_or_create(name="json", classpath="serializer.JsonSerializer")
```
Accessing the serializer

```python
>>> serializer = Serializer.objects.get("json").get_class()()
>>> serializer.serialize(data)
```
But with this solution i need either fixtures or need to call get_or_create for each Serializer class. Better way will be group all class with same base class instead of creating a model for them.

```python
class CollectionMeta(type):
    def __new__(cls, name, bases, dct):
        # filter base classes with metaclass CollectionMeta
        parents = [b for b in bases if isinstance(b, CollectionMeta)]
        if not parents:  # is baseclass
            cls = super(CollectionMeta, cls).__new__(cls, name, bases, dct)
            cls.objects = {}
            cls.name = name
        else:  # subclass
            dct['name'] = name
            # instead of creating class, object of class is created
            obj = super(CollectionMeta, cls).__new__(cls, name, bases, dct)()
            parents[0].objects[name] = obj
        return cls


class Serializer(metaclass=CollectionMeta):
    ''' for python 2 use below commented format
    class Serializer(object):
         __metaclass__ = CollectionMeta'''
    def serialize(self):
        raise NotImplementedError
    # reqired for admin view to work
    def __str__(self):  # __unicode__ for python 2
        return self.name

class Json(Serializer):
    def serialize(self, data):
        return json.dumps(data)

```
Accessing the serializer

```python
>>> Serializer.objects['Json'].serialize(data)
```
But previous solution's serializer can be used as foreign key in other models. Using Custom model field same functionality can be simulated.

```python
class CollectionField(models.CharField):
    def __init__(self, classvar=None, **kwargs):
        if classvar:
            self.classvar = classvar
        # models.Field.__init__(self, **kwargs)
        super(CollectionField, self).__init__(**kwargs)

    def from_db_value(self, value, expression, connection, context):
        if value is not None and not isinstance(value, self.classvar):
            return self.classvar.objects.get(value)
        return value

    @property
    def choices(self):
        return [(i.name, i.name) for i in self.classvar.objects.values()]

    def get_prep_value(self, value):
        if isinstance(value, self.classvar):
            value = value.name
        return super(CollectionField, self).get_prep_value(value)

    def deconstruct(self):
        print "deconstruct"
        name, path, args, kwargs = super(CollectionField, self).deconstruct()
        if hasattr(self, 'classvar'):
            kwargs['classvar'] = self.classvar
        return name, path, args, kwargs
```
