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

to get post titles ordered in descending order of when they were created. This works okay, as long as you don't intend to use the filtering mechanism in `django-graphene`/`django-filter`, where you can specify django-style filters that are converted automatically to GraphQL arguments for filtering:


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
class PostQuery(graphene.ObjectType):
    posts = OrderedDjangoFilterConnectionField(
        PostNode,
        orderBy=graphene=graphene.List(of_type=graphene.String)
    )
```

This differs from the StackOverflow answer in that it lets you sort by custom fields that you may have implemented on your GraphQL API, but not on your model (in my case, it was the dowmload count of a post - which isn't stored on the model) as long as you build a relevant annotation method for it. Let's walk through what this does:

* We call the parent `DjangoFilterConnectionField`'s `resolve_queryset` method to get the queryset with all the magic filtering taken care of already, this ensures we don't have to do any work ourselves or re-build any logic.
* If the GraphQL query is sent _without_ the `orderBy` argument then we never enter the  `if order` block and instantly return the `queryset` as-is from above.
* If the GraphQL query is sent _with_ the `orderBy` argument (say `fieldName`) then we convert this to snake case (`field_name`) and do the following (note: if the value is instead a list, we convert each value to snake case and do the following to each value)
* We look at the queryset for a method called `annotate_{field_name}` on it
    * If this method exists, we call it, expecting the result to be the same queryset-type, but with an additional annotation on it called `field_name` (this is what lets us do the ordering) and replace the queryset with the newly-annotated queryset.
    * If the method doesn't exist, then we leave the queryset as-is and do continue with the rest of the logic
* We call the `order_by` argument on the queryset we have with the values from the `orderBy` snake-cased arguments provided, this works on native db fields as normal with Django and also works with any graphene fields you've built that have custom resolvers as long as you write a mathching annotation method on the queryset for that field.
* We also call `distinct` on the queryset to ensure we get sensical results, given some of Django's quirks with `order_by` and `ManyToMany` fields.

This works as you would expect if you were ordering by a model field, say the post title, but the real beauty is in ordering on a custom field you've created on your GraphQL API, but not on the model, in my case the post download count. This is what the `annotate_field_name` method is for. My Post Manager looks like:


```python
class PostManager(models.Manager):
    def annotate_download_count(self):
        return self.annotate(download_count=models.Count("downloads"))
```

which annotates my queryset with a `download_count` field, that I can then order against.

You'd also ideally re-use this annotation within your GraphQL resolver rather than writing any new logic. For example:

```python
class PostNode:
    download_count = graphene.Int()

    @staticmethod
    def resolve_download_count(root: Post) -> int:
        return (
            Post.objects.filter(id=root.id)
            .annotate_download_count()
            .get(id=root.id)
            .download_count
        )

```

or if you want to optimise things when resolving multiple posts (e.g to list them all):

```python
# schema.py

from .types import PostNode

class PostQuery(graphene.ObjectType):
    posts = OrderedDjangoFilterConnectionField(PostNode)

    @staticmethod
    def resolve_posts():
        return Post.objects.all().annotate_download_count()

# types.py

class PostNode:
    download_count = graphene.Int()

    @staticmethod
    def resolve_download_count(root: Post) -> int:
        return root.download_count # has been annotated from the `posts` resolver
```
