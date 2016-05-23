---
layout: post
lang: en
title: Designing trees in mongodb using mongodm
tags:
- mongodb
- mongodm
---

## why mongodm?

[mongodm](https://github.com/jeanphix/mongodm) is a small API on top of pymongo that allowed to create user defined mongodb object documents.

All collection's queries are still made using pymongo, mongodm just provide an nice way to create data structures and validation schemas.

The first reason that bring me away from the great [mongoengine](http://mongoengine.org/) is that's there's no way to easily manage recursive trees.

Here are two [tree pattern](http://www.mongodb.org/display/DOCS/Trees+in+MongoDB) implementation using this small library.

## Full Tree in Single Document

code sample:

{% highlight python %}
from mongodm.connection import Connection
from mongodm.fields import StringField, ListField
from mongodm.document import Document, EmbeddedDocument


connection = Connection('localhost', 27017)
db = connection['test-database']


class TreeNode(EmbeddedDocument):
    label = StringField()
    children = ListField('TreeNode')


class Tree(Document):
    __collection__ = 'trees'
    __db__ = db
    label = StringField()
    children = ListField('TreeNode')


tree = Tree()
tree.label = 'root'


first_child = TreeNode()
first_child.label = 'node 1'
tree.children.append(first_child)


second_child = TreeNode()
second_child.label = 'node 1.1'
first_child.children.append(second_child)


third_child = TreeNode()
third_child.label = 'node 1.2'
first_child.children.append(third_child)


Tree.collection().insert(tree)
{% endhighlight %}

mongodb data:

{% highlight javascript %}
> db.trees.find()
{ "_id" : ObjectId("4caebee27478f70e7d000000"), "children" : [
        {
                "children" : [
                        {
                                "children" : [ ],
                                "label" : "node 1.1"
                        },
                        {
                                "children" : [ ],
                                "label" : "node 1.2"
                        }
                ],
                "label" : "node 1"
        }
], "label" : "root" }
{% endhighlight %}

## Materialized paths pattern

code sample:

{% highlight python %}
from mongodm.connection import Connection
from mongodm.fields import StringField, ReferenceField
from mongodm.ext.tree import TreeDocument


connection = Connection('localhost', 27017)
db = connection['test-database']


class TreeNode(TreeDocument):
    __collection__ = 'nodes'
    __db__ = db
    label = StringField()
    parent = ReferenceField('TreeNode')


TreeNode.collection().ensure_index('path') #don't forget to create index on path field


root_node = TreeNode()
root_node.label = 'root'
TreeNode.collection().insert(root_node)


first_child = TreeNode()
first_child.label = 'node 1'
first_child.parent = root_node
TreeNode.collection().insert(first_child)


second_child = TreeNode()
second_child.label = 'node 1.1'
second_child.parent = first_child
TreeNode.collection().insert(second_child)


third_child = TreeNode()
third_child.label = 'node 1.2'
third_child.parent = first_child
TreeNode.collection().insert(third_child)
{% endhighlight %}

mongodb data:

{% highlight javascript %}
> db.nodes.find()
{ "_id" : ObjectId("4caeba3b7478f70cb9000000"), "parent" : null, "level" : 0, "label" : "root", "priority" : 0, "path" : "" }
{ "_id" : ObjectId("4caeba3b7478f70cb9000001"), "parent" : { "$ref" : "nodes", "$id" : ObjectId("4caeba3b7478f70cb9000000") }, "level" : 1, "label" : "node 1", "priority" : 0, "path" : "4caeba3b7478f70cb9000000," }
{ "_id" : ObjectId("4caeba3b7478f70cb9000002"), "parent" : { "$ref" : "nodes", "$id" : ObjectId("4caeba3b7478f70cb9000001") }, "level" : 2, "label" : "node 1.1", "priority" : 0, "path" : "4caeba3b7478f70cb9000000,4caeba3b7478f70cb9000001," }
{ "_id" : ObjectId("4caeba3b7478f70cb9000003"), "parent" : { "$ref" : "nodes", "$id" : ObjectId("4caeba3b7478f70cb9000001") }, "level" : 2, "label" : "node 1.2", "priority" : 0, "path" : "4caeba3b7478f70cb9000000,4caeba3b7478f70cb9000001," }
{% endhighlight %}

you can easily get back children using TreeDocument.children property.

I'm working on mongodm on my time off for a personnal project (this is one of my first python developments), so feel free to give me your feed back.
