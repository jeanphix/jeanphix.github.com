---
layout: post
lang: en
title: Reordering data extracted using postgres WITH RECURSIVE
tags:
- postgresql
---

Building python tree dict from postgres with recursive queries look quite simple, but it's not!

{% highlight python %}
def _nestify(entities) :
    """Transforms nested collection into tree list.
    """
    class entitydict(dict):
        """Dot access dict class.
        """
        def __getattr__(self, attr):
            return self.get(attr, None)
        __setattr__= dict.__setitem__
        __delattr__= dict.__delitem__

        def __init__(self, *args):
            self.children = []
            super(entitydict, self).__init__(*args)

    if not len(entities):
        return []

    root_level = entities[0].level
    nested = []
    depth = {}
    for entity in entities:
        entity = entitydict(entity)
        depth[entity.level - root_level]=entity
        if entity.level == root_level :
            #is root
            nested.append(entity)
        else :
            parent = depth[entity.level - root_level - 1]
            parent.children.append(entity)
    return nested
{% endhighlight %}
