---
layout: post
lang: en
title: Faking geolocation javascript API using Casper.js
tags:
- casperjs
- javascript
---

Testing frontend javascript map behaviours and geolocation stuff is not an easy thing to do.

[Casper.js](https://n1k0.github.com/casperjs/), that come on top of Phantom.js, is a great tool for quickly write functionnal tests, including frontend javascript behaviours cover as it provides an API to access real DOM state.

As QtWebKit doesn't deal with javascript geolocation API, we need to write a fake location service in order to test user position changes.

{% highlight javascript %}
var addFakeGeolocation = function(self, latitude, longitude) {
    self.evaluate(function() {
        window.navigator.geolocation = function() {
            var pub = {};
            var current_pos = {
                coords: {
                    latitude: window.__casper_params__.latitude,
                    longitude: window.__casper_params__.longitude
                }
            };
            pub.getCurrentPosition = function(locationCallback,errorCallback) {
                locationCallback(current_pos);
            };
            return pub;
        }();
    }, { latitude: latitude, longitude: longitude });
};
{% endhighlight %}

will add the expected geolocation faked provider.

Now it's as easy as ever to test user move:

{% highlight javascript %}
casper.then(function(self) {
    addFakeGeolocation(self, 12, 35);
    self.assertEval(function() {
        // Check expected changes
    });
});
{% endhighlight %}
