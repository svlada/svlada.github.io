---
title: Require.js optimizer
description: This is RequireJS optimizer tutorial, second one in the RequireJs series.
date: 2013-11-25
tags:
  - javascript
  - require-js
layout: layouts/post.njk
permalink: "require-js-optimization-part2/index.html"
---

This article will help you understand the RequireJS optimizer. An introductory article about RequireJS dependency management can be found in [part 1](/require-js-dependency-management-part1/) of the series.
 
## Table of contents

1. <a href="#t0">Introduction</a>
2. <a href="#t1">Require.js optimizer</a>
3. <a href="#t2">Require.js optimizer dump dependencies to single file</a>
 
### <a name="t0" id="t0">Introduction</a>
 
James Burke [recommends](http://requirejs.org/docs/optimization.html) using node.js for optimizing and building your code. Make sure you have node.js installed on your machine. You can download node.js from [here](http://nodejs.org/#download).
 
### <a name="t1" id="t0">Require.js optimizer</a>

Create your build script file on following location `[webapp/build/build.js]`

```js
|-[wepapp]
|--- [build]
|----- r.js
|----- build.js
|----- build.single.js
|--- [js]
|------ [app]
|------ [lib]
|--------- jquery.js
|----- app.js
|----- app-built.js*
|-index.html
|-readme.md
```
 
Build your application with node.js and r.js executing the following command
 
```js
node r.js -o build.js
```
 
The following are examples of how to build an application with RequireJS.
 
**Example 1: Optimize modules**

Main module app.js has many dependencies and we want to bundle all that dependencies into single file. Mark modules for optimization in modules array of configuration object inside `build.js`. First example will contain only one module for optimization `[webapp/js/app.js]`.
 
Paste the following code to your `build.js` file.
 
```js
{
    baseUrl: "js/lib",
    appDir: "..",
    dir: "dist",

    modules: [
        { name: "app" }
    ],

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
**appDir** - Telling us where webapp root directory is located relative to `build.js` script.<br/>
**dir** - Output directory relative to `build.js`
 
 Console output of `build.js`
 
```js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/build/r.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/category.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/item.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/specialItem.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/jquery.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/require.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/text.js

js/app.js
----------------
js/lib/jquery.js
js/app/category/category.js
js/app/category/item.js
js/app/category/specialItem.js
js/app.js
```
 
All `[webapp/js/app.js]` dependencies are now minified with Uglify.js and concatenated into single file. But what happened with individual modules? Module `[webapp/js/app/category/specialItem.js]` has a dependency `[webapp/js/app/category/Item.js]`. Open your optimized `[webapp/js/app/category/specialItem.js]` and pass it to beautifier. As you can see from the output `[js/app/category/item.js]` is not bundled with `[js/app/category/specialItem.js]` file.
 
 Optimized module:
 
```js
define(["./item"], function (Item) {
    var SpecialItem = function (itemName) {
            this.color = "Default color", this.weigth = "Default weigth", Item.call(this, itemName)
        };
    return SpecialItem.prototype = new Item, SpecialItem.prototype.constructor = SpecialItem, SpecialItem
})
```
 
**Example 2: Optimize additional modules**

In this example we will add `[webapp/js/app/category/specialItem.js]` to modules array of `build.js` configuration object.
 
Paste following code to your `build.js` file.
 
```js
{
    baseUrl: "js/lib",
    appDir: "..",
    dir: "dist",

    modules: [
        { name: "app" },
        { name: "app/category/specialItem" }
    ],

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
 Console output of `build.js`
 
```js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/build/r.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/category.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/item.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app/category/specialItem.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/app.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/jquery.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/require.js
Uglifying file: C:/vlada/practice/require/webapp/build/dist/js/lib/text.js

js/app.js
----------------
js/lib/jquery.js
js/app/category/category.js
js/app/category/item.js
js/app/category/specialItem.js
js/app.js

js/app/category/specialItem.js
----------------
js/app/category/item.js
js/app/category/specialItem.js
```
 
Now `[js/app/category/item.js]` is bundled with `[js/app/category/specialItem.js]` module. 
 
 Optimized module: 
 
```js
define("app/category/item", [], function () {
    var Item = function (itemName) {
            this.name = itemName
        };
    return Item.prototype.getItemName = function () {
        return this.name
    }, Item
}), define("app/category/specialItem", ["./item"], function (Item) {
    var SpecialItem = function (itemName) {
            this.color = "Default color", this.weigth = "Default weigth", Item.call(this, itemName)
        };
    return SpecialItem.prototype = new Item, SpecialItem.prototype.constructor = SpecialItem, SpecialItem
})
```
 
### <a name="t2" id="t0">Require.js optimizer compile dependencies to single file</a>

Paste the following code to your `build.single.js` file.
 
```js
{
    baseUrl: "../js/lib",
    name: "../app",
    out: "app-built.js",

    paths: {
        app: '../app',
        jquery: 'jquery',
    }
}
```

**name** - Location of module to be exported as a single file with all dependencies.<br/>
**out** - File `app-built.js` is created.
 
 Console output of `build.single.js`:
 
```js
 Tracing dependencies for: ../app
 Uglifying file: C:/vlada/practice/require/webapp/build/app-built.js
 
 C:/vlada/practice/require/webapp/build/app-built.js
 ----------------
 C:/vlada/practice/require/webapp/js/lib/jquery.js
 C:/vlada/practice/require/webapp/js/app/category/category.js
 C:/vlada/practice/require/webapp/js/app/category/item.js
 C:/vlada/practice/require/webapp/js/app/category/specialItem.js
 C:/vlada/practice/require/webapp/js/lib/../app.js
```
 
All of our modules are now glued together into single `[webapp/build/app-built.js]` file.
 
### <a name="source" id="source">Source code listing</a>
 
`build.js`
```js
{
    baseUrl: "js/lib",
    appDir: "..",
    dir: "dist",

    modules: [
        { name: "app" },
        { name: "app/category/specialItem" }
    ],

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
`build.single.js`
```js
{
    baseUrl: "../js/lib",
    name: "../app",
    out: "app-built.js",

    paths: {
        app: '../app',
        jquery: 'jquery'
    }
}
```
 
In the RequireJS tutorial part 3 I will talk about RequireJS integration with Backbone, Handlebars and jQuery with Require.js. Stay tuned :)
 
**References**

1. http://requirejs.org/docs/optimization.html
2. https://github.com/jrburke/r.js/blob/master/build/example.build.js
