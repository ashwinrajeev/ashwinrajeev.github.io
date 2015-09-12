---
layout: post
title: Grouping classes and relationships to Django models
---

Recently i came across few use cases where need to group python classes. Let's look into case where i need a group for serializers. All of the serializer have common methods and attributes. So i choose to create a baseclass `Serializer` and inherit all other serializer(CSVSerilizer, JSONSerializer) from it. But what i really nead was `Serializer` objects with different methods. To do that i could pass these methods as arguments to `__init__` and add it to object's `__dict__`. But the code will be less readable than inheritance. I decided to use metaclass so i can write inheritive classes with baseclass with a list of each subclass object.

{% gist b04deddd5f7a7766c00b %}

This class needs to be set as metaclass for baseclass. This metaclass create object of subclass instead of class as store it in baseclass

```python
class Serializer(metaclass=CollectionMeta):
    def __str__(self):
        return self.name
    def serialize(self):
        raise NotImplementedError

class Json(Serializer):
    def serialize(self, data):
        return json.dumps(data)


In [3]: Serializer.objects.get('Json')
Out[3]: <app.serializer.Json at 0xb668184c>

In [4]: Serializer.objects['Json'].serialize({"name": "ashwin"})
Out[4]: '{"name": "ashwin"}'
```

It is also possible to use this baseclass like foriegn key to django model by using a custom django model field.

{% gist 10b15a81b0d2a1e4e554 %}

```python
class UserSerializer(models.Model):
    user = models.ForeignKey(User)
    serializer = ContainerField(classvar=Serializer, max_length=20)


In [4]: s = UserSerializer.objects.get(id=1)

In [5]: s.serializer
Out[5]: <app.serializer.Json at 0xb6654fcc>
In [6]: s.serializer.serialize({"name": s.user.username})
Out[6]: '{"name": "ashwin"}'
```