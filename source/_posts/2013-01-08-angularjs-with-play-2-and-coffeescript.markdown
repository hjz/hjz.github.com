---
layout: post
title: "Bootstrapping AngularJS with Play 2 and CoffeeScript"
date: 2013-01-08 13:55
comments: true
categories: [AngularJS, Scala, CoffeeScript, Play]
---

## Intro

[AngularJS](angularjs.org) is a javascript framework by Google that makes 2-way data binding between DOM elements and JS objects seamless.

I've always been repulsed by the amount of hacks and lack of abstractions endemic to front-end development. AngularJS follows the Model View ViewModel pattern which aims to separate view code from application logic. This is achiveved through a ViewModel which exposes and connects objects in the model to elements in the view. For AngularJS, this is the `$scope` object.

For instance, to bind an input field to a variable `searchModel`:

``` html
<input type="text" ng-model="searchModel" />

```

Any change to the input field is reflected in `$scope.searchModel`, and vice versa.

Another neat feature of AngularJS is ability to create *directives* that extend html. For instance you can create an `on-focus` directive that calls a method when an element is focused.

```
<input type="text" ng-model="searchModel" on-focus="searchModel = ''" />
```

The example above clears the input text when the element is focused. I'll touch more on creating directives in a future post.


{% gist 996818 %}


<!--more-->

{% video http://s3.imathis.com/video/zero-to-fancy-buttons.mp4 640 320 http://s3.imathis.com/video/zero-to-fancy-buttons.png %}

{% include_code test.js %}
