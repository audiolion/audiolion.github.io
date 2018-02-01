---
layout: post
title:  "DRF with Marshmallow Serializers for Fun and Profit"
date:   2018-01-31 20:14:05
author: Ryan Castner
categories: Django
tags: drf marshmallow django-rest-framework django serializers
cover:  "/assets/marshmallows.jpg"
---

## The Problem

[Django Rest Framework (DRF)](https://django-rest-framework.org) is a fantastic tool for API development in Django. However, it does have some downsides. Serializers are an especially heavy part of the framework. Rightfully so, they do quite a bit of magic to make API development lightning quick by automatically pulling fields from your Django Models and applying all the database validation. Another downside is that you are forced to declare what fields you want to serializer in the class definition. This typically means you include everything if you want to reuse the same endpoint, or you have to create separate serializers for different views.

```
# everything including the kitchen sink
class BlogSerializer(serializers.ModelSerializer):
    class Meta:
        model = Blog
        fields = '__all__'

# oops I don't want author private notes and editor notes
# passed to regular consumers of the API endpoint
class PublicBlogSerializer(serializers.ModelSerializer):
    class Meta:
        model = Blog
        exclude = ('private_notes', 'editor_notes', )

# ...but now I need a serializer for my editor that includes the editor_notes
class EditorBlogSerializer(serializers.ModelSerializer):
    class Meta:
        model = Blog
        exclude = ('private_notes', )

```

Wow! What a mess! Three serializers for essentially the same exact thing just +/- a few fields.

## The Solution

Introducing [Marshmallow](https://marshmallow.readthedocs.io/en/latest/index.html)! Marshmallow serializers are agnostic to what they are serializing, but in the case of using them with DRF we do need a compatibility layer so they work with DRF Views. Luckily, [django-rest-marshmallow](https://github.com/marshmallow-code/django-rest-marshmallow) is just a pip install away. Afterwards we can define our serializer like this:

```
from rest_marshmallow import Schema, fields

class BlogSchema(Schema):
    id = fields.Int()
    title = fields.Str(required=True)
    subtitle = fields.Str()
    body = fields.Str(required=True)
    author_id = fields.Int(required=True, load_from='author', dump_to='author')
    editor_id = fields.Int(load_from='editor', dump_to='editor')
    private_notes = fields.Str()
    editor_notes = fields.Str()

```

So we have to do a bit of work up front and actually define every field and the validators we want to apply to them. If you want `django-rest-marshmallow` to handle this automatically, you can show some support or contribute to the [open issue](https://github.com/marshmallow-code/django-rest-marshmallow/issues/15). But once this is done, we can now use the beautifully simple `only` or `exclude` fields on the schema. We can override the `get_serializer` in our DRF View to set this up.

```

class BlogDetail(generics.RetrieveAPIView):
    serializer_class = BlogSchema
    queryset = Blog.objects.select_related('author', 'editor')

    def get_serializer(self, *args, **kwargs):
        serializer_class = self.get_serializer_class()
        is_editor = self.request.user.groups.filter(name='editors')
        if self.request.user.is_superuser:
            # superuser gets everything
            serializer = serializer_class(*args, **kwargs)
        elif is_editor:
            # we exclude the private_notes from editors
            serializer = serializer_class(exclude=('private_notes'), *args, **kwargs)
        else:
            # regular user doesn't have access to private_notes
            # or the editor_notes
            serializer = serializer_class(
                exclude=('private_notes', 'editor_notes'), *args, **kwargs)
        # since we overrode the view we want to make sure we call this
        kwargs['context'] = self.get_serializer_context()
        return serializer

```

One serializer, dynamically modified to exclude fields based on the requesting user's permissions. Happy Marshmallows!
