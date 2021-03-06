---
layout: post
lang: en
title: Building frontend web applications using backbone.js & brunch.io
tags:
- javascript
- backbone
- brunch
---

Last sunday I decided to grow up my javascript skills by playing with modern and cool libraries.

I planned to test:

* a language abstraction [coffeescript](http://coffeescript.org/)
* a frontend MVC framework that deals with RESTful APIs [spine.js](http://spinejs.com/), [backbone.js](http://documentcloud.github.com/backbone/)
* a template engine (eco, [coffeekup](http://coffeekup.org/))

First of all, I tried spine.js that is a frontend framework that provides:

* bootstrapping tools (project generation...)
* views
* models
* routing
* testing tools
* ...

But I trashed it cause of hard jQuery dependency (&lt;troll>I hate jQuery&lt;/troll>).

So, next I discovered [brunch.io](http://brunch.io) that is a backbone.js projects bootstrapper that does quite the same things as spine.js with the possibility to replace jQuery by [zepto.js](http://zeptojs.com/).

The stack here is composed of:

* underscore.js
* backbone.js
* jasmine
* coffeescript
* css tools (boilerplate...)
* brunch.io (project configuration and class bootsrapping)

## Test project

My favorite RESTful API to test new client is github's one, cause it has been well designed and [CORS](http://developer.github.com/v3/#cross-origin-resource-sharing) requests are allowed.

_If you planned to write a client side application on top of that kind of library, make sure the service is consumable via XHR, otherwise you'll have to make a server side "reverse proxy application", and then doing the job client side will make no sence (IMHO)._

Lets start the project:

{% highlight bash %}
$ npm install -g brunch
$ brunch new issues
$ cd issues
{% endhighlight %}

Now directory tree and configs has been created with default assets.

{% highlight bash %}
$ brunch watch --server
{% endhighlight %}

runs an express.js instance that delivers compiled files (just open http://localhost:3000 in your favorite browser).

## Backbone main concepts

### Routers

Backbone routers is the navigation layer that links UI actions to the views.

Sample router:

{% highlight coffeescript %}
class exports.MainRouter extends Backbone.Router
  routes :
    ':username/:repository': 'repository'
    '': 'repository'

  repository: (username, repository) ->
    app.repositoryView.render(username, repository).el
{% endhighlight %}

### Views

Views are controller classes that deal with your business models and the UI. Each view as an `el` member that is the DOM element where rendering will be displayed.

Starting a view via brunch.io:

{% highlight bash %}
$ brunch generate view issue_list_view
{% endhighlight %}

Then with a piece of logic:

{% highlight coffeescript %}
{IssueView} = require 'views/issue_view'
{Issues} = require 'collections/issues'

issueListTemplate = require './templates/issue_list'


class exports.IssueListView extends Backbone.View
  el: 'section:first'

  initialize: (repository) ->
    @issues = new Issues
    @issues.url = repository.get 'url'
    @issues.url+= '/issues'
    @issues.bind 'reset', @addAll

  render: =>
    issues = @issues.fetch()
    @$(@el).html issueListTemplate
    this

  addOne: (issue) =>
    view = new IssueView model: issue
    $(@el).find('#issues tbody:first').append view.render().el

  addAll: =>
    @issues.each @addOne
{% endhighlight %}

### Models

A model is a business entity, that is linked to an HTTP ressource using the _url_ attribute.

_May be persisted and fetched from local storage to._

{% highlight bash %}
$ brunch generate model issue
{% endhighlight %}

Sample:

{% highlight coffeescript %}
class exports.Issue extends Backbone.Model
  ul: '/path/to/issue/ressource'

  defaults:
    title: 'New issue'
    body: null
{% endhighlight %}

### Collections

Collection is a set of model objects, as models, they may be linked to an HTTP ressource.

That's all for basics. For now, my application only lists issues for a given project. I'll maybe add more features (like create / update / delete issues) later. You'll find a live demo [here](http://jeanphix.me/brunch-github-issues/#brunch/brunch).

See you.


