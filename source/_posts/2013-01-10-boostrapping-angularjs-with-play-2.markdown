---
layout: post
title: "Bootstrapping AngularJS with Play 2"
date: 2013-01-10 13:55
comments: true
categories: [AngularJS, Scala, CoffeeScript, Play]
---

## AngularJS Primer

[AngularJS](http://angularjs.org) is a javascript framework by Google that makes 2-way data binding between DOM elements and JS objects seamless.

I've always been repulsed by the amount of hacks and lack of abstractions prevalent in front-end development. AngularJS follows the Model View ViewModel pattern which aims to separate view code from application logic. This is achiveved through a ViewModel which exposes and connects objects in the model to elements in the view. For AngularJS, this is the `$scope` object.

For example, to bind an input field to a variable `searchModel` on the scope:

```html
<input type="text" ng-model="searchModel">

```

Any change to the input field is reflected in `$scope.searchModel` and vice versa.

Another neat feature of AngularJS is the ability to create [directives](http://docs.angularjs.org/guide/directive) that extend html. For instance you can create an `on-focus` directive that calls a method when an element is focused.

```
<input type="text" ng-model="searchModel" on-focus="searchModel = ''">
```

The example above clears the input text when the element is focused. I'll touch more on creating directives in a future post.


## Integration with Play 2 framework


### Bootstrapping AngularJS

<!--more-->

Play is a rails inspired MVC framework built on the JVM. It has APIs for both Scala and Java and uses class reloading to quickly surface changes in the browser. One feature is the templating system that uses Scala. A html template gets compiled to a Scala object and can be called from controllers, with the input type checked.

One pattern I've employed is bootstrapping AngularJS in the template:

```html items.scala.html
@(items: Seq[JSItem])

@import com.codahale.jerkson.Json

<script>
var items = @Html(Json.generate(items));
</script>

@main("Items Page") {
<div class="container" ng-controller="ItemsCtrl">
   There are {{items.length}} items
</div>
}
```

Then in the controller on the client:

```coffeescript controllers.cofffee
ItemsCtrl = ($scope) ->
  $scope.items = items

```


Notice the input to template is a sequence of JSItems, which is a case class. This is just a compact version of the Item model that get's passed in to the client. We generate the items as json using jerkson which creates objects with the same fields as those in a case class. `@Html` tells Play not to escape the output and render it literally.

### Updating the model on the server

Typically, we want the ability to update the model on the client side and update the server,

```coffeescript controllers.cofffee
ItemsCtrl = ($scope, $http) ->
  $scope.items = items

  $scope.updateItems = ->
    $http.post '/items', $scope.items
      .success ->
        # Success callback
      .error (data, status)->
        $console.log "Failed with status #{status}, data: #{status}"

```

The updateItems method uses AngularJS $http service which HTTP calls in [promises](http://docs.angularjs.org/api/ng.$q). It is similar in concept to a `scala.concurrent.Future` but with a much simpiler api. You can actually assign AngularJS promises to variables bound in the view, and the view will get updated asynchronously once the promise is resolved.

On the server side, we can simply parse the Json back to the case class that it rendered from:

```scala routes
POST /items Application.updateItems
```

```scala Application.scala
import com.codahale.jerkson.Json

object Application extends Controller {
  def updateItems = Action { request =>
    val json = request.body.asJson.get.toString()
    val items = Jsonparse[Seq[Item]](json)
    ...
  }
}
```

### How to make all this easier

With Scala 2.10 macros, you could imagine automatically creating the JS case classes from a subset of fields in the database Model and generating customizable CRUD controllers.
