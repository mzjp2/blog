---
title: "Post mortem debugging sessions with Kedro hooks :leftwards_arrow_with_hook:"
layout: post
date: 2020-06-05 19:00
headerImage: false
tag:
- python
- quantumblack
- kedro

category: blog
author: zainpatel
description: "A technical dive into using (i)pdb's post mortem debugging to make debugging Kedro nodes easier"
---

Debugging `kedro` nodes can be hard in certain environments. If you aren't doing your `kedro run` within an IDE that supports debugging, either because it's being run in some other (production?) environment, you prefer the CLI or any other reason, you lose access to being able to jump into the `node`'s scope and debug the exception there.

Inserting `print` statements or even having to re-run your pipeline within a debugging session can be frustrating. Entering a `node` can have a lot of overhead, you need to load the data into the node, which often takes some time. This rings especially true if you've been a good disk I/O citizen and making use of passing data between nodes in memory for data that you don't need to persist. You now have to run a bunch of other nodes, starting from one that has access to persisted data before you can enter the failing node with the correct data.

I've had this pointed out to me a few times and haven't really been able to come up with a good solution to the dilemna till now. [Kedro 0.16 is out!](https://medium.com/quantumblack/new-in-kedro-this-month-991a1fb50cb4) with support for [Hooks](https://kedro.readthedocs.io/en/stable/04_user_guide/15_hooks.html), which opens up a whole new world of functionality.

## What is a Kedro Hook anyway?

A `hook` in general is this notion of injecting additional functionality in defined areas of your execution. Kind of like a `callback`, except more generic. The way that `kedro` has implemented hooks is that it defines certain hooks in certain places of its `run` execution (`after_catalog_created`, `after_node_run`, `on_node_error`, etc...) that let users inject custom functionality at the specific location within the execution of the `kedro run` session. I won't dither too much on what a `kedro` Hook is or how they work, and will leave that to the Kedro documentation and blog posts.

## What does this have to do with debugging?

We're particularly interested in the `on_node_error` hook, which is executed/called when an error is raised within a `node`. This hook here lets us inject some custom functionality between the event of the `node` raising an `Exception` and `kedro` bubbling this `Exception` back up the stack.

Now, if you're anything like me, you've realised that we can throw in a `pdb` debugging session here, with `pdb.set_trace()` or if you're in Python 3.7 or higher land, the convenient `breakpoint()` builtin that drops you into a `pdb` debugging session.

## Cool! Was that really worth a blog post?

Ah, but it doesn't end there. Because of the way that the hook is implemented, the `pdb` debugging session starts after having left the local scope within the node and exists in a pretty much blank transient scope before bubbling this error up.

This means that you enter the debugging session fine, but there's not much to do within it. You don't have access to the loaded `data`, nor any of the local variables defined before the `Exception` was raised.

## So... was reading this a waste of my time?

No! This is where it gets good. `pdb` has this notion of [`post_mortem` debugging](https://docs.python.org/3/library/pdb.html#pdb.post_mortem) which essentialy lets the Python interprert rewind back in time, and drop you back in the local scope where the underlying `Exception` was raised.

This initially sounded a little like black magic to me, but on a deeper dive, actually makes sense. The `post_mortem` actually doesn't care about the `Exception`, it cares about the `traceback`. The `traceback` is a Python object, that on a first glance doesn't look like all that much:

```python
>>> import sys
>>> try:
...   raise ValueError
... except:
...   _, _, tb = sys.exc_info()
...
>>> print(tb)
<traceback object at 0x108e98af0>
>>> dir(tb)
['tb_frame', 'tb_lasti', 'tb_lineno', 'tb_next']
```

but on a deeper look, all the juicy information is contained in `tb.tb_frame`, which contains everything from the `code` in `tb.tb_frame.f_code` to the `globals` in `tb.tb_frame.f_globals` present in the traceback object.

This is actually what lets Python do all those helpful tracebacks when an error is occured, where it highlights the line the exception occured on (`tb.tb_frame.f_lineno`), the file it came from, and all the other pertinent information.

## Where's my Hook?

So, putting all the above together, we get what we came here for. Since the `Exception` exists in the scope that the hook is executed in, we can pull the `traceback` from this exception, format it and print it to `stdout` so the user has all the details about the error raised. We then provide the traceback object to `pdb.post_mortem`, which looks into the `tb_frame` attribute and recreates the local scope. This means that you have access to the loaded data and all the other goodies within your `node'`s local scope as soon as an exception is raised:

<script src="https://gist.github.com/mzjp2/53eb88c7d388b2f712672ec458780051.js"></script>

![](https://i.imgur.com/xJjmGSg.gif)
