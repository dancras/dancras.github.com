---
layout: post
title: Using mocha with jspm to write tests in ES6
date: 2015-12-31 14:30:00
---

The siren call of ES6 and everything it brings has gone largely unnoticed by myself so far due to [other interesting technologies](http://elm-lang.org/), however last night I got the urge to experiment with it. I started by reading some comparisons of module loaders and concluded that SystemJS was the way to go. When accompanied by its sister project [jspm](http://jspm.io/) it seems to be an impressive option for managing packages and transpiling ES6 code. It turns out that getting mocha and jspm to play nice is a little tricky so I never did get round to experimenting with ES6 after all. However I did learn a few new things and ended up with a pretty neat solution for working with these technologies so I thought I'd revive my dead blog by sharing them.

 > TL;DR: The project combining these technologies can be found [on github](https://github.com/dancras/es6-jspm-mocha-chai) with fairly detailed information on how it works.

The first distraction was mocha's own compilers functionality. It has support for babel, which is great, but unfortunately if you let mocha handle the transpiling then it has no integration with the SystemJS module loader and consequently no knowledge of the dependencies managed by jspm. The downside of this is that it requires ugly workarounds to load those dependencies:

{% highlight js %}
describe('Greeter', () => {
    
    let Greeter;

    before(() => {
        return System.import('project/Greeter')
            .then((GreeterModule) => {
                Greeter = GreeterModule['default'];
            });
    });

});
{% endhighlight %}

An ideal solution would defer the module loading to SystemJS as early as possible since we are using it as our primary module loader. A simple solution seemed to be to add a main test file as the entry point for mocha, then import the test suite from there using SystemJS.

{% highlight js %}
// test/main.js
let System = require('jspm');

System.import('test/all');
{% endhighlight %}

This was very confusing for a while as the call to `System.import` was seemingly being ignored completely, with no exception and no promise resolution. It turns out that's exactly what was happening. Reading through the documentation for mocha I stumbled upon the "Delayed Root Suite" section which describes how to perform asynchronous operations before running any suites. Following the instructions I finally had my first failing test written in ES6.

Take a look at the solution [on github](https://github.com/dancras/es6-jspm-mocha-chai) with. Suggestions and improvements welcome.
