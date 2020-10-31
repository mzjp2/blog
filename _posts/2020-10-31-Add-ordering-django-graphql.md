---
title: "Add ordering to your Django-Graphene GraphQL API :arrow_down:"
layout: post
date: 2020-10-31 20:00
headerImage: false
tag:
- python
- django
- graphql

category: blog
author: zainpatel
description: "Django-Graphene provides an OrderingFilter that doesn't really work if you use any other filtering features with Django. This post walks through how I worked around this and implemented an extensible ordering mechanism on my Django GraphQL API, including the ability to filter on custom fields (as opposed to only model fields)"
---

## The problem

The canonical [Djanago-Graphene documentation on ordering](https://docs.graphene-python.org/projects/django/en/latest/filtering/#ordering) points you towards using the `OrderingFilter` on a custom `FilterSet` from `django_filter` to implement ordering on your API, so that you can do things like:


```
query {
    posts(orderBy: "-createdAt") {
        title
    }
}
```

to get post titles ordered in descending order of when they were created. This works okay, as long as you don't intend to use the filtering mechanism in `django-graphene`/`django-filter`: where you can do:


```python
class PostNode(DjangoObjectType):
    class Meta:
        fields = {
            "title": ["exact", "startswith", "icontains"]
        }
```

because as soon as you follow the documentation from `django-graphene` and provide the custom filterset class, you gain ordering via the `orderBy`, but lose the default `FilterSet` that `django-graphene` builds for your `PostNode` type, which contains the nice `title__Exact`, `title__Icontains`, etc... filters on it.

## The solution

To fix this, I built a custom connection field, inspired from [several stackoverflow answers](https://stackoverflow.com/questions/57478464/django-graphene-relay-order-by-orderingfilter) inheriting from `DjangoFilterConnectionField` that you likely already use:


```python
from graphene_django.filter import DjangoFilterConnectionField
from graphene.utils.str_converters import to_snake_case


class OrderedDjangoFilterConnectionField(DjangoFilterConnectionField):
    @classmethod
    def resolve_queryset(
        cls, connection, iterable, info, args, filtering_args, filterset_class
    ):
        qs = super().resolve_queryset(
            connection, iterable, info, args, filtering_args, filterset_class
        )
        order = args.get("orderBy", None)
        if order:
            if isinstance(order, str):
                snake_order = to_snake_case(order)
            else:
                snake_order = [to_snake_case(o) for o in order]

            # annotate counts for ordering
            for order_arg in snake_order:
                order_arg = order_arg.lstrip("-")
                annotation_name = f"annotate_{order_arg}"
                annotation_method = getattr(qs, annotation_name, None)
                if annotation_method:
                    qs = annotation_method()

            # override the default distinct parameters
            # as they might differ from the order_by params
            qs = qs.order_by(*snake_order).distinct()

        return qs
```

and you use it like so in your query schema:

```python
class PostQuery:
    posts = OrderedDjangoFilterConnectionField(
        PostNode,
        orderBy=graphene=graphene.List(of_type=graphene.String)
        )
```

This differs from the StackOverflow answer in that it lets you sort by custom fields that you may have implemented on your GraphQL API, but not on your model (for example, in my case, it was the dowmload count of a post - which isn't stored on the model). Let's walk through what this does.

When the GraphQL query is sent _without_ the `orderBy` argument then the `if order` black is never entered and we return the `queryset` as is from `DjangoFilterConnectionField`'s `resolve_queryset` method.

Then the GraphQL query is sent _with_ the `orderBy` argument (say, set to `fieldName`) then we convert this to snake case (`field_name`). If the value is instead a list, then we do this for each value in the list.

We then look at the model manager for a method called `annotate_field_name`. If this method exists, we call it. If it doesn't, then we continue. We return the `queryset` got from `DjangoFilterConnectionField`'s `resolve_queryset` method with the `order_by` method called on it with the snake-cased values from the GraphQL API. We also call `distinct` because of various quirks with ordering and `ManyToMany` fields with Django.

This works as you would expect if you were ordering by a model field, say the post title, but the real pickle is in ordering on a custom field you've created on your GraphQL API, but not on the model, in my case the post download count. This is what the `annotate_field_name` method is for. My Post Manager looks like:


```python
class PostManager(models.Manager):
    def annotate_download_count(self):
        return self.annotate(download_count=models.Count("downloads"))
```

which annotates my queryset with a `download_count` field, that I can then order against.
